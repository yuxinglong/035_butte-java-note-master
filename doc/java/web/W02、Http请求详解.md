# 一、Http协议简介

## 1、概念说明

HTTP超文本传输协议,是用于从万维网服务器传输超文本到本地浏览器的传送协议，基于TCP/IP通信协议来传递数据：HTML文件、图片、查询数据等。HTTP协议基于客户端-服务端架构模式。浏览器作为HTTP客户端通过URL向服务端即WEB服务器发送请求。Web服务器根据接收到的请求后，处理完请求后向客户端发送响应信息。

![](https://images.gitee.com/uploads/images/2022/0220/200942_bb0a74aa_5064118.png "03-1.png")

## 2、协议特点

- 简单快速

请求服务器时，只需传送请求方法和路径。请求类型常用GET、POST。由于HTTP协议简单，使得HTTP服务器的程序规模小，因而通信速度很快。

- 灵活：

HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。

- 无连接

无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。

- 无状态

HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态则后续处理需要前面的信息，没有则需要重新请求，这样可能导致每次连接传送的数据量增大。

- 支持客户/服务器模式

# 二、Http请求详解

## 1、请求接口

```java
public class ServletOneImpl extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("doGet...");
        response.setContentType("text/html;charset=utf-8");
        String userName = request.getParameter("userName") ;
        response.getWriter().print("Hello:"+userName);
    }
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("doPost...");
        response.setContentType("text/html;charset=utf-8");
        String userName = request.getParameter("userName") ;
        response.getWriter().print("Hello:"+userName);
    }
}
```

## 2、请求内容

- Get请求

地址：`http://localhost:6003/servletOneImpl?userName=cicada`

```
GET /servletOneImpl?userName=cicada HTTP/1.1
Host: localhost:6003
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
DNT: 1
Referer: http://localhost:6003/request.jsp
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: JSESSIONID=183C0F91B49A025C795FBC3067B37BC8
```

- Post请求

地址：`http://localhost:6003/servletOneImpl`

```
POST /servletOneImpl HTTP/1.1
Host: localhost:6003
Connection: keep-alive
Content-Length: 15
Cache-Control: max-age=0
Origin: http://localhost:6003
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) 
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
DNT: 1
Referer: http://localhost:6003/request.jsp
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: JSESSIONID=183C0F91B49A025C795FBC3067B37BC8
```

## 3、参数说明

- Method：常见GET 和 POST，另外还有DELETE、PUT 等 ;
- URI：/servletOneImpl，和Host组成请求的URL ;
- HTTP/1.1：传输协议Http，版本1.1 ;
- Host：请求资源所在的主机和端口 ;
- Connection：TCP连接默认不关闭，可以被多个请求复用 ;
- Upgrade-Insecure-Requests：浏览器不再显示 https 页面中的 http 请求警报 ;
- Content-Length：指示实体主体大小，以字节为单位十进制数，发送到接收方。
- Cache-Control：max-age=0使用缓存，但是会立即过期 ;
- User-Agent：客户端浏览器类型、版本、操作系统等信息 ;
- Content-Type: 请求或响应中，传输资源类型信息 ;
- Origin：当前请求出自的站点 ;
- Accept：客户端声明自己可以接收的资源格式 ;
- DNT: 请求禁用用户追踪 ;
- Referer：告诉服务器该网页是从哪个页面链接过来 ;
- Accept-Encoding: 声明浏览器支持的编码类型 ;
- Accept-Language: 声明浏览器支持的语言类型 ;
- Cookie: 辨识存储在客户端的缓存数据，通常会加密 ;

## 4、GET和POST区别

- 浏览器端

从浏览器角度看这个两种请求的区别：GET方式读取资源，比如Get到静态页面，即使多次读取不会对访问数据产生影响，也被称为"幂等"操作。POST方式在页面中定义表单，提交表单会把数据提交到服务器，而且多数情况下会产生数据，比如常用的保存数据接口，并非"幂等"操作，不幂等也就意味着不能随意多次执行。

- 服务接口

这里指用Ajax程序请求服务接口，提交的请求类型。或者其他Http请求工具类，还有情况是微服务中各种Feign接口间的请求。这种情况接口发送请求时，限制相对较少，比如REST风格接口常用GET、POST、PUT、DELETE，几种方式分别获取、创建、更新、删除 资源，

# 三、Https请求协议

## 1、Https简介

- 基础概念

HTTPS：是以安全为准则的HTTP通道，是HTTP的安全版，在HTTP请求上加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。简单来说，HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，要比http协议安全。

- 安全模式

HTTPS协议的主要作用可以分为两种：一种是建立一个信息安全通道，来保证数据传输的安全；另一种就是确认网站的真实性。

## 2、Https工作原理

![](https://images.gitee.com/uploads/images/2022/0220/201006_9110d138_5064118.png "03-2.png")

- 客户基于Https方式访问服务端，与服务器建立SSL连接 ;
- 服务端收到请求后，会将包含公钥的证书传送给客户端 ;
- 客户端与服务端进行协调SSL连接的安全等级，也就是指加密的等级 ;
- 客户端根据双方同意的安全等级，建立会话密钥，使用公钥将会话密钥加密，并传送给服务端 ;
- 服务端使用私钥解密出会话中传递的内容，使用会话密钥加密与客户端之间的通信 ;

## 3、Https和Http区别

- 安全证书

Https协议需要到CA申请证书，一般免费证书较少，因而需要一定费用。

- 数据传输

Http是超文本传输协议，信息是明文传输，https则是具有安全性的ssl加密传输协议。

- 连接方式

Http和Https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。


# 四、TCP传输协议

## 1、TCP协议简介

TCP传输控制协议是一种面向连接的、可靠的、基于字节流的传输层通信协议。TCP用意在于适应并支持多网络应用的分层协议层次结构。

## 2、三次握手

这一场景在生活中可以描述为通话：
```
甲：你好,我是甲,你是乙吗;
乙：你好甲：我是乙;
甲：正好找你有点事情,身份确认：
```

![](https://images.gitee.com/uploads/images/2022/0220/201135_2418769b_5064118.png "03-3.png")

- 第一次握手

**客户端**主动向服务器发起请求连接，请求报文中发送SYN=1，此时随机生成初始序列号seq=x，此时，客户端进程进入SYN-SENT同步已发送状态。

- 第二次握手

**服务端**收到请求报文后，确认客户的SYN，如果请求没有拒绝，则发出确认报文。报文中应该ACK=1，SYN=1，确认号是ack=x+1，同时自己也发送一个SYN包seq=y，此时，服务器进程进入SYN-RCVD同步收到状态。

- 第三次握手

**客户端**收到确认后，需要向服务器确认报文的ACK=1，ack=y+1，此时，TCP连接建立，客户端进入ESTABLISHED已建立连接状态。完成三次握手，客户端与服务器开始传送数据。

## 3、四次挥手

![](https://images.gitee.com/uploads/images/2022/0220/201147_d69ad819_5064118.png "03-4.png")

- 第一次挥手  

**客户端**发送一个结束FIN，用来主动关闭和服务端的数据传输，释放连接且停止发送数据，报文首部：FIN=1，序列号seq=u；随后客户端进入终止等待1状态FIN-WAIT-1。

- 第二次挥手

**服务端**收到这个FIN，发出确认报文ACK=1，确认收到序号是收到的序号+1，即ack=u+1，且带上自己的序列号seq=v，和SYN一样，一个FIN将占用一个序号。如此，服务器通知应用进程，客户端已经没有数据要发送，如果服务器发送数据，客户端依然要接收，该状态会持续一段时间，服务端进入关闭等待状态CLOSE-WAIT。客户端收到服务器的确认请求后，进入终止等待2状态FIN-WAIT-2，等待服务器发送连接释放报文。

- 第三次挥手

**服务器**向客户端发送释放连接报文FIN=1，ack=u+1，此时服务端还处于半关闭状态，服务器可能还会发送一些数据，此时序列号为seq=w，如此，服务器进入最后确认状态LAST-ACK，等待客户端的确认。

- 第四次挥手

**客户端**收到服务器的连接释放报文后，回发确认，ACK=1，ack=w+1，序列号是seq=u+1，如此，客户端进入时间等待状态TIME-WAIT。此时TCP连接还没有释放，必须经过最长报文段寿命的时间后，才进入CLOSED状态。MSL：最长报文段寿命，一般2分钟，TCP连接释放时，主动方必须经过2MSL后才进入CLOSED状态，因此主动方关闭时间比较晚。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent