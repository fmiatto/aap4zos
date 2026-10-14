This is real data from a z/OS LPAR Running IBM z/OS System Automation

D A,L

RESPONSE=LTE1                                                                
 CNZ4105I 01.22.00 DISPLAY ACTIVITY 526                                      
  JOBS     M/S    TS USERS    SYSAS    INITS   ACTIVE/MAX VTAM     OAS       
 00014    00027    00001      00034    00031    00001/00060       00036      
  VLF      VLF      VLF      NSW  S  LLA      LLA      LLA      NSW  S       
  AUTOSSI  AUTOSSI  NETVIEW  NSW  S  AUTOPROC AUTOPROC NETVIEW  NSW  SO      
  AUTOMAN  AUTOMAN  AMSA     NSW  SO ATRRRS   ATRRRS   RRS      NSW  S       
  BBGZANGL BBGZANGL STEP1    OWT  SO CICS62   CICS62            NSW  SO      
  RACF     RACF     RACF     NSW  S  IZUANG1  IZUANG1  STEP1    OWT  SO      
  JES2     JES2     IEFPROC  NSW  S  RSED     RSED     RSED     IN   SO      
  JMON     JMON     JMON     OWT  SO SDSF     SDSF     SDSF     NSW  S       
  VTAM     VTAM     NET      NSW  SO SDSFAUX  SDSFAUX  SDSFAUX  NSW  SO      
  RSEAPI   RSEAPI   RSEAPI   OWT  SO SYSLOGD  STEP1    OMVSKERN NSW  AO      
  TCPIP    TCPIP    TCPIP    NSW  SO TSO      TSO      TCAS     OWT  S       
  RSEAPI1  *OMVSEX  STCAPI   IN   AO DBA1MSTR DBA1MSTR IEFPROC  NSW  S       
  DBA1IRLM DBA1IRLM          NSW  S  DBA1DBM1 DBA1DBM1 IEFPROC  NSW  S       
  PAGENT   PAGENT   PAGENT   OWT  SO DBA1DIST DBA1DIST IEFPROC  NSW  SO      
  TN3270   TN3270   STEP01   NSW  SO TN3270R  TN3270R  STEP01   NSW  SO      
  SSHD3    STEP1    OMVSKERN OWT  AO FTPSERVE STEP1    TCPIP    OWT  AO      
  TN3270S  TN3270S  STEP01   NSW  SO RSED7    STEP1    STCRSE   OWT  AO      
  RSED8    STEP1    STCRSE   OWT  AO RSED9    STEP1    STCRSE   IN   AO      
 RSED1    STEP1    STCRSE   OWT  AO RSED1    STEP1    STCRSE   IN   AO
 RSED2    STEP1    STCRSE   OWT  AO SSHD1    *OMVSEX  OMVSKERN OWT  AO
 SSHD3    STEP1    OMVSKERN IN   AO ZFS01    FORMAT            OU*  JO
 SSHD5    STEP1    ZOSADMN  NSW  AO                                   
ZOSADMN  IN              

D O 

RESPONSE=LTE1                                                      
 IEE603I 01.22.37 OPDATA DISPLAY 530                               
  PREFIX     OWNER      SYSTEM     SCOPE     REMOVE   FAILDSP      
  $          JES2       LTE1       SYSTEM    NO       SYSPURGE     
  -DBA1      DBA1MSTR   LTE1       SYSTEM    NO       SYSPURGE     
  %SAE1      NetView    LTE1       SYSTEM    NO       PURGE        
  @          RACF       LTE1       SYSTEM    NO       PURGE        
  REXXE1     AXR        LTE1       SYSTEM    NO       PURGE        

D A,AUTOPROC

RESPONSE=LTE1                                                           
 CNZ4106I 23.44.46 DISPLAY ACTIVITY 245                                 
  JOBS     M/S    TS USERS    SYSAS    INITS   ACTIVE/MAX VTAM     OAS  
 00010    00022    00001      00033    00026    00001/00060       00027 
  AUTOPROC AUTOPROC NETVIEW  NSW  SO  A=001F   PER=NO   SMC=000         
                                      PGN=N/A  DMN=N/A  AFF=NONE        
                                      CT=091.407S  ET=98.17.17          
                                      WUID=STC00128 USERID=AUTOPROC     
                                      WKL=STC_WKL  SCL=NEWWORK  P=1     
                                      RGP=N/A      SRVR=NO  QSC=NO      
                                      ADDR SPACE ASTE=7EEA87C0          
                                      DSPNAME=AOFTREE  ASTE=7CEBAE80    
                                      DSPNAME=CNMCANZ1 ASTE=7D9B1080    

