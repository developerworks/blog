1. Persistent runtime configuration changes. Love it. From now on we’ll be able to use SET PERSIST innodb_buffer_pool_size = X; instead of SET GLOBAL innodb_buffer_pool_size = X; for the runtime changes to persist during a restart. It may not make much sense if you’re using a modern database that doesn’t even have a configuration file, but for us who lived with MySQL for over 20 years, this is huge!

## 持久化运行时配置

How does it work? In a nutshell, these changes are saved in mysqld-auto.cnf file in MySQL data directory.

2. MySQL privilege tables are now InnoDB. I think this was the last thing holding MyISAM as a mandatory storage engine for MySQL. Buckle your seatbelt MyISAM, ’cause Kansas is going bye-bye!

## MySQL 特权表已迁移到InnoDB

3. Roles. This is basically an alias for a collection of privileges, so you don’t have to remember whether you should GRANT INSERT, UPDATE, DELETE, SELECT for these analytics clients, or will SELECT suffice. Simply GRANT ‘analytics’ role and you’re good to go. I’m not really dying to get this feature ASAP, but my eyebrows did lift up when I saw this.

## 角色

4. Global Data Dictionary (so long .frm, .TRG and .TRN files!). Global data dictionary comes with a number of nice benefits with it (such as Dictionary object cache), although it’s also one of the reasons upgrade to 8.0 will be backwards incompatible. BTW, InnoDB will keep maintaining its own data dictionary, but I’m guessing the intention is to get rid of it eventually.

## 优化器提示

5. Optimizer hints. This is a nice alternative to the optimizer_switch session variable – I’ll definitely be using optimizer hints instead. Besides it being more convenient to use, added benefit is that you can specify different switches per table.

## 不可见索引

6. Invisible indexes. That’s right. Indexes can now be made invisible. Actually, it’s really neat feature – you can basically disable an index before you remove it to check whether it will do any harm to removee it. When you make an index invisible, it’s still maintened normally, but optimizer is not allowed to see it.

## 死锁检测

7. Deadlock detection can now be disabled with innodb_deadlock_detect variable. Guessing this was inspired by the following WebScaleSQL patch. I could be wrong though. In any case, if you have a highly concurrent workload, try it out. What happens with deadlocks when deadlock detection is disabled? Such locks will have to wait for innodb_lock_wait_timeout to occur.

## 删除了缓冲池互斥锁

8. Innodb buffer pool mutex removed. Okay, this one will probably make you roll your eyes rather than raise your eyebrows, because, well, Percona Server had it since like version 5.0. In any case, having it in official MySQL release and with appropriate acknowledgements (Yasufumi, Laurynas, wink wink) is pretty amazing.

9. Auto-increment counter value will now persist across server restarts! The value will be written to the redo log each time the value changes, and saved to an engine-private system table on each checkpoint. More on it here.

## 增加 UUID_TO_BIN() 和 BIN_TO_UUID() 函数
10. UUID_TO_BIN() and BIN_TO_UUID() functions. What for? Well, because “69de6646-7904-11e6-9ff9-99003302702e” can then be stored within 16 bytes (VARBINARY(16)) rather than 36 (CHAR(36)). And it’s not a small improvement. 16 bytes is just twice as big as bigint, whereas using CHAR(36) for UUIDs were rendering them virtually useless.

11. An insane amount of bugs fixed. For a full list, check this out.

I’m sure a long and winding road still leads to RC and GA. Many more bugs are yet to be fixed. Maybe even additional features to be added. But it is definitely a good start. I’ll definitely be playing around with it soon and I will let you know how things look.


## 参考资料

http://www.speedemy.com/new-in-mysql-8-0-dr/
https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html#mysqld-8-0-0-data-dictionary
https://yq.aliyun.com/articles/60656
