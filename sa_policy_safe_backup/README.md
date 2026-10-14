# SA z/OS policy automation: DB2 DBA1 APLs

Ansible implementation of [SA_POLICY.md](SA_POLICY.md). It adds DB2 subsystem **DBA1** to the
SA z/OS policy database of **LTE1** (`LTEPLEX_PDB` / `SYSAUTO.PDB`, SA 4.3) without anyone
working in the dialog. Every change goes to a **clone** of the PDB and a **staging** build output;
production is only read.

## What gets defined

From `D A,L` and `D O` on LTE1 (see [SA_SYSTEM_DATA.md](SA_SYSTEM_DATA.md)):

| Entry | Type | Class link | Notes |
|---|---|---|---|
| `DBA1_DB2` | APG (SYSTEM, BASIC) | – | Linked to system `LTE1` |
| `DBA1MSTR` | APL, DB2/MSTR | `C_DB2_NORM` | Command prefix `-DBA1` |
| `DBA1IRLM` | APL, DB2/IRLM | `C_DB2_DEPENDENTS` | HASPARENT `DBA1MSTR` |
| `DBA1DBM1` | APL, DB2/DBM1 | `C_DB2_DEPENDENTS` | HASPARENT `DBA1MSTR` |
| `DBA1DIST` | APL, DB2/DIST | `C_DB2_DEPENDENTS` | HASPARENT `DBA1MSTR` |

The definitions live in [vars/new_apl.yml](vars/new_apl.yml). [templates/apl.txt.j2](templates/apl.txt.j2)
renders them into the dialog's "Making Updates with Text Files" format: `NEW` sections with
`LINK_TO_CLASS` and `WHERE_USED`, then `UPD` sections for the command prefix and relationships.
`DBA1ADMT` is not running, so it is left out. Add it to `apls` if you need it.

## Layout

```
sa_policy/
├── inventory/                 # CLI only; AAP uses its own z/OS inventory
├── vars/sa_env.yml            # LTE1 facts, work HLQ, P3/P4 batch commands
├── vars/new_apl.yml           # The DB2 DBA1 APG + APLs
├── templates/apl.txt.j2       # Text-file update input
├── templates/textupd.jcl.j2   # P3 batch job (ISPF batch)
├── templates/ingebbld.jcl.j2  # P4 batch build job
├── playbooks/00..90_*.yml     # One playbook per phase
└── roles/sa_policy_common/    # Naming, ENQ check, manifests, job runner, JCL guard
```

## Run order

