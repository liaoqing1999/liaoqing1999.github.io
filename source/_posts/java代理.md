---
layout: post
title: java代理
date: 2020-10-26 09:43:34
tags: java
categories: java
---

# Java代理

最近在代码中遇到了需要后端调用第三方接口，但是由于公司电脑存在代理，所以需要在代码中设置代理，才能够调通。

## 使用HttpHost设置代理

HttpHost是org.apache.http.HttpHost下的包

当我使用HttpPost或其他请求方法时，可以使用以下代码设置代理。

```
		HttpPost httpPost = new HttpPost(url);
		//代理IP 端口号
      	HttpHost proxy = new HttpHost("127.0.0.1", 808);
        RequestConfig requestConfig = RequestConfig.custom()
                .setProxy(proxy)
                .setConnectTimeout(10000)
                .setSocketTimeout(10000)
                .setConnectionRequestTimeout(3000)
                .build();
        httpPost.setConfig(requestConfig);
        StringEntity entity = new StringEntity(jsonBody);
        httpPost.setHeader("Content-Type", "application/x-www-form-urlencoded;charset=utf-8");
        httpPost.setEntity(entity);
        CloseableHttpResponse response = httpClient.execute(httpPost);
        String jsonStr = EntityUtils.toString(response.getEntity());
```

## 使用System.setProperty设置全局代理

可以使用System.setProperty设置相关代理属性，全局有效

```
System.setProperty("proxyType", "4");
// 端口
System.setProperty("proxyPort", "808");
// IP
System.setProperty("proxyHost", "127.0.0.1");
System.setProperty("proxySet", "true");
// 用户名
System.setProperty("proxyUserName", username);
// 密码
System.setProperty("proxyPassword", password);


// 自动使用系统代理（IE）
 System.setProperty("java.net.useSystemProxies", "true");
```

## 使用Proxy类设置代理

```
URL urlClient = new URL(url);
Proxy proxy= new Proxy(Proxy.Type.HTTP,newInetSocketAddress("127.0.0.1",808)); 
URLConnection connection =url.openConnection(proxy);
// 设置通用的请求属性
httpsConn.setRequestProperty("accept", "*/*");
httpsConn.setRequestProperty("connection", "Keep-Alive");
httpsConn.setRequestProperty("user-agent",
"Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1;SV1)");
// 发送POST请求必须设置如下两行
httpsConn.setDoOutput(true);
httpsConn.setDoInput(true);
// 获取URLConnection对象对应的输出流
out = new PrintWriter(httpsConn.getOutputStream());
// 发送请求参数
out.print(param);
// flush输出流的缓冲
out.flush();
// 定义BufferedReader输入流来读取URL的响应
in = new BufferedReader(new InputStreamReader(httpsConn.getInputStream()));
```

## sdk中代理设置

一般sdk中做了对代理的封装，注意查找他们设置代理的方法