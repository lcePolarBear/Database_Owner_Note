## RMAN 的独特之处
- 支持增量备份
- 自动管理备份文件（因为恢复目录的使用，RMAN 在其中保存了备份和恢复脚本）
- 自动化备份与恢复
- 不产生重做信息

## RMAN 系统构架
- RMAN 信息库
    - 如备份的数据文件及副本的目录、归档重做日志备份文件和副本、表空间和数据文件、以及备份或恢复的脚本和 RMAN 的配置信息
    - 默认使用数据库服务器的控制文件记录这些信息，可以转储控制文件来发现这些信息
        ```
        SQL> alter database backup control file to trace;
        ```
- 恢复目录
    - 记录 RMAN 信息库的信息，但是恢复目录需要事先配置
    - 信息库可以配置在控制文件中也可以存储在恢复目录中，建议在恢复目录中

## 快闪恢复区（ flash recovery area ）
> 存储和备份数据文件已经相关信息的存储区
- 快闪恢复区保存了每个数据文件的备份、增量备份、控制文件备份以及归档 redo log 备份
- 快闪恢复区实现了备份文件的自动管理，使得备份与恢复数据库更加简单，并且可以集中管理磁盘空间
### 快闪区恢复大小
- 使用 db_recovery_file_dest 参数可以查看恢复区在磁盘上的路径和空间大小
- 参数含义
    - `db_recovery_file_dest`	设置快闪区在磁盘上的位置
    - `db_recovery_file_dest_size`	设置快闪恢复区的最大容量
- 修改快闪恢复区的空间大小
    ```
    alter system set db_recovery_file_dest_size = 2048
    ```
    > 可以通过设置 db_recovery_file_dest_size 为空格来取消恢复区的存储目录, `v$recovery_file_dest` 视图可以查看存储目录的位置及空间使用信息
- `v$flash_recovery_area_usage` 参数可以查看闪回恢复区的空间使用

## RMAN 相关概念
- 备份片
    >备份片是 RMAN 格式的操作系统文件，包含控制文件，数据文件或归档日志文件
- 通道
    >通过与数据库服务器的会话建立连接，通道代表这个链接，指定备份或备份集所在的设备
- 备份集
    >由一个或多个 RMAN 备份片组成的逻辑数据集合。在进行 RMAN 备份时将产生备份文件的备份集，备份集只有 RMAN 才可以识别。一般一个通道生成一个备份集。控制文件自动备份会生成一个单独的备份集，因为他的备份集块大小为 OS 块大小，数据文件的备份集块大小为 DB 块大小
- 映像复制
    >数据库文件的操作系统文件的一个备份，一个数据文件生成一个映像副本
- 使用 RMAN 将默认创建备份集，也可以设置备份类型为 copy 使得 RMAN 的任何备份不产生备份集而产生映像复制

- 使用 backup as copy 指令实现映像复制
    - 映像复制整个数据库： `backup as copy database`
    - 映像复制单个表空间： `backup as copy tablespace users`
    - 映像复制整个数据库的单个数据文件： `backup as copy datafile 3`
        > 数据文件 ID 可以从视图 dba_data_files 中查询
## RMAN 的配置参数
```
RMAN> show all;
```
- `configure retention policy to redundancy 1`
    >说明保留备份的副本数量，参数 1 说明只保留一个该数据文件的备份并保留最新的备份副本
- `configure default device type disk`
    >说明设备的类型
- `configure backup optimization off | on`
    >是否打开备份优化
- `configure controlfile autobackup off | on`
    >设置自动备份控制文件
- `configure device type disk parallelism 1 backup type to backupset`
    >设置备份恢复的并行度
## RMAN 备份控制文件
- RMAN 可以单独备份控制文件，控制文件会备份到快闪恢复区或者 format 参数指定的路径下
    ```
    RMAN> backup current controlfile format 'f:\pump\backup\backup_ctl_%u.dbf';
    ```
    >使用快闪恢复区可以让 Oracle 自动管理文件
