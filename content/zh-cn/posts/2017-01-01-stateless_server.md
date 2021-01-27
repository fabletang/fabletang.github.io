+++
title= "stateless server 最佳实践"
description= "无状态服务的实践"
date = "2017-01-01"
tags = [
    "practice"
]
categories = [
  "技术"
]
series = [
  "概念"
]
#[toc]
#menu = "main"
+++

   stateless server 即无状态服务。相对与传统的http server存在session,无session的
http server称为stateless server，现在一般应用于restful service.

#### Why: 
  session的概念存在这么多年，一定有其合理之处,为什么要抛弃它?

1. session的起源:
  session根源于http的cookie. http协议本身是无状态的协议，server是一问一答，答后不管。为了加入
状态，来辨识是否访问／是否登录的需要，加入了cookie。服务器的session就是对应浏览器端的cookie.

2. 为什么抛弃session:
  1. 高并发的访问，session制约了server的水平扩展。在负载均衡的多个server群，虽然可以用高速缓存比
如redis来管理全局session.但是毕竟多存在了一个节点，削弱了系统稳定性。
  2. session需要资源开销，在tomcat中，每个session至少耗费4k内存。
  3. session存在安全缺陷，一般session会以明文形式写入cookie,cookie是保存在浏览器端的硬盘，由此引发
的安全问题可以写成一本书了。即使禁用cookie, session也容易被窃取，CSRF跨域攻击就是窃取session的
安全问题。
  4. 随着server的功能增多，有时候需要跨域访问的时候，session成为了障碍。比如网页需要访问两个服务器的
资源，而且都必须要要登录授权保持状态,这样需要两个sessionid,但是浏览器只支持单一session. 


#### What:
  stateless server是否就是服务器不需要state状态？
  需要授权的资源如果不保存其状态，比如用户id／角色。每次访问都要重新提交验证信息。服务器可能要重复
验证／查询数据库,这样会带来额外的资源开销。
  所以，stateless应该理解为服务器不保存状态。

#### How:
  server需要状态，但是不保存状态。怎么做？
采用jwt,状态保存在客户端。jwt 即json web token. server 把状态放入jwt,加密，以字符串的方式发送给客户端。

  采用jwt的好处:

1. 可以封装状态，以key／value字符的形式保存状态。
2. 安全。jwt一般采用对称或者非对称加密。加密密钥存放在server.可以防止jwt被篡改。jwt同时有时效性，合适的有效期可以减少重放攻击的可能性。

