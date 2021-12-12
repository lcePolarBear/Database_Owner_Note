## 使用 RMAN 非归档模式下的完全恢复
> 环境要求：数据库处于非归档模式，做了数据库脱机备份； RMAN 参数使用快闪恢复区存储文件，配置控制文件自动备份
### 只有数据文件（部分）丢失的恢复
- 在 mount 状态下恢复数据文件
    ```
    RMAN> restore datafile 4;
    ```
- 恢复后要使用 recover 命令将 redo log 文件恢复到数据文件
    ```
    RMAN> recover datafile 4;
    ```
- 如果有幸 redo log 文件没有被覆盖。到此已经恢复成功，如果不幸重做日志组被覆盖了,就要进行不完全恢复
    ```
    RMAN> recover datafile 4 until calcel;
    SQL> alter database open resetlogs;
    ```
### online redo log 文件和数据文件损坏的恢复
- 恢复数据文件和数据库
    ```
    RMAN> restore database;
    ```
    ```
    recover database until cancel;
    ```
- 在开启数据库时重建 redo log 文件
    ```
    SQL> alter database open resetlogs;
    ```
    > 在这种情况下自备份以来所有的数据变化都丢失了，所以 DBA 应该注意重做日志组要有多个，每组要有不同路径的多个日志文件以避免 redo log 文件的丢失
### 控制文件、数据文件、以及 redo log 文件丢失后的恢复步骤
- 首先数据库在 mount 状态下做一个全库备份
    ```
    RMAN> backup as compressed backupset database;
    ```
- 我们可以把控制文件（ control01.ctl ）、 redo log 文件（ redo01.log ）、数据文件（ system01.dbf ）删除掉。同时我们可以创建一个表，这个表是在备份之后创建的，以此来验证只能恢复在备份之前的数据
- 控制文件的恢复
    ```
    RMAN> restore controlfile from 'xxx.bkp';
    ```
    > 控制文件在备份时会告知备份路径
    - 此时数据库可以启动到 mount 状态
- 数据文件的恢复
    - 在 mount 状态下进行
    ```
    RMAN> restore database;
    ```
    > RMAN 从快闪区读取备份集，通过记录的元数据将数据文件恢复到控制文件中记录的位置
- redo log 文件的恢复
    > 因为失去所有的重做日志，所以无法使用 redo 数据了。下面恢复数据库并使用 noredo 选项
    ```
    recover database noredo;
    ```
- 因为没有 redo log 文件，我们需要用 resetlogs 参数打开数据库会创建新的重做日志文件
    ```
    SQL> alter database open resetlogs;
    ```
## RMAN 归档模式下的完全恢复
>环境要求：RMAN 备份以来数据库运行在归档下；归档文件和重做日志文件无损坏
### 非系统表空间损坏
- 将受损的数据文件删除
- 将数据库 置于 mount 状态
- 将数据文件离线并打开数据库
    ```
    SQL> alter databse datafile 4 offline;

    SQL> alter database open;
    ```
- 使用 RMAN 还原数据文件
    ```
    RMAN> restore datafile 4;
    ```
- 使用归档日志或者 redo log 文件恢复数据文件
    ```
    RMAN> recover datafile 4;
    ```
- 将数据文件联机
    ```
    alter database datafile 4 online;
    ```
### 系统表空间损坏的恢复
- 恢复过程类似，但是系统表空间的缺失只能让数据库在 mount 状态下去恢复
### RMAN 实现数据块恢复
> 如果某个数据文件的的数据块损坏，通过数据文件的完整备份可以恢复数据块
- 准备工作：需要 RMAN 备份整个数据库（这是一个全备份，包括归档日志）
    ```
    RMAN> backup database plus archivelog;
    ```
- 当数据文件中的数据块发生损坏时，打开数据库会报错数据文件需要介质恢复
- 对数据文件进行有效检验
    ```
    RMAN> backup validate datafile 6;
    ```
- 通过 `v$database_block_corruption` 视图查看并记录数据文件中损坏的数据块
- 使用 RMAN 完成数据块恢复
    ```
    blockrecover datafile 6 block 118 from backupset;
    ```
- 此时块已经恢复，但是数据文件会验证失败。进行数据文件恢复
    ```
    SQL> recover datafile 6;
    ```
## RMAN 的备份维护指令
### 验证备份文件的可用性
- 查看备份集的汇总信息，获取需要验证的备份集 key
    ```
    RMAN> list backup summary;
    ```
- 验证备份集的可用性
    ```
    validate backupset 5;
    ```
### 验证某个表空间或数据文件是否在备份集中
- 验证表空间备份信息是否在备份集
    ```
    RMAN> restore tablespace users validate;
    ```
- 验证数据文件是否在备份集中
    ```
    RMAN> restore datafile '/u01/ ... .dbf' validate;
    ```
- 输出中的 “验证完成” 说明表空间或者说明该数据文件在备份集中
### 查看恢复所需的备份文件是否存在
- 查看恢复整个数据库所需的备份文件是否存在
    ```
    RMAN> restore database preview;
    ```
- 验证恢复表空间所需的备份文件是否存在
    ```
    RMAN> restore tablespace sysaux preview;
    ```
- 验证恢复表空间所需的备份文件是否存在
    ```
    RMAN> restore datafile 5 preview;
    ```
- 最后输出 Finished restore 表示查询的信息存在备份集中
### 查看当前的备份集信息（某个表空间所在的备份集，某个数据文件所在的备份集等）
- 查看所有备份集信息与查看指定的备份集信息
    ```
    RMAN> list backupset;
    ```
    ```
    RMAN> list backupset 5;
    ```
- 寻找指定表空间或数据文件备份所在的备份集
    ```
    RMAN> list backup of tablespace users;
    ```
    ```
    RMAN> list backup of database 1;
    ```
- 查看归档日志文件以及控制文件、参数文件的备份信息
    ```
    RMAN> list backup of archivelog all;
    ```
    ```
    RMAN> list backup of controlfile;
    ```
    ```
    RMAN> list backup of spfile;
    ```
### RMAN 的分析功能
- 显示数据结构
    ```
    RMAN> report schema;
    ```
- 列出需要备份的数据文件
    ```
    RMAN> report need backup;
    ```