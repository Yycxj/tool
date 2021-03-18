# 基于全备增倍及binlog的数据恢复

## Mysql binlog 恢复工具
### mysqlbinlog
##### 简介
+ Mysql自带的 binlog 解析工具，本身不具有日志闪回功能
+ 可解析 binlog 日志为可阅读的格式或数据库可执行的格式
+ 可用来重放binlog或结合MyFlash进行闪回
+ 重放精度可到库级别，无法到表级别

##### 使用
+ 常用参数

```bash
# 解析为可阅读的格式
--base64-output=decode-rows -v
# 指定 start time
--start-datetime='2019-04-11 00:00:00'
# 指定 stop time
--stop-datetime='2019-04-11 15:00:00'
# 指定 start position
--start-position=50414870
# 指定 stop position
--stop-position=47881181
# 忽略之前事务id，让Mysql像执行新事务一样执行之前sql
--skip-gtids
# 指定数据库
--database=name
```
+ 示例

```bash
# 解析 binlog
$ mysqlbinlog  --base64-output=decode-rows -v  mysql-bin.000215
# 重放binlog
$ mysqlbinlog --database=name --skip-gtids --start-position=50414870 mysql-bin.000215 |mysql -uroot -paim.sh
```

### MyFlash

+ git: https://github.com/Meituan-Dianping/MyFlash

#### 简介：

+ MyFlash是由美团点评公司技术工程部开发维护的一个回滚DML操作的工具。该工具通过解析v4版本的binlog，完成回滚操作。相对已有的回滚工具，其增加了更多的过滤选项，让回滚更加容易。

+ 针对 binlog 二进制文件，可指定要闪回的**库** **表** **语句类型**，**生成的文件仍为 binlog 二进制文件**

+ 局限：

  + binlog格式必须为row,且binlog_row_image=full
  + 仅支持5.6与5.7
  + 只能回滚DML（增、删、改）
  + Ubuntu上安装依赖库没成功过

#### 使用
+ 参数

```bash
# 指定需要回滚的数据库名。多个数据库可以用“,”隔开。如果不指定该参数，相当于指定了所有数据库。
--databaseNames
# 指定需要回滚的表名。多个表可以用“,”隔开。如果不指定该参数，相当于指定了所有表。
--tableNames
# 指定回滚开始的位置。如不指定，从文件的开始处回滚。请指定正确的有效的位置，否则无法回滚
--start-position
# 指定回滚结束的位置。如不指定，回滚到文件结尾。请指定正确的有效的位置，否则无法回滚
--stop-position
# 指定回滚的开始时间。注意格式必须是 %Y-%m-%d %H:%M:%S。 如不指定，则不限定时间
--start-datetime
# 指定回滚的结束时间。注意格式必须是 %Y-%m-%d %H:%M:%S。 如不指定，则不限定时间
--stop-datetime
# 指定需要回滚的sql类型。目前支持的过滤类型是INSERT, UPDATE ,DELETE。多个类型可以用“,”隔开。
--sqlTypes
# 一旦指定该参数，对文件进行固定尺寸的分割（单位为M），过滤条件有效，但不进行回滚操作。该参数主要用来将大的binlog文件切割，防止单次应用的binlog尺寸过大，对线上造成压力
--maxSplitSize
# 指定需要回滚的binlog文件，目前只支持单个文件，后续会增加多个文件支持
--binlogFileNames
# 指定输出的binlog文件前缀，如不指定，则默认为binlog_output_base.flashback
--outBinlogFileNameBase
# 仅供开发者使用，默认级别为error级别。在生产环境中不要修改这个级别，否则输出过多
--logLevel
# 指定需要回滚的gtid,支持gtid的单个和范围两种形式。
--include-gtids
# 指定不需要回滚的gtid，用法同include-gtids
--exclude-gtids
```
+ 示例

```bash
$ /root/a/MyFlash-master/binary/flashback  --databaseNames=shouquanba  --tableNames=tbl_card --sqlTypes=DELETE  --binlogFileNames=mysql-bin.003718
```

### binlog2ql
+ git: https://github.com/danfengcao/binlog2sql

#### 简介
+ python 开发的工具，可解析和闪回binlog为**sql**
+ 支持环境：
  + Python 2.7, 3.4+
  + MySQL 5.6, 5.7
+ 限制：
  + mysql server必须开启，离线模式下不能解析
  + 参数 binlog_row_image 必须为FULL，暂不支持MINIMAL
  + 解析速度不如mysqlbinlog
+ 优点：
  + 自带flashback、no-primary-key解析模式，无需再装补丁
  + 解析为标准SQL，方便理解、筛选

