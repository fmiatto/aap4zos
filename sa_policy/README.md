# Add DB2, CICS (or any APG/APLs) to SA z/OS

One playbook, [playbooks/add_to_sa.yml](playbooks/add_to_sa.yml), adds application groups (APGs)
and their applications (APLs) to the SA z/OS policy database, builds the configuration and
activates it on the running system, for one policy file or several in one run. Written for
freshly built systems: it goes straight to the PDB and the live configuration, with no
backups, checks or approvals.

The examples in `vars/policies/` (LTE1, SA 4.3): DB2 subsystem DBA1 after the base system,
CICS region CICS62 after DB2, and RMF restarted after every IPL.

## Requirements

- SA z/OS is running and loads its configuration from a GDG (`sa_config_gdg_base`).
- The PDB already holds the classes the APLs link to (for DB2: `C_DB2_NORM`, `C_DB2_DEPENDENTS`).
- ZOAU and `ibm.ibm_zos_core` on the target, and an inventory with `environment_vars`
  (see `inventory/`, and `inventory/local/` for a local test inventory).

## 1. Describe the system: `vars/sa_env.yml`

| Variable | Example | Meaning |
|---|---|---|
| `sa_proc` | `AUTOPROC` | SA agent STC (`F AUTOPROC,...`) |
| `sa_pdb_name` / `sa_pdb_dsn` | `LTEPLEX_PDB` / `SYSAUTO.PDB` | Policy database |
| `sa_config_gdg_base` | `SYSAUTO.PDB.SOCNTL` | Configuration GDG SA loads |
| `sa_config_volume`, `sa_config_unit`, `sa_config_space` | `LTECAT`, `3390`, `CYL,(40,20,10)` | Where the new generation goes |
| `sa_dialog_hlq` / `sa_ispf_hlq` | `SYS1.ING` / `SYS1.ISP` | SA dialog and ISPF libraries |
| `sa_work_hlq` | `SYSAUTO.PROTO` | Work data sets (text file, JCL, report, build output) |
| `sa_job` | class `A`, msgclass `X` | Job card |

`sa_textupd` and `sa_build` hold the DDs and commands of IBM's samples `SINGSAMP(INGEBFLT)`
(text-file update) and `SINGSAMP(INGEBBLD)` (build). They work as they are.

## 2. Describe the policies: `vars/policies/*.yml`

One file per APG (or per set of updates). `sa_policies` in `vars/sa_env.yml` lists the files
to add, in order, without `.yml`:

```yaml
sa_policies:
  - db2-dba1      # APG DBA1_DB2: DBA1MSTR, DBA1IRLM, DBA1DBM1, DBA1DIST
  - cics62        # APG CICS62_CICS: CICS62, child of DBA1_DB2
  - rmf           # update only: Restart after IPL on the existing APL RMF
```

All listed files go into one text file, applied in one job, then one build and one
activation. A file whose relationships point to another file's APG must come after it
(cics62 after db2-dba1). SA checks the whole text file first: if any part has an error,
nothing is written.

A policy file:

```yaml
sa_new_policy:
  apg:
    name: DBA1_DB2                    # max 20 chars
    type: SYSTEM                      # SYSTEM | SYSPLEX
    nature: BASIC                     # BASIC | MOVE | SERVER
    description: DB2 subsystem DBA1   # max 40 chars
    systems: [LTE1]                   # systems the APG is linked to

  apls:                               # every APL here joins the APG above
    - name: DBA1MSTR                  # entry name
      subsystem: DBA1MSTR
      job_name: DBA1MSTR
      job_type: MVS                   # MVS | NONMVS | TRANSIENT
      category: DB2
      subcategory: MSTR               # optional
      link_to_class: C_DB2_NORM       # inherit start/stop/message policy
      description: DB2 DBA1 - system services (MSTR)
      command_prefix: "-DBA1"         # optional
      restart_after_ipl: START        # optional
      relationships:
        - { type: HASPARENT, resource: BASE_SYS/APG/= }

    - name: DBA1IRLM
      ...
      link_to_class: C_DB2_DEPENDENTS
      relationships:
        - { type: HASPARENT, resource: DBA1MSTR/APL/=, condition: StartsMeAndStopsMe }

  update_sections:
    application_info: true            # command_prefix, restart_after_ipl
    relationships: true
    shutdown: false
```