- 配置控制文件备份的磁盘类型和备份路径
    ```
    RMAN> configure controlfile autobackup format for device type disk to '/u01/backup/%F';
    ```
- 设置控制文件自动备份
    ```
    RMAN> configure controlfile autobackup on;
    ```
## RMAN 实现全库脱机备份
使用 RMAN 登陆到数据库服务器，关闭数据库然后启动数据库到 mount 状态，再执行 backup database 指令备份整个数据库。当备份整个数据库时。 RMAN 将自动备份执行文件和参数文件，这取决于 `configure controlfile autobackup on` 设置为 on 使得自动备份控制文件和参数文件。最后打开数据库即可完成整个数据库的脱机备份

## RMAN 联机备份
- 联机备份前的准备工作
    - 设置快闪恢复区（并且空间要足够）
    - 数据库处于归档模式
- 联机备份整个数据库
    ```
    RMAN> backup as compressed backupset database plus archivelog delete all input;
    ```
- 联机备份一个表空间(压缩备份 users 表空间)
    ```
    RMAN> backup as compressed backupset tablespace users;
    ```
- 联机备份一个数据文件(可以通过 format 来指定备份路径)
    ```
    RMAN> backup as backupset datafile 4 format '/u01/backup/datafile_';
    ```
- 备份坏块的处理方式
## RMAN 的增量备份
- 两个级别的增量备份
    - 0 级别备份 = 全库备份
        ```
        RMAN> backup incremental level 0 database;
        ```
    - 1 级别备份 = 差异备份（只备份上次增量备份以来变化的数据）
        ```
        RMAN> backup incremental level 1 database;
        ```
## 快速增量备份
>将数据库中发生变化的数据块位置记录在一个更改跟踪文件中，减少在备份时扫描全库的时间。跟踪特性由 CTWR 进程支持
- 启动跟踪特性
    ```
    alter database enable block change tracking using file 'e:/oracle/product/10.2.0/oradata/chtrack.log';
    ```
- 查询跟踪特性可以用 `v$block_change_tracking` 视图
- 如果需要更改跟踪文件的存储位置
    - 将数据库置于 mount 状态
    - ```
        alter databse rename file 'xxx .log' to 'xxx .log';
        ```
- 禁用跟踪特性
    ```
    alter database disable block change tracking;
    ```
## 创建和维护恢复目录
> 恢复目录实际上是一个数据库，可以是另外的数据库也可以是目标数据库自身
- 在恢复数据库创建恢复目录表空间和对应的用户
    ```
    create tablespace rmantbs datafile '.../rmantbs.dbf' size 100m;

    grant recovery_catalog_owner,connect,resource to rman identified by rman;
    ```
- 连接并创建恢复目录
    ```
    $ rman catalog rman/rman@orcl

    RMAN> create catalog tablespace rmantbs;
    ```
    - 删除恢复目录： `drop catalog` 
    >此时恢复目录保存在 rmantbs 表空间中，该表空间就是在创建 rman 用户时指定的默认表空间

- 注册目标数据库
    >目的是使得恢复目录知道目标数据库的信息，并自动与目标数据库通信获得相关元数据
    ```
    $ rman target / catalog rman/rman

    RMAN> register database;
    ```
- 同步恢复目录
    ```
    RMAN> resync catalog;
    ```
---
在联机备份整个数据库时，因为之前用 rm 删掉了归档日志文件导致控制文件不一致造成无法备份的情况

解决方案：使控制文件中的归档日志信息和实际物理文件信息保持一致
```
RMAN> crosscheck archivelog all;    //检查控制文件和实际物理文件的差别

RMAN> delete expired archivelog all;    //同步控制文件的信息和实际物理文件的信息

RMAN> backup database plus archivelog;  //将数据库连同归档日志全部备份
```