#### 安装
```bash
# 下载工具
$ git clone https://github.com/danfengcao/binlog2sql.git && cd binlog2sql
# 安装依赖
$ pip3 install -r requirements.txt
# 如果 pip3 安装过程中出现**网络超时**的报错，可通过更改 pypi 源解决
$ vi ~/.pip/pip.conf
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.co
```
+ 创建用户

```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO  
```
#### 使用
+ 参数

```bash
# mysql连接配置
-h host; -P port; -u user; -p password

# 解析模式
# 持续解析binlog。可选。默认False，同步至执行命令时最新的binlog位置。
--stop-never
# 对INSERT语句去除主键。可选。默认False
-K, --no-primary-key
# 生成回滚SQL，可解析大文件，不受内存限制。可选。默认False。与stop-never或no-primary-key不能同时添加。
-B, --flashback
# 模式下，每打印一千行回滚SQL，加一句SLEEP多少秒，如不想加SLEEP，请设为0。可选。默认1.0。
--back-interval -B

# 解析范围控制
# 起始解析文件，只需文件名，无需全路径 。必须。
--start-file
# 起始解析位置。可选。默认为start-file的起始位置。
--start-position/--start-pos
# 终止解析文件。可选。默认为start-file同一个文件。若解析模式为stop-never，此选项失效。
--stop-file/--end-file
#终止解析位置。可选。默认为stop-file的最末位置；若解析模式为stop-never，此选项失效。
--stop-position/--end-pos
# 起始解析时间，格式'%Y-%m-%d %H:%M:%S'。可选。默认不过滤。
--start-datetime
# 终止解析时间，格式'%Y-%m-%d %H:%M:%S'。可选。默认不过滤。
--stop-datetime

# 对象过滤
# 只解析目标db的sql，多个库用空格隔开，如-d db1 db2。可选。默认为空。
-d, --databases
# 只解析目标table的sql，多张表用空格隔开，如-t tbl1 tbl2。可选。默认为空。
-t, --tables
# 只解析dml，忽略ddl。可选。默认False。
--only-dml
# 只解析指定类型，支持INSERT, UPDATE, DELETE。多个类型用空格隔开，如--sql-type INSERT DELETE。可选。默认为增删改都解析。用了此参数但没填任何类型，则三者都不解析。
--sql-type
```
+ 示例

```bash
$ python3 binlog2sql/binlog2sql.py --flashback  -h172.16.0.240 -P6667 -utestbinloguser -p 'Y0wvAY6UycWYCW47' -d shouquanba -t tbl_card --sql-type DELETE --start-file='mysql-bin.003718' > rollback18.sql
```
#### 补充
+ 闪回时抛出utf8编码问题
  +	编辑 *binlog2sql_util.py* ，将 *decode('utf-8')* 变更为 *decode('utf-8',"ignore")* ，约2处


## 基于全备增倍的数据恢复

### 基于全备恢复
```bash
$ xtrabackup --prepare  --user-memory=1G --target-dir=2021-03-13_01-15-01_full
```
### 基于增倍恢复
```bash
$ xtrabackup --prepare --apply-log-only  --user-memory=1G --target-dir=2021-03-13_01-15-01_full
$ xtrabackup --prepare --user-memory=1G --target-dir=2021-03-13_01-15-01_full  --incremental-dir=2021-03-14_01-15-01_inc
```

### 恢复并启动数据库
```bash
$ cp -rp 2021-03-13_01-15-01_full/* /home/mysql/mysql3307/data/data_3307
$ chown mysql.mysql -R /home/mysql/mysql3307/data/data_3307
```
### 重放 binlog 至具体位置
+ + plan1

```bash
$ mysqlbinlog --database=vegas --skip-gtids --stop-position=39758276 --start-position=4900227 mysql-bin.000254 |mysql -uyanx -h127.0.0.1 -P3307 -p
```
+ plan2

```bash
$ mysqlbinlog --database=vegas --skip-gtids --stop-position=39758276 --start-position=4900227 mysql-bin.000254 > 254.out
$ cat 254.out |mysql -uyanx -h127.0.0.1 -P3307 -p
```


### xtrabackup参数
```bash
# 处理未提交日志等信息，使数据处于一致状态
--prepare
# xtrabackup做crash recovery分配的内存大小
--user-memory
# 只会重做已提交但是未应用的事务，而不会回滚未提交的事务
# 在准备最后一份备份之前的备份时，都不用进行回滚操作，只在最后一次备份中进行回滚操作
--apply-log-only
```