
业务方反应调用接口超时，但是在服务端监控并没有看到5xx异常， 于是我们模拟一下请求超时时发生了什么？


## 1\.openresty模拟长耗时服务端


延迟5s响应



```


|  | error_log logs/error.log; |
| --- | --- |
|  |  |
|  | http { |
|  | server { |
|  | listen 80; |
|  | charset  utf-8; |
|  |  |
|  | location /reqtimeout { |
|  | default_type text/html; |
|  | content_by_lua ' |
|  | local  start  = os.clock() |
|  | while os.clock() - start  <  5 do  end |
|  | ngx.say("delay  success!!") |
|  | '; |
|  | } |
|  | } |
|  | } |


```

## 2\.golang和.net默认的httpclient对外都只有一个timeout设置


用于控制请求、响应的整体时间



> .net httpclient 默认timeout\= 100s；
> golang net/http 无默认值设置，强烈推荐设置timeout，以避免服务端慢响应拖垮客户端。



```


|  | static void Main(string[] args) |
| --- | --- |
|  | { |
|  | Console.WriteLine("Hello, World!"); |
|  | var a =  HttpReqTimeout(); |
|  | Console.WriteLine(a.Result); |
|  | } |
|  |  |
|  | static async  Task<string> HttpReqTimeout() |
|  | { |
|  | var handler = new SocketsHttpHandler |
|  | { |
|  | PooledConnectionLifetime = TimeSpan.FromMinutes(1) |
|  | }; |
|  | using (var hc = new HttpClient(handler)) |
|  | { |
|  | hc.Timeout = TimeSpan.FromSeconds(3); |
|  | return  await hc.GetStringAsync("http://localhost/reqtimeout"); |
|  | } |
|  | } |


```

dotnet run ./ 显示客户端请求3s超时，爆出异常



```


|  | Hello, World! |
| --- | --- |
|  | Unhandled exception. System.AggregateException: One or more errors occurred. (A task was canceled.) |
|  | ---> System.Threading.Tasks.TaskCanceledException: A task was canceled. |
|  | at System.Threading.Tasks.Task.GetExceptions(Boolean includeTaskCanceledExceptions) |
|  | at System.Threading.Tasks.Task.ThrowIfExceptional(Boolean includeTaskCanceledExceptions) |
|  | at System.Threading.Tasks.Task`1.GetResultCore(Boolean waitCompletionNotification) |
|  | at ConsoleApp1.Program.Main(String[] args) in /Users/admin/RiderProjects/TestHttpClientFactory/ConsoleApp1/Program.cs:line 9 |
|  | --- End of stack trace from previous location --- |
|  |  |
|  | --- End of inner exception stack trace --- |
|  | at System.Threading.Tasks.Task.ThrowIfExceptional(Boolean includeTaskCanceledExceptions) |
|  | at System.Threading.Tasks.Task`1.GetResultCore(Boolean waitCompletionNotification) |
|  | at ConsoleApp1.Program.Main(String[] args) in /Users/admin/RiderProjects/TestHttpClientFactory/ConsoleApp1/Program.cs:line 9 |


```

openresty服务端日志，显示执行完成，返回200ok：



```


|  | 127.0.0.1 - - [04/Dec/2024:15:17:50 +0800] "GET /reqtimeout HTTP/1.1" 200 28 "-" "-" |
| --- | --- |


```

这也正是对应上了业务方的反馈和服务端的监控现象（无5xx报错）。


## 3\.wireshark抓包看实质


`tcp.port == 80 && ip.addr ==127.0.0.1 && ip.dst ==127.0.0.1`


![](https://files.mdnice.com/user/4236/278536c6-4ca2-4978-8bfd-24637dd9d2dc.png)


从tcp抓包过程看，分为三阶段：


1\>. httpclient请求， 正常tcp三次握手\+ 请求确认；
2\>. 客户端3s之后超时， 客户端发送FIN\+ACK 数据包（客户端标记连接已经被关闭）， 服务端确认收到客户端的FIN包；
3\>. 服务端5s尝试响应给客户端，最终会检测到客户端已经关闭而释放资源。


也就是说**客户端请求超时，只会影响客户端， 服务端还会继续处理并响应**， 这也是我们在服务端监控上看不到5xx报错的原因，可以通过在服务端设置： request\_time between (\-xx, 3s) 监测请求耗时占比。


正常的请求/响应读者可以参考下图：


![](https://files.mdnice.com/user/4236/006c21d5-47e0-41b5-90ef-b0d6c613c18c.png)


## 4\. 服务端能感知到客户端请求超时吗 ？


客户端请求超时， 默认情况下服务端都是继续执行之后响应；


**服务器是具备感知客户端请求取消的能力的。**


C\# 是通过`CancellationToken`，感知客户端取消，之后服务端可以做一些逻辑，比如记录客户端请求超时（常规实践是记录408响应码）



```


|  | // 在控制器/服务获取到当前请求的上下文，通过token感知到客户端取消， |
| --- | --- |
|  | var cancellationToken = httpContext.RequestAborted; |
|  | await LongLoop(cancellationToken); |
|  |  |
|  |  |
|  | public Task LongLoop(CancellationToken  token) |
|  | { |
|  | while(true) |
|  | { |
|  | if  (token.IsCancellationRequested == true) |
|  | { |
|  | break; |
|  | } |
|  | //---  长耗时循环 |
|  | } |
|  |  |
|  | return Task.CompletedTask; |
|  | } |


```

golang 是通过request.Context获取客户端取消信号，内核类似于C\#



```


|  | func getHello(w http.ResponseWriter, r *http.Request) { |
| --- | --- |
|  | ctx := r.Context() |
|  | select { |
|  | case <-ctx.Done(): |
|  | // 如果请求已取消或超时，这里会被触发 |
|  | err := ctx.Err() |
|  | fmt.Println("Request cancelled:", err) |
|  | return |
|  | case <-time.After(5 * time.Second): |
|  | io.WriteString(w, "Hello, HTTP!\n") |
|  | return |
|  | } |
|  | } |


```



---


本文记录了httpclient客户端超时在双端的现象， 服务端会继续响应，在服务端可能检测不到客户端认定的报错， 经验无他，唯手熟尔。


 本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
