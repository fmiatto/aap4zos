# SA z/OS policies with Ansible

[playbooks/add_to_sa.yml](playbooks/add_to_sa.yml) defines classes, application groups (APGs)
and applications (APLs) in the SA z/OS policy database (PDB), builds the configuration and
activates it on the running system. The entries are described in YAML files under
`vars/policies/`; the playbook turns them into one SA text file ("Making Updates with Text
Files"), applies it with SA's batch text-file update, runs a full build and loads the new
configuration.

The normal run works straight on the live PDB and the live configuration, with no backups or
approvals. To try something first, the same run can work on a copy of the PDB instead
([Test PDB](#test-pdb)).

| Path | What it is |
|---|---|
| `playbooks/add_to_sa.yml` | The playbook: stages apply, build, activate |
| `vars/sa_env.yml` | The target system: PDB, configuration GDG, libraries, work data sets, jobs |
| `vars/policies/*.yml` | One policy file per set of entries |
| `templates/apl.txt.j2` | Writes the SA text file from a policy file |
| `templates/textupd.jcl.j2`, `ingebbld.jcl.j2`, `newgen.jcl.j2`, `jobcard.j2` | Batch jobs: text-file update, build, new configuration generation |
| `roles/sa_policy_common/tasks/` | `render.yml` / `render_one.yml` (check and render the policies), `run_job.yml` (submit a job and check its output) |
| `SA_Helpful_Commands.md` | SA display commands and where to find SA's own data |

## Requirements

- SA z/OS is running and loads its configuration from a GDG (`sa_config_gdg_base`).
- The classes the APLs link to exist in the PDB (for DB2: `C_DB2_NORM`, `C_DB2_DEPENDENTS`), or
  are defined in a policy file under `classes:`.
- ZOAU and `ibm.ibm_zos_core` on the target, and an inventory that defines `zos_host` and
  `environment_vars` (see `inventory/`; `inventory/local/` is git-ignored, for a local inventory
  with credentials).
- The Ansible user can update the PDB, the configuration GDG and the work data sets
  (`sa_work_hlq`).

## 1. The system: `vars/sa_env.yml`

| Variable | Value | Meaning |
|---|---|---|
| `sa_policies` | `db2-dba1`, `cics62`, `rmf`, `mq1` | Policy files to process, in order (names in `vars/policies/` without `.yml`) |
| `sa_proc` | `AUTOPROC` | SA agent STC (`F AUTOPROC,...`) |
| `sa_pdb_name` / `sa_pdb_dsn` | `LTEPLEX_PDB` / `SYSAUTO.PDB` | Policy database |
| `sa_config_gdg_base` | `SYSAUTO.PDB.SOCNTL` | Configuration GDG SA loads |
| `sa_config_volume`, `sa_config_unit`, `sa_config_space` | `LTECAT`, `3390`, `CYL,(40,20,10)` | Where a new generation goes |
| `sa_dialog_hlq` / `sa_ispf_hlq` | `SYS1.ING` / `SYS1.ISP` | SA dialog and ISPF libraries |
| `sa_work_hlq` | `SYSAUTO.PROTO` | Work data sets: JCL (`.SAPDB.WORK`), text file (`.SAPDB.WORK.UPD`), update report (`.SAPDB.WORK.RPT`), build output (`.SAPDB.SOCNTL`) |
| `sa_text_lrecl` | `255` | Longest text-file line. 255 is SA's own maximum; the text file is VB, so lines are not padded |
| `sa_job` | class `A`, msgclass `X` | Job card |

`sa_textupd` and `sa_build` hold the DDs and commands of IBM's samples `SINGSAMP(INGEBFLT)`
(text-file update) and `SINGSAMP(INGEBBLD)` (build).

Variables given only with `-e`:

| Variable | Meaning |
|---|---|
| `sa_policies=a,b` | Process these policy files instead of the list above |
| `sa_update_only=true` | Make every policy file update-only |
| `sa_test_pdb=<dsn>` | Work on a copy of the PDB ([Test PDB](#test-pdb)) |
| `sa_test_pdb_refresh=true` | Copy the live PDB into the test PDB again before apply |
| `sa_test_build_dsn=<dsn>` | Build output for the test PDB (default `<test pdb>.SOCNTL`) |

## 2. The policies: `vars/policies/*.yml`

| File | Entries |
|---|---|
| [db2-dba1.yml](vars/policies/db2-dba1.yml) | APG `DBA1_DB2`: DB2 subsystem DBA1 (MSTR, IRLM, DBM1, DIST) on the DB2 classes |
| [cics62.yml](vars/policies/cics62.yml) | APG `CICS62_CICS`: CICS region CICS62 on class `C_CICS`, with its own shutdown |
| [rmf.yml](vars/policies/rmf.yml) | Update-only: Restart after IPL `START` on the existing APL `RMF` |
| [mq1.yml](vars/policies/mq1.yml) | APG `MQ1_MQ`: IBM MQ queue manager, channel initiator and web console, with their own start and stop commands and messages |
| [syntax-test.yml](vars/policies/syntax-test.yml) | Test PDB only: two classes and an APL using every section the template writes |
| [syntax-test-thresholds.yml](vars/policies/syntax-test-thresholds.yml) | Test PDB only, update-only: APL thresholds on the syntax-test APL |

All files of a run go into one text file, applied in one job, then one build and one
activation. SA checks the whole text file before it writes anything: if any part has an
error, nothing is written. A file whose entries refer to another file's entries (a class, an
APG in a relationship) comes after that file in `sa_policies`.

### File layout

```yaml
sa_update_only: false                 # true: update entries that already exist (UPD only)
sa_test_only: false                   # true: run only with -e sa_test_pdb

sa_new_policy:
  classes: [...]                      # optional: new classes
  apg: {...}                          # optional: one application group
  apls: [...]                         # applications
  update_sections: {...}              # which UPD parts to write
```

The text file gets, per policy file:

1. `NEW` sections, unless the file is update-only: the classes, the APG (linked to its
   `systems`), then the APLs (each linked to its class and joined to its APGs). They create
   the entries with the fields marked NEW below.
2. One `UPD` section per class, APG and APL, made of the parts switched on in
   `update_sections`. A part is written only for entries that have data for it.

A full add fails with `Name ... already exists` on an entry that is already in the PDB; SA
then writes nothing. Update-only files need only each entry's `name` and the fields to change.

### APG (`apg:`)

| Field | | Values |
|---|---|---|
| `name` | NEW | max 20: `A-Z 0-9 @ # $ _` |
| `type` | NEW | `SYSTEM` \| `SYSPLEX` |
| `nature` | NEW | `BASIC` \| `MOVE` \| `SERVER` |
| `description` | NEW | max 40 |
| `systems` | NEW | systems the APG is linked to; may be empty (then nothing in the APG is automated) |
| `long_description` | `apg` | up to 5 lines, max 50 each |
| `automation_name` | `apg` | default: the APG name |
| `behavior` | `apg` | `ACTIVE` (default) \| `PASSIVE` |
| `automatically_link` | `apg` | `YES` (default) \| `NO` |
| `relationships` | `apg` | as for an APL |

### APL (`apls:`)

| Field | | Values |
|---|---|---|
| `name` | NEW | entry name, max 20: `A-Z 0-9 @ # $ _` |
| `subsystem` | NEW | max 11: `A-Z 0-9 @ # $` |
| `job_name` | NEW | max 8 |
| `job_type` | NEW | `MVS` \| `NONMVS` \| `TRANSIENT` |
| `description` | NEW | max 40 (in update-only files written by `description`) |
| `category`, `subcategory` | NEW | optional, e.g. `DB2` / `MSTR`, `MQ` / `CHIN` |
| `link_to_class` | NEW | class whose policy the APL inherits |
| `apgs` | NEW | APGs the APL joins: default the file's APG; `[]` for none; or several |
| `long_description` | `description` | up to 5 lines, max 50 each |
| `command_prefix` | `application_info` | e.g. `"-DBA1"` |
| `restart_after_ipl` | `application_info` | e.g. `START` (start after every IPL; without it the class decides) |
| `external_startup` | `application_info` | `INITIAL` \| `ALWAYS` \| `NEVER` |
| `external_shutdown` | `application_info` | `FINAL` \| `ALWAYS` \| `NEVER` |
| `skip_active_status` | `application_info` | `YES` \| `NO` |
| `relationships` | `relationships` | see below |
| `messages` | `messages` | see below |
| `startup` | `startup` | see below |
| `shutdown` | `shutdown` | see below |
| `thresholds` | `thresholds` | see below |
| `minor_resources` | `minor_resources` | see below |
| `db2_control` | `db2_control` | `subsystem_id` (1-4), `active_log_dataset` (data set name) |

An APL in no APG, or in an APG linked to no system, is defined in the PDB but not automated.

### Class (`classes:`)

A class is an APL entry with object type CLASS. APLs (and other classes) link to it with
`link_to_class` and inherit its policy. A class takes the APL fields except `job_name`,
`job_type`, `apgs` and `db2_control`:

| Field | | Values |
|---|---|---|
| `name` | NEW | max 20: `A-Z 0-9 @ # $ _` |
| `subsystem` | NEW | max 11, `_` allowed; default the class name. The build refers to the class by this name |
| `description` | NEW | max 40 |
| `category`, `subcategory`, `link_to_class` | NEW | optional; `link_to_class` is the parent class |
| all `UPD` fields of an APL | | as for an APL |

```yaml
  classes:
    - name: C_SAT_CHILD
      subsystem: C_SAT_CHD
      description: Syntax test - child class
      link_to_class: C_SAT_BASE
      skip_active_status: "YES"
```

### `update_sections`

| Switch | Writes | Replacement |
|---|---|---|
| `apg` | the APG's `long_description`, `automation_name`, `behavior`, `automatically_link`, `relationships` | field / policy |
| `description` | `long_description`; in update-only files also `description` | field: only the given lines change |
| `application_info` | `command_prefix`, `restart_after_ipl`, `external_startup`, `external_shutdown`, `skip_active_status` | field: only the given fields change |
| `relationships` | `relationships` | policy: replaces all relationships of the entry |
| `messages` | `messages` | block: replaces all messages of the entry |
| `startup` | `startup` | block: replaces all start commands |
| `shutdown` | `shutdown` | block: replaces all stop commands |
| `thresholds` | `thresholds` | field |
| `minor_resources` | `minor_resources` | block: replaces all minor resources |
| `db2_control` | `db2_control` (APLs) | policy |

A switch left out is off. Because policy and block replacement replace everything, list every
relationship, message or command the entry should keep (or leave the switch off).

### Relationships

```yaml
      relationships:
        - { type: HASPARENT, resource: BASE_SYS/APG/= }
        - type: HASPARENT
          resource: ZQSXMSTR/APL/=          # <entry>/APL/<system> or <group>/APG/<system>; = is this system
          condition: StartsMeAndStopsMe     # optional
          description: Needs the queue manager   # optional; also sequence, automation, chaining
```

A `StartsMe` / `StopsMe` condition means the parent starts / stops the APL, so the APL needs
`external_startup` (`INITIAL` | `ALWAYS`) / `external_shutdown` (`FINAL` | `ALWAYS`), from its
class or set in the file. For an APL without a class the checks require them; otherwise the
build warns "has nothing or NEVER specified as external startup value". An APL that SA starts
with its own commands after its parent uses a plain `HASPARENT`.

### Start and stop commands

```yaml
      startup:
        - phase: STARTUP                    # INIT | STARTUP | POSTSTART
          mixed_case: "NO"                  # optional
          commands:
            - { command: "MVS &SUBSCMDPFX START QMGR" }
            - { type: COLD, command: "MVS S &SUBSJOB,PARM=COLD" }   # type: start type, empty = all
      shutdown:
        - phase: NORM                       # NORM | IMMED | FORCE
          mixed_case: "NO"                  # optional
          passes:
            - { pass: 1, command: "MVS F &SUBSJOB,CEMT P SHUT" }
            - { pass: 4, command: "MVS C &SUBSJOB" }
```

Both also take `function` (Automated Function/`'*'`) per command.

### Messages

Each message has an `id`, an optional `description` and any of these actions:

```yaml
      messages:
        - id: CSQY022I
          description: Queue manager is available
          at: { status: UP }                # AT: status, monitor_status, capture, override
        - id: DFS3907A
          reply: [{ text: S }]              # REPLY: text, selection, retry
        - id: DSNP007I
          cmd: [{ selection: RECOVER, command: "MSG LOG RECOVERY NEEDED" }]   # CMD: command, selection, function
          code: [{ code1: "*DSNDB01*", code2: "*", code3: BATCH, value: RECOVER }]
        - id: CICSINFO
          user: [{ keyword: CICSCMD, data: "('ACTIVE TASKS','CEMT I TA')" }]
        - id: SATM002I
          at:
            override:                       # AT override: one list entry per line
              - "IF MSGID = 'SATM002I' THEN BEGIN;"
              - "  ALWAYS"
              - "  EXEC(CMD('INGMON SATAPL1,STATUS=NORMAL')ROUTE(ONE %AOFOPGSSOPER%));"
              - "END;"
        - id: CAPMSGS                       # message capturing: capture: instead of code:
          capture:
            - { msgid: DFS2011I, jobname: "*", severity: CRITICAL, alert: "YES" }
              # also: user, inform_list, comment_type, comment_text, dom_list
```

The `messages` part replaces all messages of the entry. The build puts the AT entries into
`ACFMZ999` and the message IDs into `MPFLSTSA`; they take effect only if NetView loads that
automation table and that MPF list (LTE1 runs `DSITBL01 INGMSGSA INGMSG01` and MPF
`.NO_ENTRY AUTO(YES)`).

### Thresholds and minor resources

```yaml
      thresholds:                           # the entry's own error thresholds
        critical_number: 3                  # errors (0-999) ...
        critical_interval: "00:10:00"       # ... within hh:mm:ss
        frequent_number: 3
        frequent_interval: "01:00:00"
        infrequent_number: 3
        infrequent_interval: "12:00:00"
      minor_resources:
        - name: TRAN
          thresholds: { critical_number: 2, critical_interval: "00:05:00" }   # same keys
```

### Limits

A text-file line holds at most 255 characters (`sa_text_lrecl`), so each value must fit after
its field name; SA also limits some fields itself:

| Values | Max | Limit set by |
|---|---|---|
| `description` | 40 | SA |
| `long_description` lines | 50 | kept below SA's limit |
| message `reply` text | 61 | SA |
| AT `override` lines | 72 | SA |
| relationship values | 216 | line length |
| `db2_control.active_log_dataset` | 220 (44 for a data set name) | line length |
| message, startup, shutdown values (commands, texts, codes) | 212 | line length |

The line-length widths come from the columns in `templates/apl.txt.j2` (see its header). SA
checks its own limits again in the update report and writes nothing when one is exceeded.

### Checks before anything is submitted

`render_one.yml` stops the run, naming the policy and the entry, when:

- a name, subsystem, job name or enumerated value is not valid (tables above);
- an entry name appears twice in a file;
- an APL without a class has a `StartsMe` / `StopsMe` relationship but no matching
  `external_startup` / `external_shutdown`;
- a message has `capture:` but is not `CAPMSGS`, or `CAPMSGS` has `code:`;
- an AT key other than `status`, `monitor_status`, `capture`, `override` is used;
- a threshold is not a number of 0-999 or an `hh:mm:ss` interval;
- a value is longer than its limit (table above), or a text-file line is longer than
  `sa_text_lrecl`;
- a `sa_test_only` file runs without `sa_test_pdb`.

YAML reads unquoted `YES` / `NO` as booleans; the template writes them back as `YES` / `NO`,
so both forms work.

## 3. Run it

All policies in `sa_policies`:

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml
```

Only some of them (comma-separated, in order):

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml -e sa_policies=db2-dba1,rmf
```

| Stage | What it does |
|---|---|
| apply | Renders the policies with `templates/apl.txt.j2` into one text file (`<work>.SAPDB.WORK.UPD`, VB, lines up to 255), runs the text-file update against the PDB and prints SA's update report |
| build | Full build of the PDB into an empty `<work>.SAPDB.SOCNTL` |
| activate | IEBCOPY of the build into `<GDG>(+1)` (allocated like the active generation), `INGAMS REFRESH CFG=<new generation>`, waits until `ACF STATUS` shows it, `INGLIST` of the new APGs and APLs |

Run one or more stages with `--tags apply`, `--tags build`, `--tags activate` (or
`--tags apply,build`). Rendering and the checks run in every stage, and with any other tag
(for example `--tags render_only`) they run alone, without touching the system.

`-e sa_update_only=true` makes every listed policy update-only, to change entries that already
exist.

A job fails the run on a completion code above its `max_rc` or on an `ERROR` / `WARNING` line
in its output (`sa_fail_message_regex`). Its ISPLOG and SYSTSPRT are printed when it fails;
add `-v` to always print them. `INFO` lines in the update report are not errors (for example,
SA clears Automation and Chaining on HASPARENT relationships).

### Test PDB

`-e sa_test_pdb=<dsn>` runs apply and build on a copy of the PDB; without it, every stage
works on `sa_pdb_dsn`:

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml -e sa_test_pdb=SYSAUTO.PROTO.TESTPDB -e sa_policies=syntax-test --tags apply,build
```

- apply copies the live PDB into `<dsn>` when `<dsn>` does not exist, or every time with
  `-e sa_test_pdb_refresh=true`, then updates `<dsn>`. A refresh is needed to run a full add
  again on the same test PDB.
- build writes `<dsn>.SOCNTL` (or `-e sa_test_build_dsn=<dsn>`).
- activate refuses to run: a test build never goes into the configuration GDG.
- The copy keeps the live PDB's log settings (member `AOFTIST`), so SA records the test
  changes in the live PDB's change log (`SYSAUTO.PDB.LOG1` / `LOG2`). The live policy itself
  is not touched.

`syntax-test.yml`, then `syntax-test-thresholds.yml`, exercise every section the template
writes; after their build, the members of `<dsn>.SOCNTL` named in the build report
(`$BLDRPT`) show what SA made of each entry.

## Notes

- The text-file layout of an existing entry can be seen by exporting it in the dialog:
  `5 Data Management` → `3 Update via File` → `1 Write selected data from Policy Database to file`.
  Quote the output data set name (`'SYSAUTO.PROTO.EXPORT.CICS0001'`), or ISPF adds your TSO
  prefix. The batch export `SINGSAMP(INGEBFXT)` stops on this PDB with an abend in AOFFZIPS:
  some entries (the TCPIP ones) hold PIPE commands longer than a 255-character line.
- Start and stop through SA once active: `INGREQ DBA1MSTR/APL/LTE1 REQ=START` (or `STOP`); see
  `zos_admin/sysauto_*.yml`.

## Not supported

- Values longer than a 255-character text-file line.
- Policy items: AUTOMATION FLAGS, APPLICATION SYMBOLS, CICS / IMS CONTROL, and minor-resource
  policies other than thresholds.
- Deleting entries.

## Other playbooks

| Playbook | What it does |
|---|---|
| [grant_db2_to_sa.yml](playbooks/grant_db2_to_sa.yml) | Grants DB2 `SYSOPR` to SA's user ID, so SA can stop DB2 (`INGDB2 CHECK_INDOUBT`) |
| [fix_zoau_apf.yml](playbooks/fix_zoau_apf.yml) | APF-authorizes ZOAU's helper programs, so the `ibm_zos_core` modules work |
