#!/usr/bin/bash

################################################################################
##aix oracle 11g rac
##version:3.1
##20180510 created
##20180512 modify function:prt_dbalert function:prt_backup
##20180513 add function:env_expect function:env_ssh function:env_user
##uesage:
##su - root
##root ssh 互信
##mkdir /check
##copy script in /check
##modify some variables
##modify backup direct path
##run script
################################################################################

##function to init the script
##example:init_script
function init_script(){
##init_script begin
##switch for output
machineinfo_switch=on
network_switch=on
timezone_switch=on
timesync_switch=on
filesystem_switch=on
systemerr_switch=on
crsnodes_switch=on
votedisk_switch=on
ocr_switch=on
diagwait_switch=on
crs_resource_switch=on
cluster_alert_switch=on
crsdlog_switch=on
cssdlog_switch=on
ocrback_switch=on
diskgroup_switch=on
asmalert_switch=on
dbversion_switch=on
sga_switch=on
controlfile_switch=on
redo_switch=on
session_switch=on
tablespace_switch=on
virus_switch=on
dbuser_switch=on
dg_switch=on
dbalert_switch=on
listener_switch=on
lsnrlog_switch=off
backup_switch=on

##global var
node1=urp-db1
node2=urp-db2
user=root
checkdir=/check
time=`date +%Y%m%d%H%M%S`
rpt=$checkdir/report_$time.out
htmlrpt=$checkdir/report_$time.html
dbhome=/u01/app/oracle/product/11.2.0/db_1
gridhome=/u01/app/grid/product/11.2.0/grid_1
db_group=`cat /etc/oratab|grep $dbhome|grep -v Backup|awk -F ':' '{print $1}'`
tnsport=1521
rmanlog_list=/check/rmanlog.lst
smile=$checkdir/smile.txt

##logs
node1_cluster_alert=/u01/app/grid/product/11.2.0/grid_1/log/$node1/alert$node1.log
node1_crsdlog=/u01/app/grid/product/11.2.0/grid_1/log/$node1/crsd/crsd.log
node1_cssdlog=/u01/app/grid/product/11.2.0/grid_1/log/$node1/cssd/ocssd.log
node2_cluster_alert=/u01/app/grid/product/11.2.0/grid_1/log/$node2/alert$node2.log
node2_crsdlog=/u01/app/grid/product/11.2.0/grid_1/log/$node2/crsd/crsd.log
node2_cssdlog=/u01/app/grid/product/11.2.0/grid_1/log/$node2/cssd/ocssd.log
node1_asmalert=/u01/app/oracle/diag/asm/+asm/+ASM1/trace/alert_+ASM1.log
node2_asmalert=/u01/app/oracle/diag/asm/+asm/+ASM2/trace/alert_+ASM2.log
dbalert_base=/u01/app/oracle/diag/rdbms
node1_lsnrlog=/u01/app/oracle/diag/tnslsnr/$node1/listener/alert/log.xml
node2_lsnrlog=/u01/app/oracle/diag/tnslsnr/$node2/listener/alert/log.xml
backupbase=/backup
logicbase=/backup/oracle/logic_backup
##init_script end
}

##function to check environment:user
##example:env_user
function env_user(){
local me=`whoami`
if [ $user != $me ];then
  echo "wisedu-0001:not expected user"
  exit 1
fi
}

##function to check environment:ssh
##example:env_ssh
function env_ssh(){
ssh $node2 date>>/dev/null
if [ $? -ne 0 ];then
  echo "wisedu-002:cannot ssh $node2 without password"
  exit 1
fi
}

##function to check environment:expect
##example:env_expect
function env_expect(){
ls -lrt /usr/bin/expect>>/dev/null
if [ $? -ne 0 ];then
  echo "wisedu-003:do not have expect package"
  exit 1
fi
}

##function to print title
##example prt_title 'title'
function prt_title(){
local title=$1
cat>>$rpt<<EOF


################################################################################
##             $title                                    
################################################################################
EOF
}

##function to print machine infomation
##example:prt_machine switch
function prt_machine(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   machine infomation..."
  prt_title "$node1 machine infomation"
  prtconf|sed -n '1,/RESOURCE/p'>>$rpt
  prt_title "$node2 machine infomation"
  ssh $node2 prtconf|sed -n '1,/RESOURCE/p'>>$rpt
fi
}

##function to print network
##example:prt_network switch
function prt_network(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   network..."
  prt_title "$node1 network"
  ifconfig -a>>$rpt
  prt_title "$node2 network"
  ssh $node2 ifconfig -a>>$rpt
fi
}

##function to print time zone
##example prt_timezone switch
function prt_timezone(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   time zone..."
  prt_title "$node1 time zone"
  grep TZ /etc/environment>>$rpt
  prt_title "$node2 time zone"
  ssh $node2 grep TZ /etc/environment>>$rpt
fi
}

##function to print time sync
##example prt_timesync switch
function prt_timesync(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   time sync..."
  prt_title "time sync"
  echo $node1>>$rpt;date>>$rpt;echo $node2>>$rpt;ssh $node2 date>>$rpt
fi
}