$D PROCLIB

 RESPONSE=LTE1                                                     
  $HASP319 PROCLIB(PROC00)                                         
  $HASP319 PROCLIB(PROC00)    STATIC LIBRARY,                      
  $HASP319                    DD(1)=(DSNAME=SYS1.USER.PROCLIB,     
  $HASP319                    VOLSER=USER01,UNIT=SYSALLDA),        
  $HASP319                    DD(2)=(DSNAME=SYS1.PROCLIB,          
  $HASP319                    VOLSER=LTECAT,UNIT=SYSALLDA),        
  $HASP319                    DD(3)=(DSNAME=SYS1.IBM.PROCLIB,      
  $HASP319                    VOLSER=LA2RS1,UNIT=SYSALLDA)         
  $HASP319 PROCLIB(PROC01)                                         
  $HASP319 PROCLIB(PROC01)    STATIC LIBRARY,                      
  $HASP319                    DD(1)=(DSNAME=SYS1.USER.PROCLIB,     
  $HASP319                    VOLSER=USER01,UNIT=SYSALLDA)         
$HASP319 PROCLIB(PROC02)                                    
$HASP319 PROCLIB(PROC02)    STATIC LIBRARY,                 
$HASP319                    DD(1)=(DSNAME=SYS1.PROCLIB,     
$HASP319                    VOLSER=LTECAT,UNIT=SYSALLDA)    
$HASP319 PROCLIB(PROC03)                                    
$HASP319 PROCLIB(PROC03)    STATIC LIBRARY,                 
$HASP319                    DD(1)=(DSNAME=SYS1.IBM.PROCLIB, 
$HASP319                    VOLSER=LA2RS1,UNIT=SYSALLDA)    

D PARMLIB

RESPONSE=LTE1                                 
 IEE251I 23.46.50 PARMLIB DISPLAY 272         
  PARMLIB DATA SETS SPECIFIED                 
  AT IPL                                      
  ENTRY  FLAGS  VOLUME  DATA SET              
    1      S    USER01  SYS1.USER.PARMLIB     
    2      S    LTECAT  SYS1.PARMLIB          
    3      S    LA2RS1  SYS1.IBM.PARMLIB      

F AUTOPROC,DISPSYS                                        
 System           : LTE1                                  
 Domain           : SAE1                                  
 Sysplex Group    : LTEPLEX                               
 XCF Group  name  : INGXSG                                
 XCF PlexID       : INGPX$00                              
                                                          
 Software                                                 
    Operating System   : z/OS 03.01.00                    
    NetView            : IBM Z NetView V6R4               
       Tower(s)        : SA                               
    Automation         : IBM Z System Automation V4R3M0   
       Tower(s)        : SYSOPS PROCOPS                   
 Configuration                                               
    Data set     : SYSAUTO.PDB.SOCNTL.G0016V00(ACFZ999)      
    Built by     : ZECOADM 04/10/26 18:46:31                 
    Activated    : 10/09/26 21:27:56                         
    CFG Token    : 2026031714561900111A388561                
                                                             
 Message automation table(s)                                 
     DSITBL01 INGMSGSA INGMSG01                              
                                                             
 Flags                                                       
    Automation   :Yes                                        
    Init Start   :Yes                                        
    Start        :Yes                                        
    Recovery     :Yes                                        
    Terminate    :Yes                                        
    Restart      :Yes                                        
                                                             
 Scheduling Subsystem    : JES2                              
    Type                 : JES2                              
                                                             
 Root for SDF updates    : LTE1                              
   USS path        : /usr/lpp/ing/ussauto/lib/                               
  Inform list     : SDF                                                     
                                                                            
  IPL Complete Considerations                                               
    Time Limit    : - None -                                                
    Expected Statuses :                                                     
                    AVAILABLE                                               
    Important Resources :                                                   
                    - None -                                                
                                                                            
  Captured Messages  (maximum is 20)                                        
    2026-10-09 21:29:25 :                                                   
       AOF570I 21:29:25 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN1'" FOR MVSE    
       SMFDUMP - MSGTYPE IS ALWAYS                                          
    2026-10-09 21:43:11 :                                                   
       AOF570I 21:43:11 : ISSUED "MVS DD ALLOC=ACTIVE" FOR MVSESA MVSDUM    
       MSGTYPE IS MVSDUMPRESET                                              
    2026-10-10 14:34:10 :                                                   
       AOF570I 14:34:10 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN2'" FOR MVSE    
       SMFDUMP - MSGTYPE IS ALWAYS                                          
    2026-10-11 09:30:00 :                                                  
       AOF570I 09:30:00 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN1'" FOR MVSE   
       SMFDUMP - MSGTYPE IS ALWAYS                                         
    2026-10-12 04:50:09 :                                                  
       AOF570I 04:50:09 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN2'" FOR MVSE   
       SMFDUMP - MSGTYPE IS ALWAYS                                         
    2026-10-12 23:59:04 :                                                  
       AOF570I 23:59:04 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN1'" FOR MVSE   
       SMFDUMP - MSGTYPE IS ALWAYS                                         
    2026-10-13 19:08:10 :                                                  
       AOF570I 19:08:10 : ISSUED "MVS S SMFDUMP,DA='LTE1.MAN2'" FOR MVSE   
       SMFDUMP - MSGTYPE IS ALWAYS                                         

