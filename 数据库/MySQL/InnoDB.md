ibdata1文件是InnoDB存储引擎的共享表空间文件，该文件中主要存储着下面这些数据：

```
data dictionary
double write buffer
insert buffer/change buffer
rollback segments
undo space
Foreign key constraint system tables
```

ib_logfile1是INNODB的REDO、UNDO日志，并不是备份用的日志。
MYSQL可以通过BINLOG来恢复，但这个ib_logfile没什么恢复的作用，它主要是在事务中起一个前滚或后滚的作用。
ib_logfiles的作用，主要是在系统崩溃重启时，作事务重做的。而在系统正常时，每次checkpoint时间点，会将之前写入的事务应用到数据文件中。因此有一个问题：系统重启之后，怎么知道checkpoint做到哪儿了?