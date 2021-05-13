---
title: 深入理解JSP
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-09 22:19:10
password:
summary:
tags:
categories: java
---

#### JSP介绍

 JSP（Java server page）是Java EE规范最基本成员，他是Java Web开发的重点知识，虽然我们一直在用，但其原理知之甚少。今天重点研究一些JSP核心内容以及其工作原理。

  JSP和Servlet的本质是一样的，因为JSP最终需要编译成Servlet才能运行，换句话说JSP是生成Servler的草稿文件。

  JSP比较简单，就是在HTML中嵌入Java代码，或者使用JSP标签，包括使用用户自定义标签，从而可以动态的提供内容。早起JSP应用比较广泛，一个web应用可以全部由JSP页面组成，只需要少量的JavaBean即可，但是这样导致了JSP职责过于复杂，这是Java EE标准的出现无疑是雪中送炭，因此JSP慢慢发展成单一的表现技术，不再承担业务逻辑组件以及持久层组件的责任。

#### JSP基本原理

  JSP的本质是servlet，当用户指定servlet发送请求时，servlet利用输出流动态生成HTML页,由于包含大量的HTML标签,静态文本等格式导致servlet的开发效率极低，所有的表现逻辑，包括布局、色彩及图像等，都必须耦合在Java代码中。jsp使得静态的部分无需Java程序控制，只有那些需要从数据库读取或者需要动态生成的页面内容才使用Java脚本控制。

#### JSP和Servlet是什么关系?

　　Servlet是一个特殊的Java程序，它运行于服务器的JVM中，能够依靠服务器的支持向浏览器提供显示内容。JSP本质上是Servlet的一种简易形式，JSP会被服务器处理成一个类似于Servlet的Java程序，可以简化页面内容的生成。Servlet和JSP最主要的不同点在于，Servlet的应用逻辑是在Java文件中，并且完全从表示层中的HTML分离开来。而JSP的情况是Java和HTML可以组合成一个扩展名为.jsp的文件。有人说，Servlet就是在Java中写HTML，而JSP就是在HTML中写Java代码，当然这个说法是很片面且不够准确的。JSP侧重于视图，Servlet更侧重于控制逻辑，在MVC架构模式中，JSP适合充当视图(view)而Servlet适合充当控制器(controller)。

**思考:为什么项目部署后修改jsp不需要重新启动?**(可查阅jTomcat类加载器架构了解)

#### JSP中的四种作用域

JSP中的四种作用域包括page、request、session和application，具体来说：

- page代表与一个页面相关的对象和属性。
- request代表与Web客户机发出的一个请求相关的对象和属性。一个请求可能跨越多个页面，涉及多个Web组件;需要在页面显示的临时数据可以置于此作用域。
- session代表与某个用户与服务器建立的一次会话相关的对象和属性。跟某个用户相关的数据应该放在用户自己的session中。
- application代表与整个Web应用程序相关的对象和属性，它实质上是跨越整个Web应用程序，包括多个页面、请求和会话的一个全局作用域。

#### 实现会话跟踪的技术

　　由于HTTP协议本身是无状态的，服务器为了区分不同的用户，就需要对用户会话进行跟踪，简单的说就是为用户进行登记，为用户分配唯一的ID，下一次用户在请求中包含此ID，服务器据此判断到底是哪一个用户。

　　1)URL 重写：在URL中添加用户会话的信息作为请求的参数，或者将唯一的会话ID添加到URL结尾以标识一个会话。

　　2) 设置表单隐藏域：将和会话跟踪相关的字段添加到隐式表单域中，这些信息不会在浏览器中显示但是提交表单时会提交给服务器。

　　这两种方式很难处理跨越多个页面的信息传递，因为如果每次都要修改URL或在页面中添加隐式表单域来存储用户会话相关信息，事情将变得非常麻烦。

