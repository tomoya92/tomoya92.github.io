---
layout: post
title: 前后端开发中的鉴权
date: 2021-04-27 18:38:00
categories: 杂项
tags: token
author: 朋也
---

* content
{:toc}





目前网站开发中（包括app）基本上都采用的是前后端分离的模式，如何让后台信任前台的数据也有了新的方式

## 上古实现方式

学过servlet+jsp的基本上都知道session，当启动一个servlet+jsp开发的web服务后，浏览器访问一下任意的一个页面，打开浏览器的控制台在cookie里就会发现一个 `JSESSIONID` 的key，它还有一长串值，这时候就用到了cookie

学的时候老师都教过 session 是服务端生成并维护的，cookie是浏览器端（客户端）的东西。**细一点的老师可能还会介绍到session是要依赖cookie的**

正因为session是建立在cookie的基础上在客户端生成的，所以才能在浏览器控制台里看到 `JSESSIONID` 的cookie数据

http的每次请求都会默认带上所有的cookie数据，所以这个 `JSESSIONID` 也就同样的被带到后台了，有了这个数据后，服务端才能判断这次请求是不是从自家网站发起的，从而进行一系列其它的操作

下面举个例子（个人见解，可能不标准，仅做参考）

最常见的一个使用session的例子就是当用户登录后，把用户的对象保存在session里，以后再有浏览器请求发过来就直接从session里获取用户的信息而不是去数据库里查了，这个操作的大致流程如下

```
用户登录 -> 提交数据 -> 后台接收 -> 数据库校验 -> 正确的话获取session对象 -> 将用户数据保存在session里 -> 响应浏览器登录成功
```

下次用户请求其它需要用户登录的页面时的后台验证如下

```
用户请求 -> 后台获取session -> 根据请求中的cookie获取到 `JSESSIONID` -> 根据 `JSESSIONID` 查找保存在内存中的用户数据 -> 找到了渲染页面
```

**到这可能有人会问，既然session是服务端生成并维护的，那为啥在页面上可以取到里面的数据呢？**

有这疑问的可能忽略了一个地方，就是还没有前后台分离的时候，那时候页面的渲染是由后台完成的，也就是说见面上通过代码从服务端获取session里值的操作都是在后台执行的，执行完后生成好html代码再响应给浏览器，浏览器再将html代码给渲染成用户能看懂的页面

## 前后端分离实现方式

上面说到了，前后端没分离的时候，服务端渲染页面，可以在页面里尽情的操作session，那现在页面部分被分离出来了，又该怎么使用session呢？

答案是现在前后端分离的项目不再使用session了，有两种替换方式

1. 直接使用cookie
2. 通过接口返回鉴权信息，由客户端自己保存，浏览器上可以使用 localStorage或者sessionStorage来进行保存

以上两种方式的原理都是一样的，不过稍微有点区别

还是以用户登录为例

cookie的方式：

```
用户登录 -> 提交数据 -> 后台接收 -> 数据库校验 -> 正确的话获取cookie对象 -> 将用户数据保存在cookie里 -> 响应浏览器登录成功
```

> 等等，上面不是说cookie是客户端的吗，那为啥这时候又能在服务端操作cookie了呢？
>
> 有这疑问的还是http的请求没弄明白，上面说到了一个http请求被发起时会将浏览器里cookie里的所有数据都带着发送给后台，那既然有请求，当然也有响应，响应里同样的也有所有的cookie信息
>
> 如果不对cookie进行操作的话，每次请求会原封不动的把请求送过来的cookie又原封不动的响应给浏览器
>
> 上面通过服务端对cookie数据进行了更新，然后响应给浏览器，浏览器再去更新本地的cookie信息，从而达到了服务端操作cookie的功能

**用cookie有一点方便的地方就是，不管前台还是后台，都能访问并进行更新**

localStorage/sessionStorage的方式：

```
用户登录 -> 提交数据 -> 后台接收 -> 数据库校验 -> 响应浏览器登录成功（响应的数据里会带上能代表用户的一个令牌，以下就以token来介绍）-> 客户端拿到`用户代表`后保存在storage里 -> 下次请求放在header里或者参数里都行
```

可以看出使用 localStorage/sessionStorage的方式 只能通过接口的形式与后台进行交互了

## 优化

上古的方式没啥好优化的，因为没涉及到操作数据库，每次用的时候直接从内存里获取session的数据即可

但有些大型网站做分布式的话，session就不好使了，见下面例子

```
浏览器1 请求 -> 带着JSESSIONID -> 服务端1接收到 JSESSIONID 进行数据的操作 -> 将数据写入到服务端1所在电脑的内存里
浏览器2 请求 -> 带着JSESSIONID -> 服务端2接收到 JSESSIONID 进行数据的操作 -> 将数据写入到服务端2所在电脑的内存里
```

可以看出客户端还是一个，JSESSIONID也是一个，但后台因为分布式被拆分成了若干个，这时候如果用户登录的请求发到服务器1上，session的数据就被保存在服务器1上了，下一次用户请求到服务2上的话，再从session里取值就会导致取不到

解决办法就是再找一个服务，用来管理这些服务器产生的session数据，比如常用的redis，对应到框架可以选择 spring-session，稍微配置一下就可以用了

session被统一管理后，以后不管用户请求到哪个服务器，要获取session的话，都会去redis里找，这找到的就一定是唯一的那一个session了，也就不怕不存在或者重复了

----

由于前端数据造假起来非常容易，所以就导致了前端的数据是不可信的，这时候就需要在每次请求的时候都对用户传过来的token进行校验，最可信的校验方式就是拿着token去数据库里查，能查到就是真的数据，查不到就是假的数据

这么一来，对数据库的操作就会变多，数据库的压力就大了，所以优化的部分就在如何能快速的拿着token不经过数据库拿到数据了，常见的几种方式就是加缓存，常用的缓存有 redis，ehcache等

用户登录请求发送，从数据库校验完登录数据的准确性，然后将用户的信息保存在redis里或者用ehcache缓存下来，最后将令牌（token）返回给前端，前端保存token，下次请求带上token，后台拿着token去redis或者ehcache里找相关的数据进行校验

## 单点登录

最后聊一下单点登录，字面意思就是在chrome上登录后，然后换成firefox再登录，再回到chrome上，chrome上保存的token就失效的一个机制

这种实现起来最简单，在登录时直接将用户的token换一个新的就行，下次用户再请求的带着老的token就查不到数据了，从而也就失败了

还有一种，允许2个登录点的存在，这种就需要数据库来配合着实现了。

```
用户登录请求 -> 后台查保存了多少个token了（保存token的数据可以是一张单独的表，后面好扩展） -> >=2的时候，就把最老的那个token删掉，再创建一个新的token存入数据库
```

## 可能碰到的问题

- 为啥我用cookie，在后台设置了，浏览器里也能看见，但就是获取不到？

    > 这种一般都是设置cookie时的域名不对导致的