- Relationships (dependencies): `type` and `resource` are required; `condition`,
  `automation`, `chaining`, `description` and `sequence` are optional. `resource` is
  `<entry>/APL/<system>` or `<group>/APG/<system>` (`=` is the same system). DB2's chain:
  `DBA1MSTR` HASPARENT `BASE_SYS/APG/=`; IRLM, DBM1 and DIST HASPARENT `DBA1MSTR` with
  `StartsMeAndStopsMe` (MSTR starts and stops them); CICS62 HASPARENT `DBA1_DB2/APG/=`.
- Shutdown commands, for an APL whose class has none (as `C_CICS`), with
  `update_sections.shutdown: true`:

  ```yaml
      shutdown:
        - phase: NORM                 # NORM | IMMED | FORCE
          passes:
            - { pass: 1, command: "MVS F &SUBSJOB,CEMT P SHUT" }
            - { pass: 2, command: "MVS F &SUBSJOB,CEMT P SHUT IMMED" }
            - { pass: 4, command: "MVS C &SUBSJOB" }
  ```

- `restart_after_ipl: START` makes SA start the APL after every IPL. Without it the class
  default applies (`C_APPL`: MAYBE), and SA does not restart an application that was stopped
  outside SA before the IPL.
- Update-only file: `sa_update_only: true` at the top writes only the `UPD` sections, for
  entries that already exist; it needs no `apg:` and only the APL `name` plus the fields to
  change (see [vars/policies/rmf.yml](vars/policies/rmf.yml)).

## 3. Run it

All policies in `sa_policies`:

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml
```

Only some of them (comma-separated, in order):

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml -e sa_policies=cics62
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/add_to_sa.yml -e sa_policies=db2-dba1,rmf
```

| Stage | What it does |
|---|---|
| apply | Renders the policies with `templates/apl.txt.j2` into one text file (`<work>.SAPDB.WORK.UPD`, copy in `WORK(POLICY)`), runs the text-file update against the PDB and prints SA's update report |
| build | Full build of the PDB into an empty `<work>.SAPDB.SOCNTL` |
| activate | IEBCOPY of the build into `<GDG>(+1)` (allocated like the active generation), `INGAMS REFRESH CFG=<new generation>`, waits until `ACF STATUS` shows it, `INGLIST` of the new entries |

Run a single stage with `--tags apply`, `--tags build` or `--tags activate`.

`-e sa_update_only=true` makes every listed policy update-only: use it to change entries that
already exist (relationships, shutdown, command prefix, restart after IPL) on a system where
the APGs and APLs were added before.

The text-file layout of any existing entry can be seen by exporting it in the dialog:
`5 Data Management` → `3 Update via File` → `1 Write selected data from Policy Database to file`.
Quote the output data set name (`'SYSAUTO.PROTO.EXPORT.CICS0001'`), or ISPF adds your TSO
prefix.

A job fails the run on a completion code above its `max_rc` or on an `ERROR` / `WARNING` line
in its output. The job's ISPLOG and SYSTSPRT are printed when it fails; add `-v` to always
print them.

## Notes

- `apply` creates entries with `NEW`. Running it again on the same PDB fails with
  `Name ... already exists`; SA then changes nothing.
- `INFO` lines in the update report are not errors (for example, SA clears Automation and
  Chaining on HASPARENT relationships).
- The build also writes DB2 message entries into `ACFMZ999` and `MPFLSTSA`. They take effect
  only if NetView loads that automation table and that MPF list; LTE1 runs
  `DSITBL01 INGMSGSA INGMSG01` and MPF `.NO_ENTRY AUTO(YES)`.
- Start and stop through SA once active: `INGREQ DBA1MSTR/APL/LTE1 REQ=START` (or `STOP`); see
  `zos_admin/sysauto_*.yml`.
