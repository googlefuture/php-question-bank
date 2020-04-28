# 基础知识汇总

## Redis支持的数据类型？

String字符串、Hash（哈希）、List（列表）、Set（集合）、zset(sorted set：有序集合)


## Redis连接时的connect与pconnect的区别

connect：脚本结束之后连接就释放了。

pconnect：脚本结束之后连接不释放，连接保持在php-fpm进程中。每个php-fpm进程占用一哥连接，当php-fpm进程结束时会释放掉
所以使用pconnect代替connect，可以减少频繁建立redis连接的消耗。


## 使用过Redis分布式锁么，它是怎么实现的？
先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。