F AUTOPROC,ACF STATUS 
RESPONSE=LTE1                                                       
 NETVIEW SAE1  "                                                    
 AOF005I MEMBER ACFZ999 CURRENTLY BEING USED FOR THE CONTROL FILE   
 AOF006I BUILT     BY ZECOADM  ON 04/10/26 AT 18:46:31              
 AOF006I LOADED    BY AUTINIT1 ON 10/09/26 AT 21:27:56              
 AOF006I CONFIGURATION TOKEN   = 2026031714561900111A388561         
 AOF006I CONFIGURATION DATASET = SYSAUTO.PDB.SOCNTL.G0016V00        
 AOF002I END OF MULTI-LINE MESSAGE GROUP                            

F AUTOPROC,INGAMS
 RESPONSE=LTE1                                                                 
  Sysplex = LTEPLEX                                                            
  System   Member    Role  Status     Sysplex  XCF Group Release Comm E2E      
  -------- --------- ----- ---------- -------- --------- ------- ---- ---      
  LTE1     LTE1$$$$1 PAM   READY      LTEPLEX  INGXSG    V4R3M0  XCF           
  LTE1     LTE1      AGENT READY      LTEPLEX  INGXSG    V4R3M0  XCF           
  *** End of Display ***                                                       

DATASETS

SYSAUTO.PDB  is the correct PDB active in the system

LISTDS 'SYSAUTO.PDB' MEMBERS 
SYSAUTO.PDB                   
--RECFM-LRECL-BLKSIZE-DSORG   
  FB    80    32720   PO      
--VOLUMES--                   
  LTECAT                      
  --MEMBERS--                       
  AOFDENT                         
  AOFENCON                        
  AOFENJE2                        
  AOFENSET                        
  AOFGW                           
  AOFJ2DR                         
  AOFMSGCL                        
  AOFMVSCO                        
  AOFNEFWD                        
  AOFSSDEF                        
  AOFSUBSY                        
  AOFTADF                         
  AOFTAF                          
  AOFTAOP                         
  AOFTAOX                         
  AOFTAPG                         
  AOFTAPL                         
  AOFTASY                         
  AOFTATY                         
  AOFTCXL                         
  AOFTC2I                         
  AOFTDMN                         
  AOFTDXL                         
  AOFTENT                         
  AOFTEQC                         
AOFTEQJ 
AOFTEQR 
AOFTEQS 
AOFTETP 
AOFTEVT 
AOFTFAS 
AOFTFDT 
AOFTFLG 
AOFTGRP 
AOFTHRHD
AOFTIMS 
AOFTIST 
AOFTLCJ 
AOFTLCL 
AOFTMDF 
AOFTMNR 
AOFTMSA 
AOFTMSG 
AOFTMSO 
AOFTMSX 
AOFTMTR 
AOFTNFO 
AOFTNFY 
AOFTNTW 
AOFTOCS 
AOFTODM 
AOFTOEN
AOFTOPR
AOFTOSD
AOFTOSO
AOFTOSR
AOFTOXL
AOFTPAC
AOFTPLP
AOFTPRO
AOFTREF
AOFTREL
AOFTRES
AOFTREX
AOFTRSC
AOFTR2R
AOFTSAF
AOFTSAS
AOFTSBG
AOFTSCR
AOFTSCX
AOFTSDF
AOFTSND
AOFTSOA
AOFTSVP
AOFTSYS
AOFTTMR
AOFTTMX 
AOFTTPA 
AOFTTPX 
AOFTTRG 
AOFTUET 
AOFTUEX 
AOFTUEY 
AOFTUSS 
AOFTXDF 
AOFTX2Y 
RSMITH01
ZECOADM1

RSMITH01                  0  2025/07/14  2025/07/14 11:35:52  RSMITH 
ZECOADM1                  5  2026/04/10  2026/04/10 18:45:08  ZECOADM

No dates for other members

 BROWSE    SYSAUTO.PDB(ZECOADM1) - 01.01            Line 0000000000 Col 001 080 
 Command ===>                                                  Scroll ===> CSR  
********************************* Top of Data **********************************
IF MSGID = '$HASP395' &                                                 00010000
   JOBNAME = 'TCPRESET' &                                               00020000
   TEXT =  .' ENDED - RC=0000' THEN                                     00030000
EXEC(CMD('ISSUEACT SYSTEMMSG=YES SEL=TCPRESET')                         00040001
 ROUTE(ONE %AOFOPGSSOPER%));                                            00050001



SYSAUTO.PDB                         
SYSAUTO.PDB.BKUP                    
SYSAUTO.PDB.BKUP.G0001V00           
SYSAUTO.PDB.BKUP.G0002V00           
SYSAUTO.PDB.BKUP.G0003V00           
SYSAUTO.PDB.LOG1                    
SYSAUTO.PDB.REP                     
SYSAUTO.PDB.SAVE                    
SYSAUTO.PDB.SOCNTL                  
SYSAUTO.PDB.SOCNTL.BUILD            
SYSAUTO.PDB.SOCNTL.G0007V00         
SYSAUTO.PDB.SOCNTL.G0008V00         
SYSAUTO.PDB.SOCNTL.G0009V00         
SYSAUTO.PDB.SOCNTL.G0010V00         
SYSAUTO.PDB.SOCNTL.G0011V00         
SYSAUTO.PDB.SOCNTL.G0012V00         
SYSAUTO.PDB.SOCNTL.G0013V00         
SYSAUTO.PDB.SOCNTL.G0014V00     
SYSAUTO.PDB.SOCNTL.G0015V00     
SYSAUTO.PDB.SOCNTL.G0016V00     
SYSAUTO.PDB.SOCNTL.SAVE         

SYS1.ING.SINGSAMP        
SYS1.ING.V4R4M0.SINGSAMP 

 IBM Z System Automation 4.3 Customization Dialog  

 ISPF Menu => Z.SA

  To switch to another Policy Database, specify the Policy Database name    
 in the following field, or specify a ? to get a selection list.           
 Current Policy Database . . . LTEPLEX_PDB                                 

   COMMANDS  ACTIONS  VIEW  HELP                                                
────────────────────────────────────────────────────────────────────────────── 
                              Entry Name Selection       Policy in BROWSE mode 
Command ===>                                                  Scroll ===> CSR  
                                                                               
Entry Type : Application           PolicyDB Name   : LTEPLEX_PDB               
                                   Enterprise Name : LTEPLEX_PDB               
                                                                               
Action      Entry Name             C  Short Description                        
            APPC                      Advanced Peer-to-Peer Communication      
            ASCH                      APPC Scheduler                           
            ATRRS                     Resource Recovery Services               
            AUTOIPLC                  IPL Data Gatherer                        
            AUTOMAN                   SA Automation Manager                    
            AUTOPROC                  Automation Application                   
            AUTOSSI                   Automation Subsystem Interface           
            BLSJPRMI                  Build SNAP Tables for IPCS               
            C_AM                   *  Class for Automation Manager Definitions 
            C_APPL                 *  Class for general APL definitions        
            C_CICS                 *  Class for CICS                           
            C_CICS_CMAS            *  Class for CICS CPSMCMAS                  
            C_CICS_SHAREDSERVERS   *  Class for CICS Shared Servers            
            C_CICS_TG              *  Class for CICS Transaction Gateway       
            C_CICS_WUI             *  Class for CICS CPSMWUI                   
            C_CICS_XACTIONSERVER   *  Class for CICS Transaction Server        