In AAP, create one job template per playbook, the same way as `zos_admin/sysauto_*.yml`
(`hosts: zos_host`, `environment_vars` from the inventory). Chain them in a workflow so each
phase is a checkpoint. To run them from a terminal, see [Running manually](#running-manually-outside-aap).

| Playbook | Phase | Writes | Rollback |
|---|---|---|---|
| `00_discover.yml` | P0: baselines, DB2 evidence, already-defined guard, **spike** | nothing | – |
| `10_backup_clone.yml` | P1: Backup A (PDB), Backup B (active config), clone, all verified by cksum, `.MF` manifests | `<HLQ>.*` | `delete_clone` |
| `20_render_apl.yml` | P2: validate vars, render the text file, optional diff vs. a dialog export | nothing | – |
| `30_apply_to_pdb.yml` | P3: text-file update of the **clone** in batch | clone, `<HLQ>.SAPDB.WORK` | `reclone` |
| `40_build_staging.yml` | P4: batch build of the clone into staging (full build when staging is empty) | `<HLQ>.SAPDB.PROTO.SOCNTL` | `delete_staging` |
| `50_validate.yml` | P5: staging vs. active config: new/changed members, every APL present, unexplained changes | `WORK(P5REPORT)` | – |
| `60_activate.yml` | P6: copy the validated production build into a new GDG generation, `INGAMS REFRESH`, verify | `SYSAUTO.PDB.SOCNTL(+1)`, running configuration | `refresh_previous` |
| `90_rollback.yml` | P7: `delete_staging`, `reclone`, `delete_clone`, `restore_production_pdb`, `refresh_previous` | – | – |

Each playbook publishes its result with `set_stats` (for example `.artifacts.sa_backup` or
`.artifacts.sa_validation`), like the `sysauto_*` playbooks do.

## Running manually (outside AAP)

### 1. Set up macOS (one time)

These steps were tested on an Apple Silicon Mac with Homebrew. Other platforms need the same
pieces: ansible-core 2.18, the IBM collections, and `sshpass` if you log in with a password.

**1.1 Python 3.13 and ansible-core 2.18 in a virtualenv.** Don't use Homebrew's `ansible`
package. It brings ansible-core 2.21, and the collection's action plugins fail on it with
`not enough values to unpack (expected 4, got 2)`. ansible-core 2.18 needs Python 3.13 or
earlier, and Homebrew's default Python is 3.14:

```bash
brew install python@3.13
```

```bash
python3.13 -m venv ~/.venvs/ansible-zos
```

```bash
source ~/.venvs/ansible-zos/bin/activate && pip install "ansible-core>=2.18,<2.19"
```

**1.2 Activate the venv in every new terminal.** If Homebrew's `ansible` is also installed,
clear the shell's command cache and confirm the version shows `core 2.18.x`:

```bash
source ~/.venvs/ansible-zos/bin/activate && hash -r && ansible --version
```

**1.3 IBM collections** from the repository's `requirements.yml` (`ibm.ibm_zos_core` 2.0.0, as in AAP):

```bash
ansible-galaxy collection install -r requirements.yml
```

**1.4 `sshpass`**, only needed for password login. It isn't in Homebrew core:

```bash
brew install hudochenkov/sshpass/sshpass
```

**1.5 Run everything from the repository root**, so Ansible uses the root `ansible.cfg`. That file
sets `pipelining=True`, which z/OS needs together with `PYTHONSTDINENCODING`. It also sets
`callback_result_format=yaml`, which gives YAML output from the built-in callback. The old
`stdout_callback=yaml` setting fails in the venv with `Invalid callback for stdout specified: yaml`.

### 2. z/OS target

The user ID needs SSH access to z/OS UNIX on LTE1, with IBM Open Enterprise SDK for Python
(3.12 or later) and Z Open Automation Utilities (1.4 or later) installed. It also needs authority to:

- read `SYSAUTO.PDB` and `SYSAUTO.PDB.SOCNTL.*`;
- create data sets under `sa_work_hlq`;
- issue the `F AUTOPROC,...`, `D ...` and `-DBA1 DISPLAY` console commands.

To use an SSH key instead of a password, set it up once. Then you need neither `sshpass`
nor `ansible_password`:

```bash
ssh-copy-id <user>@<lte1-address>
```

#### 2.1 Prepare ZOAU for console commands and batch jobs (one time, per ZOAU install)

Every phase reads the active configuration with `F AUTOPROC,ACF STATUS` through `zos_operator`,
which runs ZOAU's `opercmd`. P3 and P4 also submit batch jobs through `zos_job_submit` and
wait for them with `zos_job_output`, which run ZOAU's `jsub`, `jls`, `pjdd` and related commands.
The helper programs behind these commands have to be APF-authorized in z/OS UNIX. If the
ZOAU files lost their extended attributes (for example when the install was unpacked or copied
without them), P0 fails with:

```
BGYSC0804E Unable to run authorized program opercmdhelper
command: opercmd -T 100 -w -- "F AUTOPROC,ACF STATUS "
```

and the first job submission (P3 `30_apply_to_pdb.yml`) fails with:

```
BGYSC4008E Unable to run unauthorized program jsubhelper.
command: jsub -- "SYSAUTO.PROTO.SAPDB.WORK(TEXTUPD)"
```

The same console command still works from TSO, because TSO `CONSOLE` does not use ZOAU.

**Check the attributes.** Run this in z/OS UNIX, or from the Mac with
`ansible ... -m raw -a '<command>'`:

```bash
ls -E /usr/lpp/IBM/zoau/v1r4/bin | grep -E "opercmd|mvscmdauth|jsub|jcan|jls|pjdd|pcon|vfh"
```

The first attribute column must be `a` (APF-authorized), for example `a-s-`. A broken install
shows `--s-`.

**Grant RACF access to set the APF attribute.** Being UID 0 is not enough: `extattr +a` also needs
READ on the FACILITY profile `BPX.FILEATTR.APF`. Without it, `extattr` fails with
`errno=8B, rsn=0924041A`. Run these from TSO as a RACF SPECIAL user, granting the access to the
user ID that will run `extattr` (the RACF identity, which `su` does not change):

```
RLIST FACILITY BPX.FILEATTR.APF AUTHUSER
RDEFINE FACILITY BPX.FILEATTR.APF UACC(NONE)
PERMIT BPX.FILEATTR.APF CLASS(FACILITY) ID(<user>) ACCESS(READ)
SETROPTS RACLIST(FACILITY) REFRESH
```

Run `RDEFINE` only if `RLIST` reports the profile as not found. `<user>` keeps the access
afterwards, which is fine on LTE1.

**Set the APF attribute as superuser.** Use `su` (UID 0) in a new z/OS UNIX session so the new
RACF access applies. Running as the plain user ID fails with `errno=8B, rsn=EF076043`, because
it is not the superuser.

```bash
su
cd /usr/lpp/IBM/zoau/v1r4/bin
extattr +a opercmd opercmdhelper mvscmdauth mvscmdauthhelper
extattr +a jsubhelper jcanhelper jlshelper pjddhelper pconhelper vfhelper
ls -E /usr/lpp/IBM/zoau/v1r4/bin | grep -E "opercmd|mvscmdauth|jsub|jcan|jls|pjdd|pcon|vfh"
exit
```

All ten files should now show `a-s-`; the command wrappers `jsub`, `jcan`, `jls`, `pjdd` and
`pcon` stay `--s-` (only their helpers need APF). What each group is for:

| Files | Used by |
|---|---|
| `opercmd`, `opercmdhelper` | `zos_operator` (console commands, every phase) |
| `mvscmdauth`, `mvscmdauthhelper` | `zos_mvs_raw` when it runs an authorized program |
| `jsubhelper` | `zos_job_submit` |
| `jlshelper`, `pjddhelper` | `zos_job_output` / `zos_job_query` (waiting for a job, reading its DDs) |
| `jcanhelper` | cancelling or purging a job |
| `pconhelper` | `pcon` (reading the system log) |
| `vfhelper` | ZOAU `vf` (part of the same authorized set in the v1.4.2.1 build) |

Reinstalling or upgrading ZOAU resets these attributes, so repeat this step afterwards.

### 3. Local inventory

Keep your personal settings in `sa_policy/inventory/local/`. Git ignores this directory
(see `.gitignore`), so it can hold a plain-text password. It needs two files.

`sa_policy/inventory/local/inventory.yml`:

```yaml
source_system:
  hosts:
    zos_host:
```

`sa_policy/inventory/local/host_vars/zos_host.yml` (the paths are LTE1's):

```yaml
ansible_host: <lte1-address>
ansible_user: <user>
ansible_password: "<password>"          # omit when using an SSH key
ansible_port: 22
ansible_shell_executable: /bin/sh
ansible_scp_extra_args: "-O"             # needed with OpenSSH 9+ (macOS), see below
ansible_python_interpreter: /usr/lpp/IBM/cyp/v3r13/pyz/bin/python3
PYZ: /usr/lpp/IBM/cyp/v3r13/pyz
ZOAU: /usr/lpp/IBM/zoau/v1r4
ZOAU_PYTHON_LIBRARY_PATH: /usr/lpp/IBM/zoau/v1r4/lib/3.13
environment_vars:
  _BPXK_AUTOCVT: ALL
  ZOAU_HOME: /usr/lpp/IBM/zoau/v1r4
  PYTHONPATH: /usr/lpp/IBM/zoau/v1r4/lib/3.13
  LIBPATH: /usr/lpp/IBM/zoau/v1r4/lib:/usr/lpp/IBM/cyp/v3r13/pyz/lib:/lib:/usr/lib:.
  PATH: /usr/lpp/IBM/zoau/v1r4/bin:/usr/lpp/IBM/cyp/v3r13/pyz/bin:/bin:/var/bin
  _CEE_RUNOPTS: FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)
  _TAG_REDIR_ERR: txt
  _TAG_REDIR_IN: txt
  _TAG_REDIR_OUT: txt
  LANG: C
  PYTHONSTDINENCODING: cp1047
```

Protect the password file:

```bash
chmod 600 sa_policy/inventory/local/host_vars/zos_host.yml
```

Why `ansible_scp_extra_args: "-O"`: `zos_ping` copies a REXX module with `scp`, expecting it to
be converted to EBCDIC. From OpenSSH 9, `scp` uses the SFTP protocol and copies in binary, so the
file stays ASCII and z/OS runs it as a shell script: `FSUM7332 syntax error: got (, expecting
Newline`. `-O` brings back the legacy protocol, which does the conversion.

Keep `local/` in its own directory. A `host_vars/` file beside an inventory overrides the
inventory's own values, so with the shared samples next to it, their placeholder `PYZ`/`ZOAU`
would win. The tracked samples in `sa_policy/inventory/` (`inventory.yml`,
`host_vars/zos_host.yml`, `group_vars/all.yml`) are templates for other users.

### 4. Check the connection

```bash
ansible -i sa_policy/inventory/local/inventory.yml zos_host -m ibm.ibm_zos_core.zos_ping
```

Expected:

```
zos_host | SUCCESS =>
    changed: false
    ping: pong
```

Use `-i sa_policy/inventory/local/inventory.yml` in every command below.

### Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `not enough values to unpack (expected 4, got 2)` | ansible-core 2.21 (Homebrew `ansible`) | Use the 2.18 venv (step 1.1). Check with `ansible --version` after `hash -r`. |
| `FSUM7332 syntax error: got (, expecting Newline` | OpenSSH 9+ `scp` sends the REXX module in binary | `ansible_scp_extra_args: "-O"` (step 3) |
| `Invalid callback for stdout specified: yaml` | `stdout_callback=yaml` without the old `community.general` callback | Root `ansible.cfg` uses `callback_result_format=yaml` (step 1.5) |
| `to use the 'ssh' connection type with passwords ... you must install the sshpass program` | Password login without `sshpass` | Step 1.4, or use an SSH key |
| `BGYSC0804E Unable to run authorized program opercmdhelper` | ZOAU helper programs are not APF-authorized | Step 2.1 |
| `extattr`: `chattr() error: rv=-1, errno=8B, rsn=0924041A` | No READ on `BPX.FILEATTR.APF` | Step 2.1, RACF `PERMIT` |
| `extattr`: `chattr() error: rv=-1, errno=8B, rsn=EF076043` | `extattr` run without UID 0 | Step 2.1, run it after `su` |
| P3: `ERROR (Line n): Name … already exists as entry of type …` | P3 already ran on this clone | Step 7: reclone, then run P3 and P4 again |
| P3: `Only partitioned data sets can be allocated shrw for LMINIT` | PDBRPT is not a PDS | `sa_textupd.report_dsn` as a PDS (`30_apply_to_pdb.yml` creates it) |
| P3 ends CC 12 or CC 20 with no report | PDBUPD names a PDS member, or PDBRPT is `SYSOUT` | Sequential `sa_textupd.source_dsn` and a PDS for the report, as in `batch_draft.yml` |
| P4 ends CC 0 but the playbook fails on `WARNING` | Policy warnings in the build | Step 7: fix the policy and run again |

### 5. Run the phases

Run one phase at a time and read the checkpoint at the end of each.
`ANSIBLE_SHOW_CUSTOM_STATS=true` prints the `set_stats` results (the AAP artifacts) at the end of the run.

```bash
export ANSIBLE_SHOW_CUSTOM_STATS=true
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/00_discover.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/10_backup_clone.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/20_render_apl.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/30_apply_to_pdb.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/40_build_staging.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/50_validate.yml
```

Once P3 and P4 are confirmed, `site_prototype.yml` runs P0 to P5 in one go. It stops at the first failing phase:

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/site_prototype.yml
```

### 6. Pass survey values with `-e`

AAP survey variables become extra vars on the command line:

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/10_backup_clone.yml -e sa_replace_clone=true
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/20_render_apl.yml -e "sa_reference_export_dsn='SYSAUTO.PROTO.EXPORT(FTPSERVE)'"
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/40_build_staging.yml -e sa_build_force_all=true
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/90_rollback.yml -e rollback_action=reclone
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/90_rollback.yml -e rollback_action=restore_production_pdb -e "sa_confirm_production_restore='RESTORE SYSAUTO.PDB'"
```

The batch commands for P3 and P4 are kept out of the shared `vars/sa_env.yml`, in a file of
their own passed with `-e @file`. An extra var replaces the whole dictionary, so the file must hold
every key of `sa_dialog_libs`, `sa_textupd` and `sa_build`, not only the changed ones.

This is the file that runs P3 and P4 on LTE1 (SA 4.3), taken from IBM's samples
`SYS1.ING.SINGSAMP(INGEBFLT)` and `(INGEBBLD)`. Keep your copy in
`sa_policy/output/discover_<stamp>/batch_draft.yml` (git-ignored) or anywhere else. The
comments say why each DD differs from the sample:

```yaml
# batch.yml - P3 text-file update and P4 build, both against the CLONE

# SA 4.3 dialog libraries (SA 4.2+ names, as in INGEBFLT / INGEBBLD).
sa_dialog_libs:
  - { dd: ISPLLIB, dsns: ["{{ sa_dialog_hlq }}.SINGLOAD"] }
  - { dd: ISPMLIB, dsns: ["{{ sa_dialog_hlq }}.SINGMENU", "{{ sa_ispf_hlq }}.SISPMENU"] }
  - { dd: ISPPLIB, dsns: ["{{ sa_dialog_hlq }}.SINGPENU", "{{ sa_ispf_hlq }}.SISPPENU"] }
  - { dd: ISPSLIB, dsns: ["{{ sa_dialog_hlq }}.SINGSENU", "{{ sa_ispf_hlq }}.SISPSENU"] }
  - { dd: ISPTLIB, dsns: ["{{ sa_dialog_hlq }}.SINGTENU", "{{ sa_ispf_hlq }}.SISPTENU"] }
  - { dd: SYSEXEC, dsns: ["{{ sa_dialog_hlq }}.SINGTREX"] }
  - { dd: AOFIPDB, dsns: ["{{ sa_dialog_hlq }}.SINGPDB"] }

# P3 - INGEBFLT: AOFFBFL1 pdb_name, DD:PDB, DD:PDBUPD, DD:PDBRPT
sa_textupd:
  verified: true
  max_rc: 0
  report_dsn: "{{ sa_work_pds }}.RPT"   # recreated by 30_apply_to_pdb.yml as a PDS
  source_dsn: "{{ sa_work_pds }}.UPD"   # recreated by 30_apply_to_pdb.yml, sequential
  dd:
    # Temporary table library, to keep the run away from the site AOFTABL.
    - name: AOFTABL
      spec: "SPACE=(TRK,(15,15,15)),RECFM=FB,LRECL=80,DSORG=PO"
    - name: PDB
      spec: "DISP=OLD,DSN={{ sa_clone_dsn }}"
    # Sequential: AOFFBFL1 opens it by data set name (LMINIT) and drops a
    # member name (a PDS member ended CC 12 with no report).
    - name: PDBUPD
      spec: "DISP=OLD,DSN={{ sa_work_pds }}.UPD"
    # A PDS: LMINIT ENQ(SHRW) only works on partitioned data sets
    # (SYSOUT ended CC 20, a sequential data set CC 8).
    - name: PDBRPT
      spec: "DISP=OLD,DSN={{ sa_work_pds }}.RPT"
    - { name: ISPWRK1, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK2, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK3, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK4, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPCTL0, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL1, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL2, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL3, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL4, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: SYSPRINT, spec: "SYSOUT=*" }
    - { name: SYSOUT, spec: "SYSOUT=*" }
  systsin:
    - " ISPSTART CMD(AOFFBFL1 +"
    - "              {{ sa_clone_pdb_name }}, +"
    - "              DD:PDB, +"
    - "              DD:PDBUPD, +"
    - "              DD:PDBRPT) +"
    - "              NEWAPPL(AOF) BDISPMAX(999999)"

# P4 - INGEBBLD: AOFFBBL1 type, name, ALL|MODIFIED, sysplex, option, config,
#                         DD:PDB, pdb_name, enterprise, DD:SOCNTL
# Enterprise is * so the name stored in the clone (LTEPLEX_PDB) is used;
# naming it would trip the JCL guard.
sa_build:
  verified: true
  max_rc: 0
  type: MOD
  dd:
    - name: PDB
      spec: "DISP=SHR,DSN={{ sa_clone_dsn }}"
    - name: SOCNTL
      spec: "DISP=OLD,DSN={{ sa_staging_dsn }}"
    - { name: ISPWRK1, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK2, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK3, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPWRK4, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=256,RECFM=FB" }
    - { name: ISPCTL0, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL1, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL2, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL3, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: ISPCTL4, spec: "UNIT=SYSALLDA,SPACE=(CYL,(5,10)),LRECL=80,RECFM=FB" }
    - { name: SYSPRINT, spec: "SYSOUT=*" }
    - { name: SYSOUT, spec: "SYSOUT=*" }
  systsin:
    - " ISPSTART CMD(AOFFBBL1 *,+"
    - "                       *,+"
    - "                       {{ 'ALL' if sa_build_type_effective == 'ALL' else 'MODIFIED' }},+"
    - "                       *,+"
    - "                       1,+"
    - "                       0,+"
    - "                       DD:PDB,+"
    - "                       {{ sa_clone_pdb_name }},+"
    - "                       *,+"
    - "                       DD:SOCNTL) +"
    - "                       NEWAPPL(AOF) BDISPMAX(999999)"
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/30_apply_to_pdb.yml -e @batch.yml
```

```bash
ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/40_build_staging.yml -e @batch.yml
```

### 7. Change the policy and run P3 and P4 again

P3 is not repeatable on the same clone. The text file creates the APG and APLs with `NEW`, so
a second run against a clone that already holds them fails the syntax check, and SA changes nothing:

```
ERROR (Line 7): Name DBA1MSTR already exists as entry of type Application.
WARNING: Update not run. 5 syntax error(s) detected.
```

To change the new entries (for example, after a P4 build warning) or to repeat P3 for any other
reason, start again from a fresh clone:

1. Change [vars/new_apl.yml](vars/new_apl.yml) or [templates/apl.txt.j2](templates/apl.txt.j2).
2. Optional: render the text file and check it before anything is written:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/20_render_apl.yml
   ```

3. Rebuild the clone from Backup A. This deletes the clone, copies Backup A back, and checks it
   against Backup A's `.MF` manifest. Production is not touched.

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/90_rollback.yml -e rollback_action=reclone
   ```

4. Delete the staging output, so P4 makes a full (`ALL`) build into an empty data set again,
   instead of a `MODIFIED` build on top of the previous result:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/90_rollback.yml -e rollback_action=delete_staging
   ```

5. Run P3 and P4 with the same batch file as before:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/30_apply_to_pdb.yml -e @sa_policy/output/<discover_stamp>/batch_draft.yml
   ```

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/40_build_staging.yml -e @sa_policy/output/<discover_stamp>/batch_draft.yml
   ```

How to read the result:

- The first line of each job's result shows its completion code and the flagged messages.
  P3 also prints its update report (`<HLQ>.SAPDB.WORK.RPT`). A good P3 run ends with
  `No syntax errors detected. Start writing to Policy Database.` and a `Writing data ...` line per entry.
- `INFO` lines in the report are not errors. For example, SA clears Automation and Chaining
  on HASPARENT relationships because HASPARENT does not use them.
- A P4 build can end CC 0 and still fail the playbook. Any `WARNING` in ISPLOG fails it, because
  SA reports policy problems as warnings: *"The build function completed successfully however
  some warnings have been issued."* Fix the policy and repeat steps 1 to 5. Don't raise
  `max_rc` or loosen `sa_fail_message_regex` to get past them.

  Example: an APL that inherits external startup/shutdown (INITIAL/FINAL or ALWAYS, from
  `C_DB2_DEPENDENTS`) with a HASPARENT relationship but no condition gives *"has INITIAL or
  ALWAYS specified ... none of those relationships has condition "StartsMeAndStopsMe""*. DB2
  MSTR starts and stops IRLM, DBM1 and DIST, so their relationship to MSTR has
  `condition: StartsMeAndStopsMe`.

If you only need to repeat P4 (no policy change), skip steps 1 to 3 and the P3 run. To rebuild
everything without deleting staging, pass `-e sa_build_force_all=true` to `40_build_staging.yml`.

### 8. Read the P5 validation

`50_validate.yml` compares staging with the active configuration and prints a short summary.
The full report, with diffs, is in `<HLQ>.SAPDB.WORK(P5REPORT)`; `-v` also prints every diff.
Every difference lands in one group:

| Group | Meaning | Action |
|---|---|---|
| build stamp only | Differs only in lines every build rewrites (fragment header, `* Built :`, levels, the trailing checksum lines). `sa_validate_stamp_regex` | none |
| AM configuration | `HSAZ999`, `Z999CRES`, `Z999CSCH`: binary, rebuilt every time. `sa_validate_am_regex` | none |
| stale in active | Removed members that the active `ACFZ999` does not include, left over from older builds (on LTE1: `Z97OAAPL` HSM, `Z97TAAPL` NETVSSI) | none |
| mention new entries | Members that name the new APG or APLs | none |
| explained | Listed with a reason in `sa_validate_explanations` ([vars/new_apl.yml](vars/new_apl.yml)) | none |
| TO EXPLAIN | Anything else. The playbook prints their diffs | Read the diff. If the change is expected, add the member and the reason to `sa_validate_explanations`. If not, find out why before P6 |

For the DBA1 change, the explained members are `ACFMZ999`, `ACFTZ999` and `MPFLSTSA`. They only
gain DB2 message entries (`DSNB250E`, `DSNB311I`-`DSNB351I`, `DSNJ115I`), which come from the
`C_DB2_*` classes that the build now includes because the DBA1 APLs link to them.
`sa_validate_explanations` belongs to one policy change, so empty it for the next one.

### Useful options

- `--syntax-check` validates a playbook without connecting to z/OS.
- `--list-tasks` shows what a phase will do.
- `-v` shows full module results. Use it when a job or command fails.
- Don't rely on `--check`: most `zos_*` modules don't support check mode, so it doesn't preview the data set changes.

## Before P3 and P4 can run: confirm the batch commands

IBM's public documentation does not give the batch command for the text-file update or the exact
`INGEBBLD` parameters for 4.3. The plan calls this "the spike" (SA_POLICY.md section 7). So:

1. Run `00_discover.yml`. It reads every `SYS1.ING.SINGSAMP(INGEB*)` sample and every
   `SYS1.ING.SINGISKL` skeleton that contains `ISPSTART` (the dialog builds its batch
   text-update job from one of those). The console shows a one-line index of those members
   and a numbered list of next actions. The full output goes to
   `sa_policy/output/discover_<stamp>/` on the control node (git-ignored):
   - `report.txt`: summary, next actions, and per member its title, programs, DDs, SYSTSIN and ISPSTART lines
   - `batch_draft.yml`: `sa_textupd` / `sa_build` with the candidate lines copied in as comments
   - `baselines.txt`, `datasets.txt`, `procs.txt`, `spike_full.txt`: raw command and member output

   In AAP those files don't survive the execution environment. Set `sa_discover_raw_artifacts=true`
   to add the raw output to `.artifacts.sa_discovery.raw`.
2. Fill `sa_textupd.systsin` / `dd` and `sa_build.systsin` / `dd` in `batch_draft.yml` (or in
   [vars/sa_env.yml](vars/sa_env.yml)). Point them at the clone
   (`SYSAUTO.PROTO.SAPDB.PROTO` / `LTEPLEX_PROTO`) and at the staging data set.
3. Set `verified: true` on each.

Until then, `30_apply_to_pdb.yml` and `40_build_staging.yml` stop before they submit anything.
If the commands address the PDB by name, register the clone in the Customization Dialog as
`LTEPLEX_PROTO` → `'SYSAUTO.PROTO.SAPDB.PROTO'` once.

## Safety guards

- **JCL guard:** P3 and P4 refuse JCL that names `SYSAUTO.PDB`, `LTEPLEX_PDB` or `SYSAUTO.PDB.SOCNTL*`
  on any non-comment line, and JCL lines longer than 72 columns. Only P6 production mode
  (`sa_target=production` plus the typed `sa_confirm_production`) allows production names.
- **ENQ check:** `D GRS,RES=(*,dsn)` runs before each copy or job. It fails cleanly on an
  EXCLUSIVE holder instead of waiting.
- **Copy verification:** every backup and the clone are verified member by member against
  their source. The rollback re-verifies Backup A against its `.MF` manifest before using it.
- **Job checks:** a job fails on a completion code above `max_rc` (default 0), or on any `AOF/ING/ISP/IKJ…E|W|S`
  message, `ERROR`, `WARNING` or `REXX error condition` line in its non-JES output
  (`sa_fail_message_regex`). The SA dialog writes its findings as plain text, and some
  failures still end CC 0. The output is read with ZOAU `pjdd` (DDs in `sa_job_dds`), because
  `zos_job_output` returns no DDs on LTE1.
- **Production restore:** `restore_production_pdb` needs
  `sa_confirm_production_restore="RESTORE SYSAUTO.PDB"`. It first takes a fresh safety backup,
  then restores member by member, so the data set keeps its allocation.

## Assumptions to confirm on LTE1

- `sa_work_hlq: SYSAUTO.PROTO` is a placeholder. Use an HLQ the Ansible user ID may ALTER.
- The system entry in the PDB is named `LTE1`, and the DB2 classes carry the start, stop and
  message policy. The text file only adds identity, links and relationships.
- The text file passes SA 4.3's syntax check on LTE1. The `RELATIONSHIPS` block follows IBM's
  sample `SYS1.ING.SINGSAMP(INGEFUJ3)`: 7 data lines per relationship (Relationship Type,
  Supporting Resource, Description, Sequence Number, Automation, Chaining, Condition), with a
  `*` line between blocks. If a format is wrong, P3's syntax check rejects the whole file and
  nothing is changed. The batch extract (`INGEBFXT`, `AOFFBFQ0 APL`) cannot export APLs from
  this PDB: it fails with a REXX SYNTAX error on the long TCPIP command texts. So there is no
  APL export to diff against yet.
- ISPF libraries are `SYS1.ISP.SISP*ENU` (checked on LTE1), and the SA dialog libraries are `SYS1.ING.SING*`
  (4.3, **not** `SYS1.ING.V4R4M0.*`).
- `zos_admin`-style AAP inventory provides `environment_vars`. Collection: `ibm.ibm_zos_core` 2.0.0.

## P6: activate on the live system

P6 runs the proven P3 and P4 jobs against the **production** PDB, validates the build with P5,
and then activates it. Production mode is opt-in per run and needs two extra vars:

```
-e sa_target=production -e "sa_confirm_production='ACTIVATE SYSAUTO.PDB'"
```

Only `30_apply_to_pdb.yml`, `40_build_staging.yml`, `60_activate.yml` and `90_rollback.yml`
(`refresh_previous` only) accept it. Every other playbook, and the other rollback actions, refuse
it, because there `sa_target=production` would point the clone's name at the production PDB.
In production mode the JCL guard allows production names; the column check still applies.

What happens to production, and when:

| Step | Production change |
|---|---|
| 30 (production) | `SYSAUTO.PDB` gets the DBA1 APG and APLs. Before that, `production_gate.yml` checks that the newest Backup A matches `SYSAUTO.PDB` member for member |
| 40 (production) | none: `SYSAUTO.PDB` is read and built into the work staging data set (emptied first, full build) |
| 50 | none |
| 60 | a new generation `SYSAUTO.PDB.SOCNTL(+1)` (IEBCOPY of the validated staging, allocated like the active generation: non-SMS PDSE on its volume), then `INGAMS REFRESH CFG=<new generation>` |

The GDG keeps 10 generations with SCRATCH: creating a new one deletes the oldest.
The new generation is only created after P5 passed, so there is never an empty or partial
generation (0).

Facts this relies on, checked on LTE1 (SA 4.3):

- `INGAMS REFRESH CFG=dsname` "updates an automation manager with the definition data that is
  stored in the specified configuration file. This also reloads the appropriate ACFs on the
  active automation agents" (INGAMS help). IBM recommends it only when every agent is READY, so
  60 checks `INGAMS` first.
- NetView runs the automation tables `DSITBL01`, `INGMSGSA`, `INGMSG01`, not the built
  `ACFMZ999`, and MPF is `.NO_ENTRY ... AUTO(YES)` only. The build's `ACFMZ999` / `MPFLSTSA`
  changes are therefore not loaded, and no `AUTOTBL` or `SET MPF` reload is part of P6.
- DB2 DBA1 (`DBA1MSTR`, `DBA1IRLM`, `DBA1DBM1`, `DBA1DIST`) was already running before P6. After
  the refresh, SA should find the address spaces UP, not start them.

### Run P6

In the change window, one step at a time, reading each checkpoint:

1. Fresh backups (Backup A of `SYSAUTO.PDB`, Backup B of the active configuration). The existing
   clone is replaced by a fresh copy of production:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/10_backup_clone.yml -e sa_replace_clone=true
   ```

2. Apply the text file to the production PDB:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/30_apply_to_pdb.yml -e @batch.yml -e sa_target=production -e "sa_confirm_production='ACTIVATE SYSAUTO.PDB'"
   ```

3. Build the production PDB into staging:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/40_build_staging.yml -e @batch.yml -e sa_target=production -e "sa_confirm_production='ACTIVATE SYSAUTO.PDB'"
   ```

4. Validate (no production vars). `TO EXPLAIN` must be `-`:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/50_validate.yml
   ```

5. Activate:

   ```bash
   ansible-playbook -i sa_policy/inventory/local/inventory.yml sa_policy/playbooks/60_activate.yml -e sa_target=production -e "sa_confirm_production='ACTIVATE SYSAUTO.PDB'"
   ```

   60 refuses to run unless staging was built from `SYSAUTO.PDB` (its `AOFACFMP` says so), the P5
   report for that staging passed, every agent is READY, and the active configuration is the
   newest generation. It prints the previous and new generation, the `ACF STATUS` lines and the
   `INGLIST` of the DBA1 resources.

6. Check DB2 DBA1 in SA (`INGLIST DBA1*`, `DISPSTAT`), and do the controlled `INGREQ` test agreed
   for the window with the `zos_admin/sysauto_*` playbooks.

### Roll back P6

| Failure point | Action |
|---|---|
| 30 failed or must be undone | `90_rollback.yml -e rollback_action=restore_production_pdb -e "sa_confirm_production_restore='RESTORE SYSAUTO.PDB'"` |
| 40 / 50 failed | Nothing is active. Fix, then run 40 and 50 again, or restore the PDB as above |
| After 60 | `90_rollback.yml -e rollback_action=refresh_previous -e sa_target=production -e "sa_confirm_production='ACTIVATE SYSAUTO.PDB'" -e sa_previous_cfg=<previous generation printed by 60>`, then restore the PDB as above |

After `refresh_previous`, the new generation is still the newest one of the GDG. Delete it, or
build a corrected one, before anything refreshes from the GDG base name.
