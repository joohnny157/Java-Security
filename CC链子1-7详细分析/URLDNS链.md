URLDNS 链是 Java 反序列化漏洞利用中的一种技术，

具体是 ysoserial 工具中的一个 gadget（URLDNS），利用 java.net.URL 类的 DNS 查询功能来检测或验证反序列化漏洞是否存在。

它与 CC1 链不同，URLDNS 链本身不直接执行系统命令（如弹出计算器），而是依赖 URL 对象的反序列化行为触发 DNS 请求。

调用链：

HashMap.readObject() 

HashMap.putVal() 

HashMap.hash() 

  URL.hashCode() 
    
  URLStreamHandler.hashCode() 
        
  URLStreamHandler.getHostAddress()

在HashMap.readObject()方法中调用了HashMap.putVal()方法，传入hash(key)

<img width="2097" height="1470" alt="image" src="https://github.com/user-attachments/assets/087c2a92-5beb-4cf3-b427-1f04890d2f75" />

接着，这个hash()方法中的key，又会调用key.hashCode()方法

<img width="1108" height="178" alt="image" src="https://github.com/user-attachments/assets/26268c04-d8cf-43a9-a0c9-d67c8dc1894d" />

在这里的这个key是一个object类型，
那么意思是不是就是可以调用其他对象的hashCode()方法。

这个key是什么呢？

就是HashMap.put（key，value），

第一个就是key，
在URL类中存在hashCode方法，
如果让这个key = URL 的话，就会触发URL.hashCode()方法

<img width="1665" height="837" alt="image" src="https://github.com/user-attachments/assets/f4996a06-9c68-40e9-85d7-71d057cc30f0" />

这里如果hashCode不等于-1。
那就直接return回去。
那如果等于-1 就往下走。
handler.hashCode(this)。

调用handler的hashCode方法将URL传递进去，
在URL已经定义好了，默认为-1

<img width="684" height="54" alt="image" src="https://github.com/user-attachments/assets/2031c90f-6fa6-4c80-bd87-b8a297118ba1" />

这个handler对象是一个 URLStreamHandler

<img width="697" height="61" alt="image" src="https://github.com/user-attachments/assets/3c8a87cc-c90f-449c-aa6d-a04114dd4030" />

跟踪查看URLStreamHandler中的hashCode()方法

<img width="2044" height="1470" alt="image" src="https://github.com/user-attachments/assets/b13d780b-744e-4e45-bcaa-fa49bf341b77" />

在353行会对传入的URL进行请求，如果传入DNSlog，那么在反序列化之后查看DNSlog日志，就可以判断这条链子有没有被执行成功。

这条URLDNS链是没有版本限制的，通常用来判断目标是否存在反序列化漏洞


Payload：

首先创建一个HashMap对象和URL对象，
yakit生成一个dns地址

<img width="994" height="214" alt="image" src="https://github.com/user-attachments/assets/b0e2a75c-147f-493c-952d-89e0186f3e83" />

现在只需要调用put方法，传入key为URL，就可以触发这个dns，

map.put(url,"1")

这里就会调用url中的hashCode方法，
再调用URLStreamHandler.hashCode()方法，
最后调用URLStreamHandler.getHostAddress()方法来触发请求

<img width="1801" height="1260" alt="image" src="https://github.com/user-attachments/assets/dad2c7d1-cb69-40e5-86de-e1f159931d52" />

那么在这个地方存在一个问题，
那就是在这里是序列化的时候就已经触发了，

但是目的是要让他在序列化的时候不触发，要他在反序列化的时候才触发，这样才是正确的。

在前面提到过，在URL中的hashCode方法如果不等于-1那就直接返回，
只有等于-1才会使用handler下的hashCode方法

<img width="807" height="280" alt="image" src="https://github.com/user-attachments/assets/06478dff-2d91-43a6-beab-342b21ddf847" />

在URL中的hashCode是private私有的，在外部是无法修改的

<img width="682" height="73" alt="image" src="https://github.com/user-attachments/assets/a6b3d250-5fb3-4c17-98e7-bdc299f13005" />

但是这里可以调用反射机制来修改这个值

<img width="1065" height="364" alt="image" src="https://github.com/user-attachments/assets/49bafe5a-e7b1-40b0-a8a4-bb312f955503" />

使用getClass来获取url对象，
getDeclaredField获取url中的hashCode，

最后set两个参数，第一个就是传入url，第二个参数只要不为-1就行

那么现在这个这里就不等于-1了，等于2，那么在上面的if判断就成立了，直接return hashCode，

那这样不就是走不到handler中的hashCode方法了吗，如果走不到handler，那还怎么触发最后的请求？

这里的解决方法就是

在map.put完之后，再把url的hashCode给改回-1就可以触发了

<img width="1036" height="381" alt="image" src="https://github.com/user-attachments/assets/2a24c5dc-f376-4ec5-a997-abcf465698a9" />

这样，在序列化的时候不会触发

将上面的代码序列化成ser.bin文件，然后直接反序列化ser.bin

<img width="1308" height="898" alt="image" src="https://github.com/user-attachments/assets/bbb2d1b7-84c9-4a29-ab6d-7a7314a7cd66" />

成功收到请求

<img width="1651" height="1110" alt="image" src="https://github.com/user-attachments/assets/b883d186-3c87-4ff7-89e8-26e5965632eb" />

