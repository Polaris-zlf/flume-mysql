1、创建一个Maven项目
2、将代码打成jar包后，上传到flume安装目录下的lib文件夹中，同时需要上传MySQL的驱动jar包
3、给flume添加mysqlSink.conf文件:
4、启动flume命令：
   [sparkadmin@hadoop4 flume]$ bin/flume-ng agent -c conf/ -f conf/mysqlSink.conf -n agent -Dflume.root.logger=INFO,console
5、在Mysql中创建一个表：
   mysql> create table test1( id int(11) NOT NULL AUTO_INCREMENT, content varchar(255) NOT NULL, PRIMARY KEY (id)) ENGINE=InnoDB       AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
6、生成数据到目标日志文件中：
  [sparkadmin@hadoop4 ~]$ for i in {1..1000};do echo "exec tail$i" >> /home/sparkadmin/cloud/data/log;done;
7、完成后，数据和预想中的一样，写入了数据库中。
mysql> select * from test1;
+------+---------------+
| id   | content       |
+------+---------------+
|   4 | exec tail1    |
|   5 | exec tail2    |
|   6 | exec tail3    |
.................

完整文章可以参考：http://blog.csdn.net/u012689336/article/details/53079941
