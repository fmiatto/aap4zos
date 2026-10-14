# Prototype Plan — Adding a Permanent APL to the SA z/OS Policy Database

**Goal:** Prove that a new Application (APL) can be added to the SA z/OS Policy Database (PDB) without anyone working in the dialog, built into configuration files in batch, and validated, with a tested rollback at every step. The process must be repeatable from Ansible.

**Scope of the prototype:** discovery, backup, authoring, applying to a **cloned** PDB, batch build into a **staging** output dataset, and validation.
**Out of scope for the prototype:** activating on a live system (`INGAMS REFRESH`). This is designed here but runs only after the prototype is proven.

> Ground rule from IBM: policy objects must be created through the Customization Dialog's own functions. The build output (ACF/AMC configuration files) must **never** be edited by hand.

---

## 1. Inputs to bring to the next chat

| # | Item | Why it's needed |
|---|---|---|
| 1 | SA z/OS release and service level (e.g. 4.3 + APARs) | Function availability, keyword formats |
| 2 | PDB dataset name(s) and the policy database name as listed in the dialog | Read, clone and back up |
| 3 | Text export of **one existing, similar APL** (dialog Data Management → export to text) | Shows the exact keyword/value format the update function expects |
| 4 | Text export of the APG it belongs to, plus any Application **class** it links to | Membership and inheritance |
| 5 | The build JCL in use today (`INGEBBLD` copy from SINGSAMP or a site version) | Batch build template |
| 6 | Active configuration dataset name (the one referenced in HSAPRMxx / agent startup) | Must not be touched in the prototype; needed for the rollback design |
| 7 | Current `INGLIST` / `DISPSTAT` output for the target system | Baseline |
| 8 | Details of the new STC (see section 4) | The actual change |
| 9 | Ansible setup: `ibm.ibm_zos_core` version, z/OS host inventory, HLQ allowed for work datasets | Execution environment |

---

## 2. Phases overview

```
P0 Discovery (read-only)
  → P1 Backup and clone
    → P2 Author APL text file
      → P3 Apply to CLONED PDB
        → P4 Batch build to STAGING output
          → P5 Validate
            → (later) P6 Activate + verify   /   P7 Rollback
```

Every phase ends with a checkpoint: what was produced, how to verify it, and how to undo it.

---

## 3. Phase details

### P0 — Discovery (read-only)
- Catalog the PDB datasets: attributes (DSORG, RECFM, LRECL, space) and member list. Use `zos_find` / `zos_data_set` facts and `LISTDS`/`LISTCAT` via `zos_tso_command`.
- Capture the active configuration dataset's member list and change dates as the baseline.
- Capture `INGLIST` and `DISPSTAT` baselines via `zos_operator` (`F AUTOPROC,...`).
- Confirm nobody holds the PDB in the dialog, and identify the ENQ resource names the dialog uses.
- **Spike (critical unknown):** find out how the text-file update ("Making Updates with Text Files" in *Defining Automation Policy → Data Management*) can run without a person:
  - check SINGSAMP for an `INGEB*` batch sample for import/update;
  - if none, drive the dialog in **ISPF batch** (`IKJEFT01` + `ISPSTART`, with the SA dialog libraries allocated).
  - The answer determines how P3 is implemented.

**Output:** discovery report (datasets, members, baselines, chosen P3 mechanism).

