## Redo Log 的概念
- 引入重做日志的原因：数据恢复
- 用户变更会记录到内存（数据库高速缓冲），到缓冲区满或者达到一定条件时才会触发 DBWR 写入数据文件
- 为了避免宕机时内存中的数据丢失，变更的数据先通过 LGWR 写入 redo log 文件，再写入内存
- 重做文件结构由至少两个日志组（每个日志组至少有一个成员）组成
- 不启动归档日志的话（在非归档模式下），在发生日志切换时会覆盖切入方的日志， DBWR 会在发生日志切换时将切出方所有的数据写入数据文件
- 在归档模式下 oracle 会关闭满的重做日志，ARCH 进程把旧的重做日志文件中的数据移动到归档重做日志文件中
## Redo Log 信息查询
- v$log 视图记录日志文件和日志组的关联信息
    ```
    SQL> select * from v$log;
    ```
    - status 字段表示当前的日志组所处的状态
        - INACTIVE 表示当前未使用
        - CURRENT 表示当前正在使用的
        - ACTIVE 表示正在进行归档
- v$logfile 视图记录日志文件的物理路径
    ```
    SQL> select * from v$logfile;
    ```
    - 注意 STATUS 字段
        - stale：文件内容不完整
        - 空白：正在使用
        - invalid：不能被访问
        - deleted：不再使用

## 归档模式（归档重做日志就是联机重做日志的备份）
- 归档模式判断：`archive log list;`
- 归档模式的设置：在 mount 模式下 `alter database archivelog;`
- 归档路径在 `db_recovery_file_dest` 参数指定的路径
    - 修改 `db_recovery_file_dest` 的大小
        ```
        SQL> ALTER SYSTEM SET db_recovery_file_dest_size=3g SCOPE=BOTH;
        ```
    - 修改 `db_recovery_file_dest` 的路径
        ```
        SQL> ALTER SYSTEM SET db_recovery_file_dest='/u01/oracle/fast' SCOPE=BOTH;
        ```

## 重做日志组
- 添加一个重做日志组，日志组号为4（不指定的话默认自动增长）,添加两个日志组的成员,指定大小 11M
    ```
    alter database add logfile group 4
    ('d:\temp\redo04a.log','d:\temp\redo04b.log')
    size 11M;
    ```
- 删除重做日志组
    > 如果日志文件有问题了，就要删掉重建（或者要重新设置联机重做日志的大小）
    ```
    alter database drop logfile group 4;
    ```
- 删除日志文件
    ```
    alter database  drop logfile member 'd:\temp\redo04a.log';
    ```
	- 手动删除操作系统文件
- 向日志组添加成员
    ```
	alter database add logfile member
	'd:\temp\redo01a.log' to group 1,
	'd:\temp\redo01b.log' to group 2;
    ```
- 手动日志切换
    ```
    alter system switch logfile;
    ```
- 在归档模式下，如果 redo 日志文件损坏会造成无法归档，就需要清除联机 redo 日志来重新初始化
    ```
    alter database clear logfile group 4;
    ```

## 检查点事件
- LGWR 将 redo log buffer 中的数据写入 redo 日志文件
- DBWR 将数据库高速缓存中已提交的数据写入数据文件
- 检查点事件越频繁，数据库需要的回复的重做数据就越少
- 强制启动检查点时间
    ```
    alter database checkpoint;
    ```