C_DB2_DEPENDENTS       *  Class for DB2 DIST, DBM1, IRLM, ADMT     
C_DB2_LITE             *  Class for DB2 MSTR - light start mode    
C_DB2_MSTR             *  Class for DB2 MSTR - basic definitions   
C_DB2_NORM             *  Class for DB2 MSTR - normal start mode   
C_DVIPA                *  Class for common DVIPA definitions       
C_JES2                 *  Class for Job Entry Subsystem 2          
C_USS_APPL             *  Class for USS Application                
C_USS_FILE             *  Class for USS Files                      
C_USS_PORT             *  Class for USS Port                       
CBDQDISP                  HCM Dispatcher                           
CEA                                                                
CICS0001                  prod CICS                                
DFHSM                     Hierarchical Storage Manager             
DFRMM                                                              
DLF                       Data Lookaside Facility                  
DSIRQJOB                  NetView JES-JobID-Requestor              
FTPSERVE                                                           
HZR                       Runtime Diagnostics (RTD)                
HZSPROC                   z/OS Health Checker                      
ICSF                      Integrated Cryptographic Serv. Facility  
INETD                                                              
IRRDPTAB                  RACF dynamic parse table loader          
IZUANG1                   z/OSMF angel process                     
IZUSVR1                   z/OSMF server                            
JES2                      Job Entry Subsystem 2                    
JMON                      IBM Explorer for z/OS                    
LLA                       Library Lookaside                        
OMVS                      Unix System Services subsystem           
PAGENT                    TCPIP Policy Agent for AT-TLS policy     
RACF                      Resource Access Control Facility        
RESOLVER                  TCP/IP Name Resolver                    
RMF                       Resource Measurement Facility           
RMFGAT                    RMF Monitor III Data Gatherer           
RPCBIND                   Portmapper                              
RSEAPI                    IBM Explorer for z/OS                   
RSED                      IBM Explorer for z/OS                   
SDSF                      System Display and Search Facility      
SDSFAUX                   SDSF enhanced system data collector     
SMF_REC                   SMF recording                           
SSHD                                                              
SYSLOGD                   Logging Services Daemon                 
TCPIP                     TCP/IP                                  
TN_PORT                   Port which is used by TELNET server     
TN3270                    TELNET Server for 3270                  
TN3270R                   TELNET Server for 3270                 
TN3270S                   TELNET Server for 3270                 
TSO                       Time Sharing Option                    
VLF                       Virtual Lookaside Facility             
VTAM                      Virtual Telecommunication Access Method
ZFS                       z/OS File System                       

                                Policy Selection       Policy has been browsed  
Command ===>                                                  Scroll ===> CSR   
                                                                                
Entry Type : Application             PolicyDB Name   : LTEPLEX_PDB              
Entry Name : FTPSERVE                Enterprise Name : LTEPLEX_PDB              
                                                                                
Action      Policy Name          Policy Description                             
            MESSAGES/USER DATA   Define messages and user data                  
            STARTUP              Define startup procedures                      
            SHUTDOWN             Define shutdown procedures                     
            THRESHOLDS           Define error thresholds                        
            MINOR RESOURCES      Define application minor flags and thresholds  
            SYSTEM ASSOCIATION   Define primary and secondary associations      
            -------------------- -----RESOURCES-------------------------------  
            GENERATED RESOURCES  List resources generated for this entry        
            MEMBER OF            List resources where this entry is a member    
            -------------------- ---------------------------------------------  
            WHERE USED           List application groups linked to this entry   
            COPY                 Copy data from an existing entry               

 ────────────────────────────────────────────────────────────────────────────── 
                             Policy Database Selection          Row 1 to 1 of 1 
 Command ===>                                                   SCROLL===> CSR  
                                                                                
 Action     Policy Database      Enterprise Name/Data Set Name                  
            LTEPLEX_PDB          LTEPLEX_PDB                                    
                                 'SYSAUTO.PDB'                                  