　　3)cookie：cookie有两种，一种是基于窗口的，浏览器窗口关闭后，cookie就没有了;另一种是将信息存储在一个临时文件中，并设置存在的时间。当用户通过浏览器和服务器建立一次会话后，会话ID就会随响应信息返回存储在基于窗口的cookie中，那就意味着只要浏览器没有关闭，会话没有超时，下一次请求时这个会话ID又会提交给服务器让服务器识别用户身份。会话中可以为用户保存信息。会话对象是在服务器内存中的，而基于窗口的cookie是在客户端内存中的。如果浏览器禁用了cookie，那么就需要通过下面两种方式进行会话跟踪。当然，在使用cookie时要注意几点：首先不要在cookie中存放敏感信息;其次cookie存储的数据量有限(4k)，不能将过多的内容存储cookie中;再者浏览器通常只允许一个站点最多存放20个cookie。当然，和用户会话相关的其他信息(除了会话ID)也可以存在cookie方便进行会话跟踪。

　　4)HttpSession：在所有会话跟踪技术中，HttpSession对象是最强大也是功能最多的。当一个用户第一次访问某个网站时会自动创建HttpSession，每个用户可以访问他自己的HttpSession。可以通过HttpServletRequest对象的getSession方法获得HttpSession，通过HttpSession的setAttribute方法可以将一个值放在HttpSession中，通过调用HttpSession对象的getAttribute方法，同时传入属性名就可以获取保存在HttpSession中的对象。与上面三种方式不同的是，HttpSession放在服务器的内存中，因此不要将过大的对象放在里面，即使目前的Servlet容器可以在内存将满时将HttpSession中的对象移到其他存储设备中，但是这样势必影响性能。添加到HttpSession中的值可以是任意Java对象，这个对象最好实现了Serializable接口，这样Servlet容器在必要的时候可以将其序列化到文件中，否则在序列化时就会出现异常。

#### 过滤器作用和用法

　　Java Web开发中的过滤器(filter)是从Servlet 2.3规范开始增加的功能，并在Servlet 2.4规范中得到增强。对Web应用来说，过滤器是一个驻留在服务器端的Web组件，它可以截取客户端和服务器之间的请求与响应信息，并对这些信息进行过滤。当Web容器接受到一个对资源的请求时，它将判断是否有过滤器与这个资源相关联。如果有，那么容器将把请求交给过滤器进行处理。在过滤器中，你可以改变请求的内容，或者重新设置请求的报头信息，然后再将请求发送给目标资源。当目标资源对请求作出响应时候，容器同样会将响应先转发给过滤器，在过滤器中你可以对响应的内容进行转换，然后再将响应发送到客户端。

　　常见的过滤器用途主要包括：对用户请求进行统一认证、对用户的访问请求进行记录和审核、对用户发送的数据进行过滤或替换、转换图象格式、对响应内容进行压缩以减少传输量、对请求或响应进行加解密处理、触发资源访问事件、对XML的输出应用XSLT等。

　　过滤器相关的接口主要有：Filter、FilterConfig和FilterChain。

　　监听器有哪些作用和用法?

　　Java Web开发中的监听器(listener)就是application、session、request三个对象创建、销毁或者往其中添加修改删除属性时自动执行代码的功能组件，如下所示：

　　ServletContextListener：对Servlet上下文的创建和销毁进行监听。

　　ervletContextAttributeListener：监听Servlet上下文属性的添加、删除和替换。

　　HttpSessionAttributeListener：对Session对象中属性的添加、删除和替换进行监听。

　　ServletRequestListener：对请求对象的初始化和销毁进行监听。

　　ServletRequestAttributeListener：对请求对象属性的添加、删除和替换进行监听。

　　HttpSessionListener：对Session的创建和销毁进行监听。

　　补充： session的销毁有两种情况：

　　session超时(可以在web.xml中通过/标签配置超时时间);

　　通过调用session对象的invalidate()方法使session失效。

注意:为什么项目部署后修改jsp不需要重新启动?