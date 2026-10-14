The key thing to know first: the running SA system never reads the PDB. It only knows the build output (the configuration files dataset). So the PDB has to be found from the authoring side: the dialog, the build JCL, or naming conventions. The running system tells you the config dataset, and that’s usually a strong hint.

All commands below are display-only.

1. Identify the tasks
D A,L                      ← find the NetView/SA agent (AUTOPROC) and the automation manager task
D A,AUTOPROC               ← proc name / step for the agent
$D PROCLIB                 ← (JES2) PROCLIB concatenation, to browse those procs
D PARMLIB                  ← PARMLIB concatenation (for the HSAPRMxx automation manager member)
2. Ask SA what configuration it’s running
F AUTOPROC,DISPSYS         ← SA level, configuration dataset, build date/time, system info
F AUTOPROC,ACF STATUS      ← loaded automation control file: dataset, member, load timestamp
F AUTOPROC,INGAMS          ← automation managers (PAM/SAM), config token/dataset in the details

INGAMS is panel-oriented, so if the MODIFY output is thin, run it from a NetView session. The configuration dataset name from these is your strongest lead: the PDB usually shares the HLQ.

3. Find the PDB dataset
Method	How
Customization dialog	Start it the way your shop does (an ISPF menu option or exec). The Policy Database Selection panel lists each PDB with its dataset name; select one to see details. This is the authoritative answer.
Existing build JCL	Search your JCL libraries for the build job: ISPF 3.14 search-for INGEBBLD, or SRCHFOR. The JCL references both the PDB and the output dataset.
Catalog search	ISPF 3.4 with **.*PDB* or the HLQ from step 2; TSO LISTCAT LEVEL('hlq') ALL
Ask	Whoever runs builds today will know. Also confirm whether there’s more than one PDB (test/prod).
4. Datasets to browse and save (read-only)
Dataset	What to capture	Notes
PDB	LISTDS 'pdb.dsn' MEMBERS, DCB attributes, space	Members are largely internal format, not readable text. Capture the structure and attributes only (needed for cloning).
Config output (from step 2)	Member list with dates; browse the members that mention your reference APL	Becomes the “before” baseline for the P5 diff
SINGSAMP	LISTDS 'hlq.SINGSAMP' MEMBERS, then look at every INGEB* member	Directly answers the plan’s spike: which dialog functions ship as batch samples
HSAPRMxx (PARMLIB)	Whole member	Automation manager parms and config references
NetView DSIPARM (CNMSTYLE/CNMSTUSR and SA includes)	SA-related statements	Agent initialization settings
PROCLIB	Agent proc, automation manager proc, and the reference STC’s proc	Template for the new STC’s proc
Site build JCL	The full job	Template for the batch build step
5. SA display commands for a reference APL

Pick an existing APL closest to what you’ll add (similar STC, same system/APG). In this example it’s REFAPL on LTE1:

F AUTOPROC,INGINFO REFAPL/APL/LTE1,OUTMODE=LINE       ← status, class, relationships, history
F AUTOPROC,DISPACF REFAPL                              ← loaded policy entries: start/stop cmds, messages
F AUTOPROC,ACF ENTRY=REFAPL,TYPE=*,REQ=DISP            ← raw ACF entries for the subsystem
F AUTOPROC,DISPINFO REFAPL                             ← job name, proc, parms, thresholds
F AUTOPROC,INGRELS REFAPL/APL/LTE1                     ← dependencies (verify OUTMODE support)
F AUTOPROC,DISPTREE REFAPL                             ← dependency tree
F AUTOPROC,DISPFLGS                                    ← automation flags

Baseline for the whole system:

F AUTOPROC,INGLIST */APL/LTE1,OUTMODE=LINE
F AUTOPROC,INGLIST */APG/LTE1,OUTMODE=LINE
F AUTOPROC,DISPSTAT
F AUTOPROC,INGVOTE

DISPACF and the ACF ... REQ=DISP output are the most useful ones here. They show the built form of a real APL, which you’ll later diff against the build output for the new one.

6. The most valuable artifact: a text export from the dialog

In the customization dialog, open the PDB, go to Data Management, and use the text-file function to export the reference APL. Also export its APG and any Application class it links to. Optionally generate a policy Report for those entries.

This export is the exact keyword/value format the prototype will generate, so it’s the #1 input for the next chat.

7. Saving it all to files
Console output: in SDSF LOG (or ULOG), find the responses and use XDC to print to a dataset. Alternatively, run the commands with Ansible’s zos_operator and write the returned content to local files:
  ansible zos_host -m ibm.ibm_zos_core.zos_operator -a "cmd='F AUTOPROC,DISPSYS'"
Dataset members: ISPF 3.3 copy to a work PDS, or zos_fetch to pull them to your workstation.
Dataset attributes: LISTDS 'dsn' MEMBERS and LISTCAT ENTRIES('dsn') ALL output, saved from the TSO session or via zos_tso_command.

Suggested bundle for the next chat:

DISPSYS / ACF STATUS / INGAMS output
PDB and config dataset names, attributes, and member lists
INGEB* member list from SINGSAMP, plus your build JCL
HSAPRMxx
The text exports of the reference APL, APG, and class
DISPACF / ACF REQ=DISP / INGINFO for the reference APL
The INGLIST / DISPSTAT baselines

Mask anything sensitive (user IDs, passwords in start commands) before sharing.