# SA z/OS: useful commands and data

The running SA system never reads the PDB. It only knows the build output (the configuration
data set it loaded). The PDB is found from the authoring side: the dialog, the build JCL, or
naming conventions; the configuration data set the running system shows usually shares its
high-level qualifier.

All commands below are display-only.

## 1. The SA tasks

```text
D A,L                      find the NetView/SA agent (AUTOPROC) and the automation manager
D A,AUTOPROC               proc name and step of the agent
$D PROCLIB                 (JES2) PROCLIB concatenation, to browse those procs
D PARMLIB                  PARMLIB concatenation (HSAPRMxx automation manager member)
```

## 2. The configuration SA runs

```text
F AUTOPROC,DISPSYS         SA level, configuration data set, build date/time, system info
F AUTOPROC,ACF STATUS      loaded automation control file: data set, member, load time
F AUTOPROC,INGAMS          automation managers (PAM/SAM), configuration in the details
```

INGAMS is panel-oriented; if the MODIFY output is thin, run it from a NetView session.

## 3. The PDB data set

| Method | How |
|---|---|
| Customization dialog | The Policy Database Selection panel lists each PDB with its data set name |
| Build JCL | Search the JCL libraries for INGEBBLD (ISPF 3.14, SRCHFOR): the job names the PDB and the output data set |
| Catalog | ISPF 3.4 with `**.*PDB*` or the HLQ of the configuration data set; `LISTCAT LEVEL('hlq') ALL` |

On LTE1: PDB `SYSAUTO.PDB` (`LTEPLEX_PDB`), configuration GDG `SYSAUTO.PDB.SOCNTL`, change log
`SYSAUTO.PDB.LOG1` / `LOG2` (named in the PDB member `AOFTIST`).

## 4. Data sets worth looking at

| Data set | What to look at |
|---|---|
| PDB | Member list and DCB attributes; the members are SA's internal format, not text |
| Configuration (step 2) | `$BLDRPT` (build report: which member was built for which entry), the entry fragments (`Z9nnAAPL`, ...) and their include list `ACFZ999`, `ACFMZ999` (automation table entries), `MPFLSTSA` |
| `hlq.SINGSAMP` | IBM samples: `INGEBFLT` (text-file update), `INGEBBLD` (build), `INGEBFXT` (text-file export), `INGEFUJ3` (sample text file) |
| HSAPRMxx (PARMLIB) | Automation manager parameters |
| NetView DSIPARM (CNMSTYLE/CNMSTUSR, SA includes) | Agent initialization |
| PROCLIB | Agent, automation manager and application procs |

## 5. One application in detail

For an APL `REFAPL` on LTE1:

```text
F AUTOPROC,INGINFO REFAPL/APL/LTE1,OUTMODE=LINE       status, class, relationships, history
F AUTOPROC,DISPACF REFAPL                              loaded policy: start/stop commands, messages
F AUTOPROC,ACF ENTRY=REFAPL,TYPE=*,REQ=DISP            raw ACF entries of the subsystem
F AUTOPROC,DISPINFO REFAPL                             job name, proc, parameters, thresholds
F AUTOPROC,INGRELS REFAPL/APL/LTE1,OUTMODE=LINE       relationships both ways, with their conditions
F AUTOPROC,DISPTREE REFAPL                             dependency tree (APL-to-APL relationships only)
F AUTOPROC,DISPFLGS                                    automation flags
```

DISPTREE draws only relationships defined between APLs. Dependencies defined on application
groups (`HASPARENT <group>/APG/=`, as in `ims15.yml`) apply to the group's members but do not
appear there; `INGRELS` shows them, for an APG or a member:

```text
F AUTOPROC,INGRELS IMS15_CTRL/APG/LTE1,OUTMODE=LINE    F = the group's parents, B = its dependents
F AUTOPROC,INGRELS IMS15CTL/APL/LTE1,OUTMODE=LINE      HasMember: the dependency it gets from its group
```

A graphical view of resources and relationships (NetView Management Console, NMC) needs
RODM, GMFHS and SA's topology manager (`INGTOPO`) on the host; LTE1 runs none of them.

The whole system:

```text
F AUTOPROC,INGLIST */APL/LTE1,OUTMODE=LINE
F AUTOPROC,INGLIST */APG/LTE1,OUTMODE=LINE
F AUTOPROC,DISPSTAT
F AUTOPROC,INGVOTE
```

## 6. Text exports

A text export shows an entry exactly in the format `templates/apl.txt.j2` writes. In the
customization dialog: open the PDB, `5 Data Management` → `3 Update via File` →
`1 Write selected data from Policy Database to file`, and select the entries (APL, its APG and
its class). Quote the output data set name, or ISPF adds your TSO prefix.

The batch export `SINGSAMP(INGEBFXT)` (CLIST `AOFFBFQ0`) exports every entry of the given
types (`APL`, `APG`, ...), with no name filter. A text-file record holds at most 255
characters; on LTE1 the APL export stops with an abend in AOFFZIPS on entries whose PIPE
commands are longer (the TCPIP ones). The APG export works.

## 7. Saving output

- Console output: SDSF LOG (or ULOG) and XDC to a data set, or Ansible:
  `ansible zos_host -m ibm.ibm_zos_core.zos_operator -a "cmd='F AUTOPROC,DISPSYS'"`
- Members: ISPF 3.3 to a work PDS, or `zos_fetch`; from z/OS UNIX, `dcat 'DSN(MEMBER)'` (ZOAU).
- Attributes: `LISTDS 'dsn' MEMBERS`, `LISTCAT ENTRIES('dsn') ALL` (TSO, or `zos_tso_command`).

Mask anything sensitive (user IDs, passwords in start commands) before sharing.
