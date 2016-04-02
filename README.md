背景：
   打算将一个从库从5.5.39 升级到5.6.27，但在同步过程中，slave同步一直很慢，因此进行排查。
 
原同步关系：
     5.1.73 master -> 5.5.39 slave
新同步关系：
    5.1.73 master -> 5.6.27 slave

从库信息查看：
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.003106
          Read_Master_Log_Pos: 201031632
               Relay_Log_File: mysql-relay-bin.000059
                Relay_Log_Pos: 200662695
        Relay_Master_Log_File: binlog.003106
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 200662539
              Relay_Log_Space: 201031997
首先对主要参数解释：
Master_Log_File
I/O线程当前正在读取的主服务器二进制日志文件的名称 
Read_Master_Log_Pos
在当前的主服务器二进制日志中，I/O线程已经读取的位置
Relay_Master_Log_File
由SQL线程执行的包含多数近期事件的主服务器二进制日志文件的名称 
Exec_Master_Log_Pos  
来自主服务器的二进制日志的由SQL线程执行的上一个时间的位置（Relay_Master_Log_File）。在主服务器的二进制日志中的(Relay_Master_Log_File, Exec_Master_Log_Pos)对应于在中继日志中的(Relay_Log_File,
Relay_Log_Pos)。  
Relay_Log_Space
所有原有的中继日志结合起来的总大小
Relay_Log_File
SQL线程当前正在读取和执行的中继日志文件的名称。
Relay_Log_Pos
在当前的中继日志中，SQL线程已读取和执行的位置  
分析：
1、首先看 Relay_Master_Log_File 和 Master_Log_File 是否有差异；
    Master_Log_File: binlog.003106
    Relay_Master_Log_File: binlog.003106
无差异，在继续看其他方面

2、看Exec_Master_Log_Pos和 Read_Master_Log_Pos 的差异，对比SQL线程比IO线程慢了多少个binlog事件；
        Exec_Master_Log_Pos: 200662539
        Read_Master_Log_Pos: 201031632
        明显sql线程比io线程慢了369093个事件。可以看出sql线程执行较慢。
     初步分析原因：
        1、sql 执行大事务（当时还原两台机器，另一台5.1.73 没问题）
        2、io 速度跟不上（同一台机器，io肯定没问题）
        3、配置参数（可能性较大）   
问题排查：
看错误日志：
2016-04-02 09:14:26 23761 [Warning] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
2016-04-02 09:14:26 23761 [Warning] Slave I/O: Notifying master by SET @master_binlog_checksum= @@global.binlog_checksum failed with error: Unknown system variable 'binlog_checksum', Error_code: 1193
2016-04-02 09:14:26 23761 [Warning] Slave I/O: Unknowbinlog_checksumn system variable 'SERVER_UUID' on master. A probable cause is that the variable is not supported on the master (version: 5.1.73-log), even though it is on the slave (version: 5.6.27-76.0-log), Error_code: 1193
解决方法：

5.6.x 后用server_uuid 代替了server_id ,并且binlog_checksum 默认支持了crc32校验，但5.1.73 不支持。
Why did this happen? Starting in MySQL 5.6.6, the new binlog_checksum option defaults to CRC32. Since that option did not exist in MySQL 5.5, the replica can’t handle the checksums coming from the master. Therefore I recommend settingbinlog_checksum=NONE in my.cnf as part of the upgrade process for a master-master setup to avoid this error.
My fix was to run this on the passive master:
1
set global binlog_checksum='NONE';
Then I added this to my.cnf so it would survive a restart:
1
binlog_checksum=NONE
参考： http://mechanics.flite.com/blog/2014/04/29/disabling-binlog-checksum-for-mysql-5-dot-5-slash-5-dot-6-master-master-replication/

另外从5.5 到5.6复制也会出现问题，已经确认为bug
如：
After upgrading a slave to 5.6.9-rc from 5.523, the master still runs 5.5 I see:
2013-01-24 13:18:40 28611 [Warning] Slave I/O: Unknown system variable 'SERVER_UUID' on master, maybe it is a *VERY OLD MASTER*. Error_code: 1193
Text is copied exactly as is from the logs.
This is rather alarming and I don't think it should be so dramatic as it's likely to worry people. When doing an upgrade initially you expect the master to be on a version behind the newly upgraded slave.
这些bug在高版本已经修复
官方恢复：
Fixed in 5.6+. Documented in the 5.6.11 and 5.7.1 changelogs as follows:
        When replicating to a MySQL 5.6 master to an older slave, Error
        1193 (ER_UNKNOWN_SYSTEM_VARIABLE) was logged with a message such
        as -Unknown system variable 'SERVER_UUID' on master, maybe it is
        a *VERY OLD MASTER*.- This message has been improved to include
        more information, similat to this one: -Unknown system variable
        'SERVER_UUID' on master. A probable cause is that the variable
        is not supported on the master (version: 5.5.31), even though it
        is on the slave (version: 5.6.11)-
参考： https://bugs.mysql.com/bug.php?id=68164

先把有错误的解决掉，在待观察，希望对大家有此情况有所帮助。