##function to print filesystem
##example:prt_filesystem switch
function prt_filesystem(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   filesystem..."
  prt_title "$node1 filesystems"
  df -g>>$rpt
  prt_title "$node2 filesystems"
  ssh $node2 df -g>>$rpt
fi
}

##function to print system error log
##example:prt_systemerr switch
function prt_systemerr(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   system error..."
  prt_title "$node1 system error log"
  errpt|tail -200>>$rpt
  prt_title "$node2 system error log"
  ssh $node2 errpt|tail -200>>$rpt
fi
}

##function to print cluster nodes
##example:prt_crsnodes switch
function prt_crsnodes(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   cluster nodes..."
  prt_title "cluster nodes"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/olsnodes -n -i>>$rpt
fi
}

##function to print votedisk
##example:prt_votedisk switch
function prt_votedisk(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   votedisk..."
  prt_title "votedisk"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/crsctl query css votedisk>>$rpt
fi
}

##function to print ocr disk
##example:prt_ocr switch
function prt_ocr(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   ocr..."
  prt_title "ocr disk"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/ocrcheck>>$rpt
fi
}

##function to print css diagwait
##example:prt_diagwait switch
function prt_diagwait(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   css diagwait..."
  prt_title "css diagwait status"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/crsctl get css diagwait>>$rpt
fi
}

##function to print cluster resource details
##example:prt_crs_resource switch
function prt_crs_resource(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   crs resource details..."
  prt_title "crs resource details"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/crsctl status res -t>>$rpt
fi
}

##function to print cluster logs
##example:prt_clusterlog switch1 switch2 switch3
function prt_clusterlog(){
local cluster_alert_switch=$1
local crsdlog_switch=$2
local cssdlog_switch=$3
if [ $cluster_alert_switch == 'on' ];then
  echo "   $node1 cluster alert..."
  prt_title "$node1 cluster alert"
  tail -200 $node1_cluster_alert>>$rpt
fi
if [ $crsdlog_switch == 'on' ];then
  echo "   $node1 crsd log..."
  prt_title "$node1 crsd log"
  tail -200 $node1_crsdlog>>$rpt
fi
if [ $cssdlog_switch == 'on' ];then
  echo "   $node1 cssd log..."
  prt_title "$node1 cssd log"
  tail -200 $node1_cssdlog>>$rpt
fi
if [ $cluster_alert_switch == 'on' ];then
  echo "   $node2 cluster alert..."
  prt_title "$node2 cluster alert"
  ssh $node2 tail -200 $node2_cluster_alert>>$rpt
fi
if [ $crsdlog_switch == 'on' ];then
  echo "   $node2 crsd log..."
  prt_title "$node2 crsd log"
  ssh $node2 tail -200 $node2_crsdlog>>$rpt
fi
if [ $cssdlog_switch == 'on' ];then
  echo "   $node2 cssd log..."
  prt_title "$node2 cssd log"
  ssh $node2 tail -200 $node2_cssdlog>>$rpt
fi
}

##function to print ocr backup
##example:prt_ocrback switch
function prt_ocrback(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   ocr back..."
  prt_title "backup of ocr"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  $ORACLE_HOME/bin/ocrconfig -showbackup>>$rpt
fi
}

##function to print asm disk group
##example:prt_diskgroup switch
function prt_diskgroup(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   disk group..."
  prt_title "asm disk group"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  ORACLE_SID=+ASM1;export ORACLE_SID
  /usr/bin/expect<<EOF>>$rpt
  spawn su grid
  expect "grid@$node1"
  send "$ORACLE_HOME/bin/asmcmd lsdg\r"
  expect "grid@$node1"
  send "\n"
  send "exit\r"
  expect eof
EOF
fi
}

##function to print asm instance alert
##example:prt_asmalert switch
function prt_asmalert(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   asm alert..."
  prt_title "$node1 asm instance alert"
  tail -200 $node1_asmalert>>$rpt
  prt_title "$node2 asm instance alert"
  ssh $node2 tail -200 $node2_asmalert>>$rpt
fi
}

##function to print database verion
##example: prt_dbversion switch database
function prt_dbversion(){
local switch=$1
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   database version..."
  prt_title "database version"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  select * from v\$version;
  quit;
EOF
fi
}

##function to print sga status
##example:prt_sga switch database
function prt_sga(){
local switch=$1
local service_name=$2
local instance1=$2'1'
local instance2=$2'2'
if [ $switch == 'on' ];then
  echo "   $database sga..."
  prt_title "$instance1 sga"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance1;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  show sga;
  quit;
EOF
  prt_title "$instance2 sga"
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump@$node2:$tnsport/$service_name<<EOF>>$rpt
  show sga;
  quit;
EOF
fi
}

##function to print controlfile
##example:prt_controlfile switch database
function prt_controlfile(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database controlfile..."
  prt_title "$database controlfile"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  select name from v\$controlfile;
  quit;
EOF
fi
}

##function to print redo log
##example:prt_redo switch database
function prt_redo(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database redo logs..."
  prt_title "$database redo log"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  set lines 400
  set pagesize 50
  col member format a47
  col status format a8
  select a.group#,a.member,b.status,b.bytes/1024/1024 "M" from v\$logfile a,v\$log b where a.group# = b.group# order by group#;
  quit;
EOF
fi
}

##function to print session
##example:prt_session switch database
function prt_session(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database sessions..."
  prt_title "$database sessions"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  select inst_id,count(*) from gv\$session group by inst_id;
  quit;
EOF
  prt_title "$database session by schema"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  set lines 400
  col machine format a40
  select inst_id,username,count(*)  from gv\$session
  where machine not in ('$node1','$node2')
  group by inst_id,username
  order by username;
  quit;
EOF
fi
}

##function to print tablespace
##example:prt_tablespace switch tablespace
function prt_tablespace(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database tablespace..."
  prt_title "$database tablespaces"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  set pagesize 50
  set lines 400
  col talbespace_name format a20
  col status format a10
  col auto format a5
  select a.tablespace_name, total, free, (total-free)/total*100 as "used%",c.file_id,c.status,c.M,c.maxM,c.incM,c.auto from 
  (select tablespace_name, sum(bytes)/1024/1024 as total from dba_data_files group by tablespace_name) a, 
  (select tablespace_name, sum(bytes)/1024/1024 as free from dba_free_space group by tablespace_name) b,
  (select a.file_id,a.tablespace_name,a.bytes/1024/1024 as M,b.status,a.maxbytes/1024/1024 as maxM,(a.increment_by*b.block_size)/1024/1024 as incM,a.autoextensible as auto
  from dba_data_files a,v\$datafile b
  where a.file_id=b.file#) c
  where a.tablespace_name = b.tablespace_name
  and a.tablespace_name=c.tablespace_name
  order by tablespace_name;
  quit;
EOF
fi
}

##function to print virus
##example:prt_virus switch database
function prt_virus(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database virus..."
  prt_title "$database known virus"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  select 'there has virus if something outputs' readme from dual;
  select owner,object_name,object_type from dba_objects where object_name like '%DBMS_SUPPORT_DBMONITOR%' order by object_name;
  select owner,object_name,object_type from dba_objects where object_name like '%DBMS_SUPPORT_INTERNAL%' order by object_name;
  select owner,object_name,object_type from dba_objects where object_name like '%DBMS_SYSTEM_INTERNAL%' order by object_name;
  select owner,object_name,object_type from dba_objects where object_name like '%DBMS_STANDARD_FUN9%' order by object_name;
  select owner,object_name,object_type from dba_objects where object_name like '%DBMS_CORE_INTERNAL%' order by object_name;
  quit;
EOF
fi
}

##function to print database users
##example:prt_dbuser switch database
function prt_dbuser(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database users..."
  prt_title "$database users"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  set lines 400
  set pagesize 80
  col username format a20
  col created format a20
  col profile format a10
  col expiry_date format a20
  col resource_name format a25
  col limit format a10
  col lock_date format a25
  col account_status format a20
  select a.username,a.created,a.profile,a.expiry_date,b.resource_name,b.limit,a.lock_date,a.account_status
  from dba_users a,dba_profiles b
  where a.profile=b.profile
  and b.resource_type like '%PASSWORD%'
  and b.resource_name like '%PASSWORD_LIFE_TIME%'
  and a.username not in ('SYS','SYSTEM','OUTLN','MGMT_VIEW','FLOWS_FILES','MDSYS','ORDSYS','EXFSYS','DBSNMP','WMSYS','APPQOSSYS','APEX_030200','OWBSYS_AUDIT','ORDDATA','CTXSYS','ANONYMOUS','SYSMAN','XDB','ORDPLUGINS','OWBSYS','SI_INFORMTN_SCHEMA','OLAPSYS','SCOTT','ORACLE_OCM','XS$NULL','MDDATA','DIP','APEX_PUBLIC_USER','SPATIAL_CSW_ADMIN_USR','SPATIAL_WFS_ADMIN_USR');
  quit;
EOF
fi
}

##function to print dataguard scn
##example prt_dg switch database
function prt_dg(){
local switch=$1
local database=$2
local instance=$2'1'
if [ $switch == 'on' ];then
  echo "   $database dataguard status..."
  prt_title "$database dataguard scn"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  $ORACLE_HOME/bin/sqlplus -s usr_dump/usr_dump<<EOF>>$rpt
  select 'pri' as db,to_char(checkpoint_change#,'9999999999999') scn from v\$database
  union all
  select 'std' as db,to_char(applied_scn,'9999999999999') from v\$archive_dest where dest_id=2;
  quit;
EOF
fi
}

##function to print database alert
##example:prt_dbalert switch database
function prt_dbalert(){
local switch=$1
local database=$2
local instance1=$2'1'
local instance2=$2'2'
if [ $switch == 'on' ];then
  echo "   $database alert..."
  prt_title "$instance1 alert"
  tail -200 $dbalert_base/$database/$instance1/trace/alert_$instance1.log>>$rpt
  prt_title "$instance2 alert"
  ssh $node2 tail -200 $dbalert_base/`echo $database|tr '[:upper:]' '[:lower:]'`/$instance2/trace/alert_$instance2.log>>$rpt
fi
}

##function to print listener
##example:prt_listener switch
function prt_listener(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   listener status..."
  prt_title "$node1 listener"
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  ORACLE_SID=+ASM1;export ORACLE_SID
  $ORACLE_HOME/bin/lsnrctl status>>$rpt
  prt_title "$node2 listener"
  ##这地方只配ssh等效性没用，只能用expect了
  /usr/bin/expect<<EOF>>$rpt
  spawn ssh -o "StrictHostKeyChecking no" root@$node2
  expect "root@$node2"
  send "su - grid\r"
  expect "grid@$node2"
  send "lsnrctl status\r"
  send "\n"
  expect "grid@$node2"
  send "exit\r"
  expect "root@$node2"
  send "exit\r"
  expect "\n"
  expect eof
EOF
fi
}

##function to print listener log
##example:prt_lsnrlog switch
function prt_lsnrlog(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   listener log..."
  prt_title "$node1 listener log"
  tail -200 $node1_lsnrlog>>$rpt
  prt_title "$node2 listener log"
  ssh $node2 tail -200 $node2_lsnrlog>>$rpt
fi
}

##function to print database backup
##example:prt_backup switch database
function prt_backup(){
local switch=$1
local database=$2
local instance=$2'1'
local fullback_dir=$backupbase/`echo $database|tr '[:upper:]' '[:lower:]'`/fullbackup
local archback_dir=$backupbase/`echo $database|tr '[:upper:]' '[:lower:]'`/archbackup
local ctlback_dir=$backupbase/`echo $database|tr '[:upper:]' '[:lower:]'`/ctlbackup
local rmanlog_dir=$backupbase/`echo $database|tr '[:upper:]' '[:lower:]'`/logs
local logicdir=$logicbase
local logiclog=`ls -lrt $logicdir|grep expdb|tail -1|awk '{print $9}'`
if [ $switch == 'on' ];then
  echo "   $database rman backup..."
  prt_title "$database backup"
  ORACLE_HOME=$dbhome;export ORACLE_HOME
  ORACLE_SID=$instance;export ORACLE_SID
  ##这里光设环境变量没用，又要使用expect
  /usr/bin/expect <<EOF>>$rpt
  spawn su oracle
  expect "oracle@$node1"
  send "$ORACLE_HOME/bin/rman target /\r"
  expect "RMAN>"
  send "show all;\r"
  expect "RMAN>"
  send "quit;\r"
  expect "oracle@$node1"
  expect eof
EOF
  echo "########ls fullback#########">>$rpt
  ls -lrt $fullback_dir>>$rpt
  echo "########ls archback#########">>$rpt
  ls -lrt $archback_dir>>$rpt
  echo "########ls ctlback#########">>$rpt
  ls -lrt $ctlback_dir>>$rpt
  last_full=`ls -lrt $rmanlog_dir|grep full|tail -1|awk '{print $9}'`
  ls -lrt $rmanlog_dir|grep rman|awk '{print $9}'|sed -n '/'"$last_full"'/,$p'>$rmanlog_list
  while read line
  do
    echo "################################################################################">>$rpt
	echo "     $line"
	echo $line>>$rpt
	echo "################################################################################">>$rpt
	cat $rmanlog_dir/$line>>$rpt
	echo -e "\n">>$rpt
	echo -e "\n">>$rpt
  done<$rmanlog_list
  echo "   $database logic backup..."
  echo "########ls logic backup#########">>$rpt
  ls -lrt $logicdir|tail -50>>$rpt
  echo "################################################################################">>$rpt
  echo "     $logiclog"
  echo $logiclog>>$rpt
  echo "################################################################################">>$rpt
  cat $logicdir/$logiclog>>$rpt
fi
}

##function to check os
##example:chk_os
function chk_os(){
echo "checking os waiting..."
prt_machine $machineinfo_switch
prt_network $network_switch
prt_timezone $timezone_switch
prt_timesync $timesync_switch
prt_filesystem $filesystem_switch
prt_systemerr $systemerr_switch
}

##function to check cluster
##example:chk_cluster
function chk_cluster(){
echo "checking cluster waiting..."
prt_crsnodes $crsnodes_switch
prt_votedisk $votedisk_switch
prt_ocr $ocr_switch
prt_diagwait $diagwait_switch
prt_crs_resource $crs_resource_switch
prt_clusterlog $cluster_alert_switch $crsdlog_switch $cssdlog_switch
prt_ocrback $ocrback_switch
prt_diskgroup $diskgroup_switch
prt_asmalert $asmalert_switch
}

##function to check database by options
##example:chk_db_byopt
function chk_db_byopt(){
echo "checking database waiting..."
local db=`echo $db_group|awk '{print $1}'`
prt_dbversion $dbversion_switch $db

for database in $db_group;
do
  prt_sga $sga_switch $database;
done

for database in $db_group;
do
  prt_controlfile $controlfile_switch $database;
done

for database in $db_group;
do
  prt_redo $redo_switch $database;
done

for database in $db_group;
do
  prt_session $session_switch $database;
done

for database in $db_group;
do
  prt_tablespace $tablespace_switch $database;
done

for database in $db_group;
do
  prt_virus $virus_switch $database;
done

for database in $db_group;
do
  prt_dbuser $dbuser_switch $database;
done

for database in $db_group;
do
  prt_dg $dg_switch $database;
done

for database in $db_group;
do
  prt_dbalert $dbalert_switch $database;
done
}

##function to check database by databases including backup
##example:chk_db_bydb
function chk_db_bydb(){
echo "checking database waiting..."
local db=`echo $db_group|awk '{print $1}'`
prt_dbversion $dbversion_switch $db
for database in $db_group;
do
  prt_sga $sga_switch $database
  prt_controlfile $controlfile_switch $database
  prt_redo $redo_switch $database
  prt_session $session_switch $database
  prt_tablespace $tablespace_switch $database
  prt_virus $virus_switch $database
  prt_dbuser $dbuser_switch $database
  prt_dg $dg_switch $database
  prt_dbalert $dbalert_switch $database
  prt_backup $backup_switch $database
done
}

##function to check listener
##example:chk_lsnr
function chk_lsnr(){
echo "checking listener waiting..."
prt_listener $listener_switch
prt_lsnrlog $lsnrlog_switch
}

##function to check database backup
##example:chk_backup
function chk_backup(){
echo "checking database backup waiting..."
for database in $db_group;
do
  prt_backup $backup_switch $database
done
}

##function to fuck the user
##example:fuck
function fuck(){
cat<<EOF>$smile
           /'-/
          /--/ 
         /--/ 
     /-/'--'/'¯'')
    /'/--/—-/—--/-\ 
   ('(----- ¯-/'--') 
    \-----'—--/'-') 
    '\'----_--' -)
     \------( --)
      \----- .................fuck!!!
EOF
cat $smile
}

##function to init html
##example:html_begin
function html_begin(){
cat<<EOF>>$htmlrpt
<html lang="en"><head><title>inspect $time</title>
<style type="text/css">
body.awr {font:bold 10pt Arial,Helvetica,Geneva,sans-serif;color:black; background:White;}
pre.awr {font:14pt Courier;color:black; background:#FFFFCC;}
pre.log {font:12pt Courier;color:black; background:#FFFFCC;}
h1.awr {font:bold 20pt Arial,Helvetica,Geneva,sans-serif;color:#336699;background-color:White;border-bottom:1px solid #cccc99;margin-top:0pt; margin-bottom:0pt;padding:0px 0px 0px 0px;}
h2.awr {font:bold 18pt Arial,Helvetica,Geneva,sans-serif;color:#336699;background-color:White;margin-top:4pt; margin-bottom:0pt;}
h3.awr {font:bold 16pt Arial,Helvetica,Geneva,sans-serif;color:#336699;background-color:White;margin-top:4pt; margin-bottom:0pt;}
h4.awr {font:bold 14pt Arial,Helvetica,Geneva,sans-serif;color:#336699;background-color:White;margin-top:4pt; margin-bottom:0pt;}
table.tdiff {  border_collapse: collapse; }
th.awrnobg {font:bold 8pt Arial,Helvetica,Geneva,sans-serif; color:black; background:White;padding-left:4px; padding-right:4px;padding-bottom:2px}
th.awrbg {font:bold 8pt Arial,Helvetica,Geneva,sans-serif; color:White; background:#0066CC;padding-left:4px; padding-right:4px;padding-bottom:2px}
td.awrnc {font:8pt Arial,Helvetica,Geneva,sans-serif;color:black;background:White;vertical-align:top;}
td.awrc {font:8pt Arial,Helvetica,Geneva,sans-serif;color:black;background:#FFFFCC; vertical-align:top;}
td.log {font:12pt Arial,Helvetica,Geneva,sans-serif;color:black;background:#FFFFCC;vertical-align:top;border:0pt}
a.awr {font:bold 8pt Arial,Helvetica,sans-serif;color:#663300; vertical-align:top;margin-top:0pt; margin-bottom:0pt;}
li.awr {font: 8pt Arial,Helvetica,Geneva,sans-serif; color:black; background:White;}
</style></head>
<body class="awr">
<h1 class="awr">INSPECT $db_group $time</h1>
<a class="awr" name="top"></a>
<h2 class="awr">Main Report</h2>
<ul>
<li class="awr"><a class="awr" href="#platform">platform</a></li>
<li class="awr"><a class="awr" href="#cluster">cluster</a></li>
<li class="awr"><a class="awr" href="#database">database</a></li>
<li class="awr"><a class="awr" href="#backup">backup</a></li>
</ul>
EOF
}

##function to print machine info in html format
##example:html_machine switch
function html_machine(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   machine infomation..."
  echo "<a class=\"awr\" name=\"machine\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">machine infomation</h3>">>$htmlrpt
  local node1_machineinfo=`prtconf|sed -n '1,/RESOURCE/p'`
  local node2_machineinfo=`ssh $node2 prtconf|sed -n '1,/RESOURCE/p'`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 machine infomation</th>
  <th class="awrbg" scope="col">$node2 machine infomation</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$node1_machineinfo</pre></td>
  <td scope="row" class='awrc'><pre class="awr">$node2_machineinfo</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#platform">Back to platform</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print network in html format
##example:html_network switch
function html_network(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   network..."
  echo "<a class=\"awr\" name=\"network\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">network</h3>">>$htmlrpt
  local node1_network=`ifconfig -a`
  local node2_network=`ssh $node2 ifconfig -a`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 network</th>
  <th class="awrbg" scope="col">$node2 network</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$node1_network</pre></td>
  <td scope="row" class='awrc'><pre class="awr">$node2_network</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#platform">Back to platform</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print time zone in html format
##example:html_timezone switch
function html_timezone(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   time zone..."
  echo "<a class=\"awr\" name=\"timezone\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">time zone</h3>">>$htmlrpt
  local node1_timezone=`grep TZ /etc/environment`
  local node2_timezone=`ssh $node2 grep TZ /etc/environment`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 time zone</th>
  <th class="awrbg" scope="col">$node2 time zone</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$node1_timezone</pre></td>
  <td scope="row" class='awrc'><pre class="awr">$node2_timezone</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#platform">Back to platform</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print filesystem in html format
##example:html_filesystem switch
function html_filesystem(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   filesystem..."
  echo "<a class=\"awr\" name=\"filesystem\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">filesystem</h3>">>$htmlrpt
  local node1_filesystem=`df -g`
  local node1_fs_wc=`df -g|wc -l`
  local node2_filesystem=`ssh $node2 df -g`
  local node2_fs_wc=`ssh $node2 df -g|wc -l`
  #node1 filesystem
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col" colspan="7">$node1 filesystem</th></tr>
  <tr><th class="awrbg" scope="col">Filesystem</th>
  <th class="awrbg" scope="col">GB blocks</th>
  <th class="awrbg" scope="col">Free</th>
  <th class="awrbg" scope="col">%Used</th>
  <th class="awrbg" scope="col">Iused</th>
  <th class="awrbg" scope="col">%Iused</th>
  <th class="awrbg" scope="col">Mounted on</th></tr>
EOF
  for((l=2;l<=$node1_fs_wc;l++));do
    echo "<tr>">>$htmlrpt
	col1=`df -g|sed -n "$l"'p'|awk '{print $1}'`
	col2=`df -g|sed -n "$l"'p'|awk '{print $2}'`
	col3=`df -g|sed -n "$l"'p'|awk '{print $3}'`
	col4=`df -g|sed -n "$l"'p'|awk '{print $4}'`
	col5=`df -g|sed -n "$l"'p'|awk '{print $5}'`
	col6=`df -g|sed -n "$l"'p'|awk '{print $6}'`
	col7=`df -g|sed -n "$l"'p'|awk '{print $7}'`
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col1</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col2</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col3</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col4</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col5</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col6</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col7</pre></td>">>$htmlrpt
	echo "</tr>">>$htmlrpt
  done
  echo "</table><br />">>$htmlrpt
  #node2 filesystem
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col" colspan="7">$node2 filesystem</th></tr>
  <tr><th class="awrbg" scope="col">Filesystem</th>
  <th class="awrbg" scope="col">GB blocks</th>
  <th class="awrbg" scope="col">Free</th>
  <th class="awrbg" scope="col">%Used</th>
  <th class="awrbg" scope="col">Iused</th>
  <th class="awrbg" scope="col">%Iused</th>
  <th class="awrbg" scope="col">Mounted on</th></tr>
EOF
  for((l=2;l<=$node2_fs_wc;l++));do
    echo "<tr>">>$htmlrpt
	col1=`df -g|sed -n "$l"'p'|awk '{print $1}'`
	col2=`df -g|sed -n "$l"'p'|awk '{print $2}'`
	col3=`df -g|sed -n "$l"'p'|awk '{print $3}'`
	col4=`df -g|sed -n "$l"'p'|awk '{print $4}'`
	col5=`df -g|sed -n "$l"'p'|awk '{print $5}'`
	col6=`df -g|sed -n "$l"'p'|awk '{print $6}'`
	col7=`df -g|sed -n "$l"'p'|awk '{print $7}'`
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col1</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col2</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col3</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col4</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col5</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col6</pre></td>">>$htmlrpt
	echo "<td scope=\"row\" class=\'awrc\'><pre>$col7</pre></td>">>$htmlrpt
	echo "</tr>">>$htmlrpt
  done
  echo "</table><br />">>$htmlrpt
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#platform">Back to platform</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print system error in html format
##example:html_systemerr switch
function html_systemerr(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   system error..."
  echo "<a class=\"awr\" name=\"systemerr\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">system error</h3>">>$htmlrpt
  local node1_systemerr=`errpt|tail -200`
  local node2_systemerr=`ssh $node2 errpt|tail -200`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 system error</th>
  <th class="awrbg" scope="col">$node2 system error</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$node1_systemerr</pre></td>
  <td scope="row" class='awrc'><pre class="awr">$node2_systemerr</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#platform">Back to platform</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print platform in html format
##example:html_platform
function html_platform(){
echo "checking platform waiting..."
cat <<EOF>>$htmlrpt
<a class="awr" name="platform"></a>
<h2 class="awr">platform</h2>
<ul>
<li class="awr"><a class="awr" href="#machine">machine infomation</a></li>
<li class="awr"><a class="awr" href="#network">network</a></li>
<li class="awr"><a class="awr" href="#timezone">time zone</a></li>
<li class="awr"><a class="awr" href="#filesystem">filesystem</a></li>
<li class="awr"><a class="awr" href="#systemerr">system error log</a></li>
</ul>
EOF
html_machine $machineinfo_switch
html_network $network_switch
html_timezone $timezone_switch
html_filesystem $filesystem_switch
html_systemerr $systemerr_switch
}

##function to print cluster nodes in html format
##example:html_crsnodes switch
function html_crsnodes(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   cluster nodes..."
  echo "<a class=\"awr\" name=\"crsnodes\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">cluster nodes</h3>">>$htmlrpt
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  local crsnodes=`$ORACLE_HOME/bin/olsnodes -n -i`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">cluster nodes</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$crsnodes</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#cluster">Back to cluster</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print votedisk in html format
##example:html_votedisk switch
function html_votedisk(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   votedisk..."
  echo "<a class=\"awr\" name=\"votedisk\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">votedisk</h3>">>$htmlrpt
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  local votedisk=`$ORACLE_HOME/bin/crsctl query css votedisk`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">votedisk</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$votedisk</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#cluster">Back to cluster</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print ocr in html format
##example:html_ocr switch
function html_ocr(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   ocr..."
  echo "<a class=\"awr\" name=\"ocr\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">ocr</h3>">>$htmlrpt
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  local ocr=`$ORACLE_HOME/bin/ocrcheck`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">ocr</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$ocr</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#cluster">Back to cluster</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print diagwait in html format
##example:html_diagwait
function html_diagwait(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   css diagwait..."
  echo "<a class=\"awr\" name=\"diagwait\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">css diagwait</h3>">>$htmlrpt
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  local diagwait=`$ORACLE_HOME/bin/crsctl get css diagwait`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">css diagwait</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$diagwait</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#cluster">Back to cluster</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print cluster resource in html format
##example:html_crs_resource switch
function html_crs_resource(){
local switch=$1
if [ $switch == 'on' ];then
  echo "   crs resource details..."
  echo "<a class=\"awr\" name=\"crs_resource\"></a>">>$htmlrpt
  echo "<h3 class=\"awr\">cluster resource details</h3>">>$htmlrpt
  ORACLE_HOME=$gridhome;export ORACLE_HOME
  local crs_resource=`$ORACLE_HOME/bin/crsctl status res -t`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">cluster resource details</th></tr>
  <tr><td scope="row" class='awrc'><pre class="awr">$crs_resource</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#cluster">Back to cluster</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print cluster logs in html format
##example:html_clusterlog switch1 switch2 switch3
function html_clusterlog(){
local cluster_alert_switch=$1
local crsdlog_switch=$2
local cssdlog_switch=$3
echo "   cluster logs..."
cat <<EOF>>$htmlrpt
<a class="awr" name="clusterlog"></a>
<h3 class="awr">cluster logs</h3>
<ul>
<li class="awr"><a class="awr" href="#node1_crsalert">$node1 cluster alert</a></li>
<li class="awr"><a class="awr" href="#node1_crsdlog">$node1 crsd log</a></li>
<li class="awr"><a class="awr" href="#node1_cssdlog">$node1 cssd log</a></li>
<li class="awr"><a class="awr" href="#node2_crsalert">$node2 cluster alert</a></li>
<li class="awr"><a class="awr" href="#node2_crsdlog">$node2 crsd log</a></li>
<li class="awr"><a class="awr" href="#node2_cssdlog">$node2 cssd log</a></li>
</ul>
EOF
##node1 cluster alert
if [ $cluster_alert_switch == 'on' ];then
  echo "     $node1 cluster alert..."
  echo "<a class=\"awr\" name=\"node1_crsalert\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node1 cluster alert</h4>">>$htmlrpt
  tail -200 $node1_cluster_alert>>$smile
  local node1_crsalert=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 cluster alert</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node1_crsalert</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
##node1 crsd log
if [ $crsdlog_switch == 'on' ];then
  echo "     $node1 crsd log..."
  echo "<a class=\"awr\" name=\"node1_crsdlog\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node1 crsd log</h4>">>$htmlrpt
  tail -200 $node1_crsdlog>>$smile
  local node1_crsd_log=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 crsd log</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node1_crsd_log</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
##node1 cssd log
if [ $cssdlog_switch == 'on' ];then
  echo "     $node1 cssd log..."
  echo "<a class=\"awr\" name=\"node1_cssdlog\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node1 cssd log</h4>">>$htmlrpt
  tail -200 $node1_cssdlog>>$smile
  local node1_cssd_log=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 cssd log</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node1_cssd_log</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
##node2 cluster alert
if [ $cluster_alert_switch == 'on' ];then
  echo "     $node2 cluster alert..."
  echo "<a class=\"awr\" name=\"node2_crsalert\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node2 cluster alert</h4>">>$htmlrpt
  ssh $node2 tail -200 $node2_cluster_alert>>$smile
  local node2_crsalert=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node2 cluster alert</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node2_crsalert</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
##node2 crsd log
if [ $crsdlog_switch == 'on' ];then
  echo "     $node2 crsd log..."
  echo "<a class=\"awr\" name=\"node2_crsdlog\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node2 crsd log</h4>">>$htmlrpt
  ssh $node2 tail -200 $node2_crsdlog>>$smile
  local node1_crsd_log=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node1 crsd log</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node1_crsd_log</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
##node2 cssd log
if [ $cssdlog_switch == 'on' ];then
  echo "     $node2 cssd log..."
  echo "<a class=\"awr\" name=\"node2_cssdlog\"></a>">>$htmlrpt
  echo "<h4 class=\"awr\">$node2 cssd log</h4>">>$htmlrpt
  ssh $node2 tail -200 $node2_cssdlog>>$smile
  local node2_cssd_log=`fold -w 180 $smile`
  cat<<EOF>>$htmlrpt
  <table border="1" class="tdiff">
  <tr><th class="awrbg" scope="col">$node2 cssd log</th></tr>
  <tr><td scope="row" class='awrc'><pre class="log">$node2_cssd_log</pre></td></tr>
  </table><br />
EOF
  cat<<EOF>>$htmlrpt
  <a class="awr" href="#clusterlog">Back to cluster logs</a><br />
  <a class="awr" href="#top">Back to Top</a><br /><br />
EOF
fi
}

##function to print cluster in html format
##example:html_cluster
function html_cluster(){
echo "checking cluster waiting..."
cat <<EOF>>$htmlrpt
<a class="awr" name="cluster"></a>
<h2 class="awr">cluster</h2>
<ul>
<li class="awr"><a class="awr" href="#crsnodes">cluster nodes</a></li>
<li class="awr"><a class="awr" href="#votedisk">votedisk</a></li>
<li class="awr"><a class="awr" href="#ocr">ocr</a></li>
<li class="awr"><a class="awr" href="#diagwait">css diagwait</a></li>
<li class="awr"><a class="awr" href="#crs_resource">cluster resource details</a></li>
<li class="awr"><a class="awr" href="#clusterlog">cluster logs</a></li>
<li class="awr"><a class="awr" href="#ocrback">ocr backup</a></li>
<li class="awr"><a class="awr" href="#diskgroup">asm diskgroup</a></li>
<li class="awr"><a class="awr" href="#asmalert">asm alert</a></li>
</ul>
EOF
html_crsnodes $crsnodes_switch
html_votedisk $votedisk_switch
html_ocr $ocr_switch
html_diagwait $diagwait_switch
html_crs_resource $crs_resource_switch
html_clusterlog $cluster_alert_switch $crsdlog_switch $cssdlog_switch
}

##function to end html
##example:html_end
function html_end(){
cat<<EOF>>$htmlrpt
</body>
</html>
EOF
}

##mian
init_script
env_user
env_ssh
echo "################################################################################"
echo "##             wisedu"
echo "################################################################################"
echo "1) check os"
echo "2) check cluster,listener"
echo "3) check database by options"
echo "4) check database by db including backup"
echo "5) check backup"
echo "6) check os,cluster,listener,db by options,backup(inspect)"
echo "7) html rport for all"
read -p "which? :" number
case $number in
  1)
  chk_os
  echo "report is $rpt"
  ;;2)
  env_expect
  chk_cluster
  chk_lsnr
  echo "report is $rpt"
  ;;3)
  chk_db_byopt
  echo "report is $rpt"
  ;;4)
  env_expect
  chk_db_bydb
  echo "report is $rpt"
  ;;5)
  env_expect
  chk_backup
  echo "report is $rpt"
  ;;6)
  env_expect
  chk_os
  chk_cluster
  chk_db_byopt
  chk_lsnr
  chk_backup
  echo "report is $rpt"
  ;;7)
  env_expect
  html_begin
  html_platform
  html_cluster
  html_end
  echo "report is $htmlrpt"
  ;;*)
  fuck
  ;;
esac
##end