### P1 — Backup and clone
- **Backup A (golden, read-only):** copy the PDB to `<HLQ>.SAPDB.BKP.D<yymmdd>.T<hhmm>` with IEBCOPY via `zos_copy` / `zos_mvs_raw`, or use `zos_backup_restore` (DFDSS dump).
- **Backup B:** copy the active configuration dataset the same way.
- **Clone (work copy):** a second copy of the PDB, `<HLQ>.SAPDB.PROTO`. **All prototype changes go here, so the production PDB is never modified during the prototype.**
- Record member counts and checksums (e.g. a hash of each member's contents) for each backup in a manifest file.

**Rollback:** delete the clone. Production is untouched.

### P2 — Author the APL as a text file
- Use the exported sample APL (input #3) as the format reference, and create a Jinja2 template (`templates/apl.txt.j2`).
- Fill it from a vars file (`vars/new_apl.yml`) holding the fields in section 4.
- Prefer **linking to an existing Application class**, so the new APL inherits most policy and the text file only carries the differences.
- Include the APG membership update and system/class links. These may be separate entries in the text file.
- Store the template and vars in Git; the rendered file is a build artifact.

**Checkpoint:** a diff between the rendered file and the sample export shows only the intended differences.

### P3 — Apply the text file to the cloned PDB
- Upload the rendered file to a work dataset with `zos_copy`.
- Run the update with the mechanism chosen in P0 (batch sample, or ISPF batch driving the dialog function) against `<HLQ>.SAPDB.PROTO`.
- Capture the job output and the update log/report; fail on any warning or error message.
- Re-export the new APL from the clone and diff it against the rendered input (round-trip check).

**Rollback:** re-clone from Backup A.

### P4 — Batch build to a STAGING output dataset
- Allocate a new, empty staging output dataset `<HLQ>.SAPDB.PROTO.SOCNTL` with the same attributes as the active one.
- Tailor `INGEBBLD`: point it at the clone, write to staging, and set the build options below.
- Build options:
  - The first build into an empty output dataset **must be a full build** (IBM rule).
  - Later iterations can use a build scoped to the single sysplex or system to save time.
- Submit with `zos_job_submit` and collect output with `zos_job_output`. Pass only on the expected RC, and parse the build messages.

**Rollback:** delete the staging dataset. The active configuration is untouched.

### P5 — Validate
- Compare the staging output to the active configuration dataset. Expect new or changed fragments only for the new APL, its APG, and the system include member; anything else is unexpected and must be explained.
- Confirm the new subsystem appears in the build output for the right system.
- Optionally generate a dialog policy report for the new entry from the clone (batch or ISPF batch) for human review.
- **Optional, only on a sandbox SAplex:** activate the staging dataset there and run `INGLIST` / `DISPSTAT` / `INGINFO` on the new APL, then a test `INGREQ START`/`STOP`.

**Exit criteria for the prototype:**
- P3 and P4 complete without errors.
- The round-trip diff is clean.
- The output diff contains only the expected changes.
- Every rollback step has been executed at least once.

---

## 4. New APL data checklist (vars file contents)

| Area | Fields |
|---|---|
| Identity | Entry name, subsystem name, job name, job type (STC/MVS), category, description |
| Inheritance | Application class to link (if any) |
| Placement | System(s) or system class to link, APG(s) to join, APG type (BASIC/MOVE/SERVER) |
| Startup | Start command or proc parameters, startup parms, Initstart flag, start delay/timeout |
| Shutdown | NORM / IMMED / FORCE command sequences, shutdown delay |
| Messages | UP message ID(s), final/termination message ID(s), error message IDs and actions, capture settings |
| Relationships | e.g. `HASPARENT` TCPIP/APL, `PREPAVAILABLE`, sequencing |
| Automation flags | Automation, Initstart, Start, Recovery, Shutdown, Restart |
| Thresholds | CRIT / FREQ / INFR |
| Monitoring | Monitor routine / health check (if used) |
| Notify | Inform list (SDF, IOM, EIF) |

**Non-SA prerequisites** (separate tasks, same pipeline): PROCLIB member, RACF `STARTED` profile and user ID, WLM classification.

---

## 5. Activation and rollback design (for after the prototype)

### P6 — Activate
1. Repeat P1 (fresh backups) against **production** artifacts.
2. Apply the same rendered text file to the **production** PDB.
3. Build into a **new** output dataset `...SOCNTL.V<n>`, not over the active one.
4. Activate that dataset with `INGAMS REFRESH`, or REST/Zowe `activate policy <dsn>`.
5. Verify with `INGLIST`, `DISPSTAT`, `INGINFO`, and a controlled `INGREQ START`. Compare against the P0 baseline: the only expected difference is the new resource.

### P7 — Rollback
| Failure point | Action |
|---|---|
| Before activation | Restore the PDB from Backup A; delete the new output dataset |
| After activation | Refresh with the **previous** output dataset (`V<n-1>`); restore the PDB from Backup A; verify the resource is gone from `INGLIST` |
| Message-automation fragments changed | Reload the previous automation table / MPF members if the build touched them (`AUTOTBL`, `SET MPF=`) |

Keep the last N output versions and PDB backups, with a retention job to prune them.

---

## 6. Proposed Ansible structure

```
sa-policy/
├── inventory/
├── vars/new_apl.yml
├── templates/
│   ├── apl.txt.j2
│   └── ingebbld.jcl.j2
├── playbooks/
│   ├── 00_discover.yml
│   ├── 10_backup_clone.yml
│   ├── 20_render_apl.yml
│   ├── 30_apply_to_pdb.yml
│   ├── 40_build_staging.yml
│   ├── 50_validate.yml
│   ├── 60_activate.yml        # later
│   └── 90_rollback.yml
└── roles/
    └── sa_policy_common/      # dataset naming, manifest, message parsing
```

Main modules from `ibm.ibm_zos_core`: `zos_data_set`, `zos_copy`, `zos_fetch`, `zos_find`, `zos_backup_restore`, `zos_job_submit`, `zos_job_output`, `zos_mvs_raw`, `zos_tso_command`, `zos_operator`.

---

## 7. Risks and open questions

- **Unattended text-file update:** confirm whether a batch sample exists or whether ISPF batch is required. This is the main technical risk (P0 spike).
- **PDB contention:** the dialog ENQs the PDB. Batch steps must handle an ENQ conflict and fail cleanly instead of waiting forever.
- **Keyword format** for text files must match the installed SA release exactly. Always derive it from a live export, never from documentation alone.
- **Build scope:** a partial build does not always propagate link changes. Use a full build when links change, or when in doubt.
- **Space:** output datasets and backups need enough space to avoid an x37 abend mid-build.
- **Security:** the Ansible user ID needs UPDATE on the clone and staging datasets only during the prototype. Grant production access separately for P6.
- **Message automation:** confirm whether the build regenerates automation-table/MPF fragments, and whether those need a separate reload.