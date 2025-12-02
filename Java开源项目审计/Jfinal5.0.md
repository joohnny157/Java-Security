该项目版本存在

sql注入、任意文件写入、任意文件删除、任意文件读取

JFinalCMS Custom Data Page SQL注入 CVE-2024-2568 

接口存在于 "/admin/div_data"  

<img width="1110" height="735" alt="Image" src="https://github.com/user-attachments/assets/36dc3013-e92a-4213-b002-8deb9e6792c7" />


在代码的131行 采用了拼接的方式来执行SQL语句
这个方法用于删除后台自定义表

<img width="851" height="680" alt="Image" src="https://github.com/user-attachments/assets/06fb58db-10a2-4ca1-a1cd-f1e5feac3349" />

使用了工具类DB中的update方法 执行sql语句
delete from + 'div.getTableName()' + where id in +StringUtils.join(ids)
其中div.getTableName是拼接使用的
那么也就是说 这个 div.getTableName是可控制的
跟踪这个方法来到 src/main/java/com/cms/controller/admin/DivController.java

<img width="1107" height="735" alt="Image" src="https://github.com/user-attachments/assets/549f4ed4-f5ef-4cbc-b5d8-ba24bb457dd9" />

分析save方法

现在是前面delete的方法使用了拼接sql语句，拼接的sql语句呢就是getTableName这个方法
那么只要在使用delete删除方法的时候，在getTableName插入恶意语句，就可以实现SQL注入

<img width="823" height="191" alt="Image" src="https://github.com/user-attachments/assets/e558a6be-2fa4-45fa-9aa8-26e4e8af6c08" />

getTableName是获取表名的意思，那么现在是不是可以在创建表的时候将表名直接写成SQL注入语句
那么delete在获取表名的时候，就会将表名中的SQL注入语句拼接到DB.update中，
将tablename中的恶意SQL语句拿到数据库执行
实现SQL注入

复现：

在div/add接口处将 数据库表名称填入恶意payload 然后保存

cms_admin where id =20 or sleep(5) #

<img width="863" height="750" alt="Image" src="https://github.com/user-attachments/assets/5ba88867-edca-4cee-a2f6-fbb61a373d9c" />

<img width="863" height="750" alt="Image" src="https://github.com/user-attachments/assets/bd2dced0-6aca-4f63-a2f0-9244faa17b4e" />

构造数据包
调用delete接口 构造参数 divId=

<img width="927" height="704" alt="Image" src="https://github.com/user-attachments/assets/f0906dc7-3e8f-419f-a796-11ec7b5d4274" />

成功延时注入10秒 
不知道为啥是两倍 我填的5秒就延时10秒    填3秒就是延时6秒

JFinalCMS /admin/admin nameSQL注入 CVE-2024-24375

接口存在于/admin/admin

代码位于src/main/java/com/cms/entity/Admin.java  99行 findPage方法
在101行采用了拼接SQL语句的方式  对name参数 和 username参数进行拼接到sql语句
name和username参数都是可操控的

代码分析：

首先定义形参name，username，pageNumber，pageSize
用来接收前端传入的数据
判断 如果name不为空 就拼接sql语句 " and name like '%"+name+"%'";
判断 如果username不为空 拼接sql  " and username like '%"+username+"%'";

<img width="1110" height="735" alt="Image" src="https://github.com/user-attachments/assets/9ebdac34-d59b-4bd2-a67c-d4bab3793aa5" />

跟进findPage方法来到 controller层 index方法

<img width="794" height="357" alt="Image" src="https://github.com/user-attachments/assets/f2071b8f-eeb2-4347-b0c6-38d82a2b5452" />

代码分析：

定义变量name，username，pageNumber 用来接收存储前端传入的数据
判断 如果pageNumber为空，那么定义pageNumber为1
然后使用jfinal的数据库操作工具类调用findPage方法，里面传入name，username，pageNumber，pageSize

那么现在通过跟踪代码，发现接口位于admin/admin
在findPage方法中，使用拼接SQL语句方式的有name和username两个参数

构造注入参数接口
/admin/admin?name=
/admin/admin?username=

<img width="876" height="468" alt="Image" src="https://github.com/user-attachments/assets/48e4b114-ceaa-4eec-b85a-09cf42cc5920" />

两个接口参数都可以成功跑出来


SQL注入

位于 src/main/java/com/cms/controller/admin/ContentController.java  

data方法

<img width="1157" height="735" alt="Image" src="https://github.com/user-attachments/assets/650542d1-7031-4033-b327-5dec783e1767" />

代码分析：

定义三个变量用来接收传入的title，categoryId，pageNumber
判断，如果pageNumber不为空 则定义pageNumber为1

然后使用Content().dao()调用findPage方法

跟进findPage方法
来到src/main/java/com/cms/entity/Content.java    findPage方法处

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/f5632550-0cbc-4ab2-85d6-58c4e249cb39" />

在代码95行处使用了拼接SQL语句的方式
拼接了title参数
那么在title参数处很有可能存在sql注入漏洞
接口位于

<img width="773" height="449" alt="Image" src="https://github.com/user-attachments/assets/e65be0b2-97bd-4c78-b155-05a7399fa133" />

/admin/content
位于网站后台 /内容管理/数据管理处

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/ae58f0bf-1ef6-419e-aeb0-b29fea427531" />

<img width="1280" height="764" alt="Image" src="https://github.com/user-attachments/assets/1ac74541-58b1-4b21-aa91-1ef950d48ff7" />

<img width="1280" height="764" alt="Image" src="https://github.com/user-attachments/assets/198ae7ed-cb7c-4c0e-95a1-55c89951dd50" />

任意文件写入漏洞:

在网站后台系统管理/模板管理/新增处存在文件写入功能

<img width="860" height="750" alt="Image" src="https://github.com/user-attachments/assets/0af7e3d3-de4f-4eeb-b558-b300effc0139" />

来到代码src/main/java/com/cms/controller/admin/TemplateController.java
save()方法处

<img width="1160" height="735" alt="Image" src="https://github.com/user-attachments/assets/5513f229-48f9-4e1d-93c5-83a947f931e1" />

定义三个变量用来接收前端传入的 fileName content directory
if判断 如果directory不为空 那么filePath就为 
/+directory.replaceAll(",", "/")+"/"+fileName;

try捕获异常
使用FileUtils.write写入文件
new一个File 使用getWebRootPath获取项目根路径+ /templates/+filePath 
在save方法处打断点 然后随便上传一个文件 观察文件上传到哪里

<img width="851" height="679" alt="Image" src="https://github.com/user-attachments/assets/05880e92-984f-4546-9db0-39abde536a18" />

放在web根目录的default文件夹下
也就是 项目路径 src/main/webapp/templates/default/1.txt

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/6a79a24b-b93b-44b1-b662-d814d7059748" />

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/8b151884-2aa6-457e-8a04-f26224fd0ed0" />

构造数据包 尝试写入文件到其他目录  我要写到src目录下面
就是4个../ 

<img width="927" height="704" alt="Image" src="https://github.com/user-attachments/assets/d2ddeb97-42aa-4973-9acf-31f57f4a4a75" />

<img width="1161" height="735" alt="Image" src="https://github.com/user-attachments/assets/ade447ff-44aa-4481-afbc-707281921906" />

写入成功 存在任意文件写入
这个漏洞造成的根本原因就是因为在save方法中
没有对传入的文件内容，文件路径，文件后缀进行任何的过滤和校验
没有对../过滤，导致存在目录穿越漏洞，文件写入功能配合目录穿越，导致可以写入任意文件到任意目录下面

<img width="715" height="307" alt="Image" src="https://github.com/user-attachments/assets/81251237-0f99-4ed5-b1f5-3180854fb2bd" />

可以将webshell写入到static静态目录下面，这样就可以成功连接

<img width="927" height="704" alt="Image" src="https://github.com/user-attachments/assets/89540727-1ccc-4bcf-868f-7b29a0bf15ac" />

<img width="1161" height="735" alt="Image" src="https://github.com/user-attachments/assets/6f4a3267-1efd-4ed5-8131-2ade59522866" />

任意文件读取漏洞:

存在于src/main/java/com/cms/controller/admin/TemplateController.java edit方法

<img width="1161" height="735" alt="Image" src="https://github.com/user-attachments/assets/a6827f7d-0f59-423e-ad38-e0dd885db5e3" />

代码分析：

定义两个变量用来接收传入的fileName和directory
if判断 如果传入的fileName为空，那就报错
if判断 如果directory不为空那就将,替换成/ 再加上fileName
然后使用TemplateUtils.read来读取这个文件

<img width="642" height="173" alt="Image" src="https://github.com/user-attachments/assets/7d2d56e6-4cbc-4cfa-a99b-49d5337bc6f1" />

read方法代码分析：

首先定义形参templatePath
try catch捕获异常
定义path 使用getWebRootPath获取web根路径+/template/+templatePath
然后使用FileUtils.readFileToString来读取传入的templatePath UTF-8格式

<img width="927" height="704" alt="Image" src="https://github.com/user-attachments/assets/6ffcaa67-196f-4a59-94c6-6db1cc1f3677" />

读取成功 存在任意文件读取漏洞


任意文件删除漏洞:

同样存在于src/main/java/com/cms/controller/admin/TemplateController.java 

<img width="1161" height="735" alt="Image" src="https://github.com/user-attachments/assets/59d8430e-8a7c-4eb9-836e-1041236f3485" />

delete方法

<img width="927" height="704" alt="Image" src="https://github.com/user-attachments/assets/328572e2-e8a3-4188-9559-64769e67fb60" />
<img width="880" height="527" alt="Image" src="https://github.com/user-attachments/assets/e016389f-a54e-4dd1-b8b0-8995d52f39d8" />

删除成功 存在任意文件删除漏洞

在这几个文件操作漏洞产生的原因都是因为没有对路径做校验，攻击者可以通过构造特殊的路径 ../来实现目录穿越，写入，读取其他目录下的文件
并且在文件写入处没有对文件内容和后缀进行过滤，攻击者可以上传webshell或者恶意的文件到服务器
