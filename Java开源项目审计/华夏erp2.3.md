管伊佳ERP（原名华夏ERP）基于SpringBoot框架和SaaS模式，立志为中小企业提供开源好用的ERP软
件，目前专注进销存+财务+生产功能。主要模块有零售管理、采购管理、销售管理、仓库管理、财务管
理、报表查询、系统管理等。支持预付款、收入支出、仓库调拨、组装拆卸、订单等特色功能。拥有商
品库存、出入库统计等报表。同时对角色和权限进行了细致全面控制，精确到每个按钮和菜单。


该版本项目存在

1、Fastjson反序列化漏洞

2、SQL注入漏洞

3、权限绕过漏洞

4、越权漏洞

5、XSS漏洞

SQL注入漏洞

全局搜索${ 检查是否存在拼接SQL语句

<img width="821" height="432" alt="Image" src="https://github.com/user-attachments/assets/7a643ca1-d92d-4ae6-81d3-87223b2fffee" />

搜索发现存在很多
来到mapper层 
src/main/resources/mapper_xml/UserMapperEx.xml

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/c3ab9ce7-553f-45c5-bbff-6e9e06a23877" />

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/d52e66eb-05eb-4e7e-8bd3-250ee0464fdf" />

传入两个参数 userName，loginName
继续跟进

来到UserController












