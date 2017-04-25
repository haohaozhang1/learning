
**HTTP协议格式请求例子**

**GET https://baidu.com HTTP/1.1**            
**Host: gwtest.pawjzs.com**     
**Connection: Keep-Alive**        
**Accept-Encoding: gzip**          
**User-Agent: okhttp/3.2.0**            

**username=123&passwrd=123**           


----------


**概述：**   

**请求方法 url 协议版本**         
**header字段名称：值**           
**....**           
**header字段名称：值**             
**空行**           
**请求包体**              


----------


**HTTP协议格式响应例子**           

**HTTP/1.1 200 OK**           
**Server: nginx/1.8.1**             
**Date: Tue, 25 Apr 2017 06:11:41 GMT**           
**Content-Type: application/json;charset=UTF-8**               
**Connection: keep-alive**              
**Vary: Accept-Encoding**            
**Set-Cookie: BIGipServerUAT_1_nginx_pool=489820844.36895.0000; path=/**           
**Content-Length: 129**           

**{"status":"SUCCESS","result":[]}**                


----------


**概述：**           

**协议版本 状态码 状态码描述**             
**header字段key：value**                   
**...**                   
**header字段key：value**                 
**空行**                
**响应包体**                     


----------


GMT是格林尼治所在地的标准时间，HTTP协议中用到的都是这个格式，防止时区不同时间有差异性
Connection：表示TCP连接是长连接，可以继续发送数据
其他的字段意思一看就明了

另外对HTTP的响应数字做一下概述

1xx:信息响应类，表示接收到请求并且继续处理 

2xx:处理成功响应类，表示动作被成功接收、理解和接受 

3xx:重定向响应类，为了完成指定的动作，必须接受进一步处理 

4xx:客户端错误，客户请求包含语法错误或者是不能正确执行 

5xx:服务端错误，服务器不能正确执行一个正确的请求 

----------


**HTTP交互过程**
1、组装HTTP请求，请求头，请求内容
2、解析DNS
3、建立TCP链接
4、发送HTTP请求
5、服务器的永久重定向响应
6、发起请求方跟踪重定向地址
7、服务器处理请求
8、服务器返回response
9、服务器释放TCP链接
10、请求方解析response
