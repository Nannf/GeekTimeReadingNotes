##### 网络世界

- http协议下的世界，这个就是http协议和一些资源构成的，因为超文本的表达性很强，所以基本其他不是http协议的数据都可以封装成http
- 邮件，ftp服务



##### 浏览器

- user agent 用户代理
- http协议中的请求方



##### web服务器

- http协议中的应答方
- apache、nginx、tomcat



##### cdn

- 内容分发网络
- http协议中的缓存和代理
- 代替服务端响应客户端请求



##### 爬虫

- 也是用户代理



##### html  & WebService  & WAF

- html 是http传输的主要实体
- webservice 类似于一种编码指导
- WAF 软件防火墙



##### TCP/IP

- 标准通信协议
- 四层：应用层、传输层、网络层、链接层
- ip协议：解决寻址和路由
- tcp协议： 传输控制协议，基于ip协议提供可靠的、字节流形式的通信
- http协议是基于tcp/ip协议的



##### DNS

- 域名系统
- 出现的原因是因为ip地址太难记了
- 域名从左到右，级别依次升高，最右边是顶级域名
- 域名的表现形式是一个好记得数字和英文得序列，比如www.12306.com
- 把序列转换成ip地址就是域名解析
- dns是http访问的第一步



##### URL & URI

- DNS  和 ip 只能找到哪台机器
- 一台机器上可供别人访问的资源是有限的，每个资源都需要一个独一无二的标识符，URI应运而生
- URL是URI的子集，URI是统一资源标识符，URL是统一资源定位符

```tcl
http://nginx.org/en/download.html
```

- 协议： http
- 域名： nginx.org
- 资源标识符：/en/download.html



##### https

- 之前的http是http over tcp/ip
- https over SSL/TLS; SSL/TLS over tcp/ip
- SSL : Secure  Socket Layer 标准化后更名为 TLS(Transport Layer Security)



##### 代理

- http请求方和应答方的一个中转站
- 可以转发请求方的请求，也可以转发应答方的应答
- 分类
  - 匿名代理： 外界只能看到代理服务器
  - 透明代理： 外界知道代理服务器，也知道客户端
  - 正向代理： 靠近客户端，代表客户端向服务器发送请求
  - 反向代理： 靠近服务器，代表服务器响应客户端请求
- CDN代表服务器向客户端响应，属于反向代理。
- 功能：
  - 负载均衡，客户端只是想找一个人响应，代理服务器可以做请求的分发
  - 内容缓存，虽然http是无状态的，但是代理可以缓存
  - 安全防护，可以在服务器处置之前进行一波过滤
  - 数据处理，可以压缩、加密



