创建一个Maven项目，在pom.xml中添加下面的内容
<dependencies>
    <dependency>
      <groupId>org.apache.flume</groupId>
      <artifactId>flume-ng-configuration</artifactId>
      <version>1.5.2</version>
    </dependency>
    <dependency>
      <groupId>org.apache.flume</groupId>
      <artifactId>flume-ng-core</artifactId>
      <version>1.5.2</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.26</version>
    </dependency>
  </dependencies>

创建包com.polaris.flume
package com.polaris.flume;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;
import org.apache.flume.Channel;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.Transaction;
import org.apache.flume.conf.Configurable;
import org.apache.flume.sink.AbstractSink;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.google.common.base.Preconditions;
import com.google.common.base.Throwables;
import com.google.common.collect.Lists;

public class MysqlSink extends AbstractSink implements Configurable{
private Logger logger = LoggerFactory.getLogger(MysqlSink.class);
private String hostname;   //主机名
private String port;     //端口
private String databaseName;   //数据库名字
private String tableName;     //表名
private String user;      //用户名
private String password;   //密码
private int batchSize; //每次数据库交互， 处理多少条数据
private Connection conn;
private PreparedStatement preparedStatement;

public MysqlSink(){
logger.info("MysqlSink start...");
}

public void configure(Context context) {
hostname = context.getString("hostname");  
        Preconditions.checkNotNull(hostname, "hostname must be set!!");  
        
        port = context.getString("port");  
        Preconditions.checkNotNull(port, "port must be set!!"); 
        
        databaseName = context.getString("databaseName");  
        Preconditions.checkNotNull(databaseName, "databaseName must be set!!"); 
        
        tableName = context.getString("tableName");  
        Preconditions.checkNotNull(tableName, "tableName must be set!!");
        
        user = context.getString("user");  
        Preconditions.checkNotNull(user, "user must be set!!"); 
        
        password = context.getString("password");  
        Preconditions.checkNotNull(password, "password must be set!!");
        
        batchSize = context.getInteger("batchSize", 100);  
        Preconditions.checkNotNull(batchSize > 0, "batchSize must be a positive number!!");  
    }  

@Override
public void start() {
super.start();
try {
Class.forName("com.mysql.jdbc.Driver");
} catch (ClassNotFoundException e) {
e.printStackTrace();
}

String url = "jdbc:mysql://" + hostname + ":" + port + "/" + databaseName + "?Unicode=true&characterEncoding=UTF-8";
try {
conn = DriverManager.getConnection(url, user, password);
conn.setAutoCommit(false);

preparedStatement = conn.prepareStatement("insert into " + tableName + " (content) values (?)");
} catch (SQLException e) {
e.printStackTrace();
System.exit(1);
}
}

public void stop() {  
        super.stop();  
        if (preparedStatement != null) {  
            try {  
                preparedStatement.close();  
            } catch (SQLException e) {  
                e.printStackTrace();  
            }  
        }  
   
        if (conn != null) {  
            try {  
                conn.close();  
            } catch (SQLException e) {  
                e.printStackTrace();  
            }  
        }  
    }  

    public Status process() throws EventDeliveryException {
Status result = Status.READY;
Channel channel = getChannel();
Transaction transaction = channel.getTransaction();
Event event;  
        String content;  
        
List<String> actions = Lists.newArrayList();
transaction.begin();

try {  
            for (int i = 0; i < batchSize; i++) {  
                event = channel.take();  
                if (event != null) {  
                    content = new String(event.getBody());  
                    actions.add(content);  
                } else {  
                    result = Status.BACKOFF;  
                    break;  
                }  
            }  
   
            if (actions.size() > 0) {  
                preparedStatement.clearBatch();  
                for (String temp : actions) {  
                    preparedStatement.setString(1, temp);  
                    preparedStatement.addBatch();  
                }  
                preparedStatement.executeBatch();  
   
                conn.commit();  
            }  
            transaction.commit();  
        } catch (Throwable e) {  
            try {  
                transaction.rollback();  
            } catch (Exception e2) {  
                logger.error("Exception in rollback. Rollback might not have been" + "successful.", e2);  
            }  
            logger.error("Failed to commit transaction." + "Transaction rolled back.", e);  
            Throwables.propagate(e);  
        } finally {  
            transaction.close();  
        }  
        return result;  
   }
}

将代码打成jar包后，上传到flume安装目录下的lib文件夹中，同时需要上传MySQL的驱动jar包
给flume添加mysqlSink.conf文件:
[sparkadmin@hadoop4 conf]$ vim mysqlSink.conf 
agent.sources = r1
agent.sinks = s1
agent.channels = c1
 
#source
agent.sources.r1.type = exec
agent.sources.r1.command = tail -F /home/sparkadmin/cloud/data/log
agent.sources.r1.channels = c1

#channel
agent.channels.c1.type = memory
agent.channels.c1.capacity = 1000
agent.channels.c1.transactionCapactiy = 100

#sink
agent.sinks.s1.type = com.polaris.flume.MysqlSink
agent.sinks.s1.hostname = hadoop4
agent.sinks.s1.port = 3306
agent.sinks.s1.databaseName = xiaoluo
agent.sinks.s1.tableName = test1
agent.sinks.s1.user = root
agent.sinks.s1.password = root
agent.sinks.s1.channel = c1

启动flume命令：
[sparkadmin@hadoop4 flume]$ bin/flume-ng agent -c conf/ -f conf/mysqlSink.conf -n agent -Dflume.root.logger=INFO,console
在Mysql中创建一个表：
mysql> create table test1( id int(11) NOT NULL AUTO_INCREMENT, content varchar(255) NOT NULL, PRIMARY KEY (id)) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

生成数据到目标日志文件中：
[sparkadmin@hadoop4 ~]$ for i in {1..1000};do echo "exec tail$i" >> /home/sparkadmin/cloud/data/log;done;
完成后，数据和预想中的一样，写入了数据库中。
mysql> select * from test1;
+------+---------------+
| id   | content       |
+------+---------------+
|   4 | exec tail1    |
|   5 | exec tail2    |
|   6 | exec tail3    |
.................
