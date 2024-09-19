# 背景

今天收到一个线上告警，是一个没见过的报错：

`org.springframework.web.HttpMediaTypeNotAcceptableException: Could not find acceptable representation`

这个错误翻译过来叫：`无法找到可接受的代理`

请求返回码是：`406`

# 分析

我们代码里面没有定义`406`这个返回码，所以可以推测这个错误码是spring或其他自检返回的。

检查了业务日志发现没有什么异常；

## 报错的请求日志

```
Request processed:scheme=http,host=svc-for-hktw.internal.shopline,port=26080,method=GET,headerHost=svc-for-hktw.internal.shopline, uri=/api/posts/app-proxy/live/viewer/unauthorized/post/sales/frontLive, requestHeader:{"x-request-id":"0789540b1a795e5c0ae2a87fd15e6093","trace_id":"kbvsqsyrjhhnh7xe8twyiwqfocj9gwkr","trusted-by-alb":"shopline-cloudfront","x-forwarded-proto":"http","x-forwarded-port":"26080","x-forwarded-ssl":"on","x-ingress-class":"shopline","accept":"text/html,text/plain;q=0.9","via":"1.1 [14d757a67b913f1bc93427e69819362c.cloudfront.net](http://14d757a67b913f1bc93427e69819362c.cloudfront.net/) (CloudFront)","x-real-ip":"172.31.162.153","x-sent-from":"nginx-ingress-controller","x-original-forwarded-for":"54.162.129.185, 54.162.129.185, 172.27.53.97, 172.24.54.87","x-amzn-trace-id":"Self=1-65dd6af1-74fc92643c11a18c661d649a;Root=1-65dd6af1-30db61374b04a3f838fc2675","x-forwarded-host":"svc-for-hktw.internal.shopline","host":"svc-for-hktw.internal.shopline","x-scheme":"http","x_server_name":"[default-server.shoplineapp.com](http://default-server.shoplineapp.com/)","x-amz-cf-id":"s0MrvM2F5K0fh-j-4Xoo8NlmDpi_bP99i0GF602L2a6qVjRvSKUXaQ==","user-agent":"Mozilla/5.0 (compatible; proximic; +https://www.comscore.com/Web-Crawler)"}, requestBody=, requestParams=[customer_id=&merchant_id=619c964490f7b4001667d80f&path_prefix=/apps/front-live&timestamp=1709009649&sign=304550f92874ca32f7dd231f40f0c4983e30b8170df8f217e6762f03e9d8bb7f], respStatus=406, responseData=, usedTime=8
```

## 报错信息如下

异常已经被GlobalExceptionHandler抓住了

<img width="1319" alt="Untitled" src="https://github.com/user-attachments/assets/acd02eab-9159-4357-8c18-94434e4b7701">


但是GlobalExceptionHandler处理也报错了

<img width="1317" alt="Untitled" src="https://github.com/user-attachments/assets/0af9688b-382f-49e2-ad04-a613850ae275">


## 正常的请求日志

```
Request processed:scheme=http,host=svc-for-hktw.internal.shopline,port=26080,method=GET,headerHost=svc-for-hktw.internal.shopline, uri=/api/posts/app-proxy/live/viewer/unauthorized/post/sales/frontLive, requestHeader:{"referer":"https://www.babyshop4you.net/categories/26022024-%E9%9F%93%E5%9C%8B%E7%9B%B4%E6%92%AD%F0%9F%87%B0%F0%9F%87%B7?page=3&sort_by=&order_by=&limit=24","sec-fetch-site":"same-origin","trusted-by-alb":"shopline-cloudfront","x-forwarded-port":"26080","via":"2.0 22bd4d630b6e92aa10d682cdcf897598.cloudfront.net (CloudFront)","x-sent-from":"nginx-ingress-controller","sec-ch-ua-mobile":"?0","x-forwarded-host":"svc-for-hktw.internal.shopline","host":"svc-for-hktw.internal.shopline","x_server_name":"default-server.shoplineapp.com","x-request-id":"1ef9367006a3e3924715cee3cb9e5ddb","sec-fetch-mode":"cors","trace_id":"1kjd7vcybhdcatlob1q6ldbyygnx869y","x-forwarded-proto":"http","accept-language":"en-US,en;q=0.9","x-forwarded-ssl":"on","x-ingress-class":"shopline","accept":"*/*","x-real-ip":"172.31.161.21","sec-ch-ua":"\"Not A(Brand\";v=\"99\", \"Google Chrome\";v=\"121\", \"Chromium\";v=\"121\"","x-original-forwarded-for":"137.189.140.139, 137.189.140.139, 172.27.25.177, 172.24.124.98","x-amzn-trace-id":"Self=1-65ddacaa-2aa8cdea228cf54d12a8075b;Root=1-65ddacaa-2aec7fc02f8f9b3d61e2d33d","sec-ch-ua-platform":"\"Windows\"","x-scheme":"http","x-amz-cf-id":"gjyuea9-ljB0wWPnRFD9urP9wPXzsdQ4oD9EmcBXYGZPJAyqSbYaAA==","user-agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36","sec-fetch-dest":"empty"}, requestBody=, requestParams=[customer_id=623dc6317b3a44000c23794b&merchant_id=619c964490f7b4001667d80f&path_prefix=/apps/front-live&timestamp=1709026474&sign=17b397aaa55aaeaebc290b14b2cea5d85e73cb6abe293c108bc823b9d2d8677f], respStatus=200, responseData={"code":"SUCCESS","msg":"success","data":{"salesId":420068,"postSaleStatus":2,"liveViewerSdk":"FACEBOOK"},"success":true,"trace_id":"0123b8b0366be5ac3c0371d7973308ea"}, usedTime=6
```

# 结论

本次报错主要问题是request header里的`accept`字段，`accept`这个字段的值为`text/html,text/plain;q=0.9`，spring会尝试将response转换为html格式，但是我们返回的是object，然后就报错了。

报错之后被**GlobalExceptionHandler**抓住了，但是**GlobalExceptionHandler**返回的也是

`ApiResponse.fail` ，spring又会尝试将response转换为html格式，所以又报错。

进一步分析发现`user-agent` 值为`Mozilla/5.0 (compatible; proximic; +https://www.comscore.com/Web-Crawler)` 网址指向的是一个爬虫网站，所以请求应该是来自于爬虫。

# 扩展

> 出现这个异常主要是客户端请求期望响应的媒体类型与服务器响应的媒体类型不一致造成的。例如客户端希望返回的媒体类型是json对象（application/json），服务器返回的媒体类型是一个普通的json字符串（text/plain）;又或者是客户端希望返回的是html页面，服务器返回的却是json对象。
> 

基于springboot，使用 `@RestController` 和 `@PostMapping` 注解

这样 web 层返回结果时，直接 `return Object`，由 spring 将 `Object` 转化为 json 返回给前端