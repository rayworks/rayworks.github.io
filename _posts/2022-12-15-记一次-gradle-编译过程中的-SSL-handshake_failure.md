---
layout: post
title: 记一次 gradle 编译过程中的 SSL handshake_failure
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

## 1. 背景：Android Studio中 Gradle 同步拉取library信息失败

```
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] Caused by: javax.net.ssl.SSLHandshakeException: Received fatal alert: handshake_failure
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.conn.ssl.SSLConnectionSocketFactory.createLayeredSocket(SSLConnectionSocketFactory.java:436)
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.conn.ssl.SSLConnectionSocketFactory.connectSocket(SSLConnectionSocketFactory.java:384)
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:142)
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.connect(PoolingHttpClientConnectionManager.java:374)
2022-12-14T17:13:14.916+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.execchain.MainClientExec.establishRoute(MainClientExec.java:393)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:236)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:186)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:89)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:83)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.gradle.internal.resource.transport.http.HttpClientHelper.performHttpRequest(HttpClientHelper.java:141)
2022-12-14T17:13:14.917+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.gradle.internal.resource.transport.http.HttpClientHelper.performHttpRequest(HttpClientHelper.java:121)
2022-12-14T17:13:14.918+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.gradle.internal.resource.transport.http.HttpClientHelper.executeGetOrHead(HttpClientHelper.java:106)
2022-12-14T17:13:14.918+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	at org.gradle.internal.resource.transport.http.HttpClientHelper.performRequest(HttpClientHelper.java:97)
2022-12-14T17:13:14.919+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 	... 50 more


Daemon worker: released lock on root.1
2022-12-14T17:13:14.860+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 
2022-12-14T17:13:14.862+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] FAILURE: Build failed with an exception.
2022-12-14T17:13:14.863+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 
2022-12-14T17:13:14.863+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * What went wrong:
2022-12-14T17:13:14.864+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] Could not determine the dependencies of task ':app:compileJunoUatDebugKotlin'.
2022-12-14T17:13:14.867+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] > Could not resolve all files for configuration ':app:googleplayUatDebugRuntimeClasspath'.
2022-12-14T17:13:14.867+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]    > Could not download base-1.1.2-SNAPSHOT.aar (com.your.pkg:base:1.1.2-SNAPSHOT:20210628.100523-1)
2022-12-14T17:13:14.867+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]       > Could not get resource 'https://nexus3.***.io/repository/maven-mobile-snapshots/com/your/pkg/base/1.1.2-SNAPSHOT/base-1.1.2-20210628.100523-1.aar'.
2022-12-14T17:13:14.868+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]          > Could not HEAD 'https://nexus3.***.io/repository/maven-mobile-snapshots/com/your/pkg/base/1.1.2-SNAPSHOT/base-1.1.2-20210628.100523-1.aar'.
2022-12-14T17:13:14.868+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]             > Received fatal alert: handshake_failure
2022-12-14T17:13:14.868+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] 
2022-12-14T17:13:14.868+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * Try:
2022-12-14T17:13:14.868+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter]  Run with --scan to get full insights.
```

## 2. 查看nexus服务器支持的tls 版本以及客户端的设置

经过在网上搜索，有一些线索。Https握手阶段失败，最直接的想法💡是 [客户端与服务器 TLS 版本协商失败](https://stackoverflow.com/questions/50824789/why-am-i-getting-received-fatal-alert-protocol-version-or-peer-not-authentic)。重新看了下客户端JDK的版本已经是JDK11，再 [分析](https://www.ssllabs.com/ssltest/analyze.html) 下 nexus 服务器上支持的协议，看到并不存在它所提到的问题。

## 3. 工具分析

通过 openssl 命令 `openssl s_client -connect nexus3.***.io:443 [-servername nexus3.***.io]` （带或不带 参数`servername`）的结果来看, 服务器端是否支持 [SNI](https://help.aliyun.com/document_detail/40519.html)，即它存在多台虚拟主机，可以根据客户端请求中不同的host，将请求分发给不同的域名（虚拟主机）来处理。

注意到 JDK 1.8中 参数 `jsse.enableSNIExtension` 默认值为 `true`。
~~于是在AS 项目中附加了参数 `-Djsse.enableSNIExtension=true` 到 `org.gradle.jvmargs`中。操作完成后重试，发现本地编译偶有成功，但问题依旧存在。~~

>可以使用openssl s_client和不使用-servername选项：
># without SNI
>$ openssl s_client -connect host:port
># use SNI
>$ openssl s_client -connect host:port -servername host
>如果你获得两个同名的不同证书，则表示支持并正确配置了 SNI。
>但是，如果返回的证书中的输出不同，或者没有SNI的调用无法建立SSL连接，则表明需要SNI但没有正确配置。解决此问题可能需要切换到专用 IP 地址。

此时发现 不带 -servername 的命令返回的结果确实是有问题的：
```
> openssl s_client -connect nexus3.***.io:443 -tlsextdebug

CONNECTED(00000006)
140704677139648:error:1404B410:SSL routines:ST_CONNECT:sslv3 alert handshake failure:/AppleInternal/Library/BuildRoots/810eba08-405a-11ed-86e9-6af958a02716/Library/Caches/com.apple.xbs/Sources/libressl/libressl-3.3/ssl/tls13_lib.c:129:SSL alert number 40
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 7 bytes and written 287 bytes
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : 0000
    Session-ID:
    Session-ID-ctx:
    Master-Key:
    Start Time: 1671086879
    Timeout   : 7200 (sec)
    Verify return code: 0 
```

## 4. 和SRE一起解决问题

于是找到公司的SRE同事一起来看，他建议断开cloud flare，设置本地host为固定IP地址后再尝试。果然Jenkins上的编译成功了。回头再看看服务器 不带 -servername 的返回，证书信息也有了。此刻真相大白了：上了cloud flare后 SNI的配置出现了问题！

## 5. 番外: regression ?!

 又有一天看，其它业务组的同事也**报出了相同的问题导致Jenkins上打包📦 出错**。在升级 gradle 版本到6.8后，发现除了上面的错误外，还额外有一条错误信息是
```
[org.gradle.internal.buildevents.BuildExceptionReporter] > 
The server may not support the client's requested TLS protocol versions: (TLSv1.2, TLSv1.3). 
You may need to configure the client to allow other protocols to be used. 
See: https://docs.gradle.org/6.8/userguide/build_environment.html#gradle_system_properties
```
这无疑之中让我想到除了在Jenkins 主机上抓网络数据包外，是不是还有其它办法可以看到更多`HTTPS` 握手阶段出错的更多细节呢？果然万能的SO给出类似问题的了一条很赞的回复[^1], 在JVM中设置参数[-Djavax.net.debug=all](https://docs.oracle.com/javase/6/docs/technotes/guides/security/jsse/ReadDebug.html) ，就可以调试 `HTTP SSL/TLS` 连接了。

在单独指定tls 版本为 `TLSv1.3`后，再看Jenkins上详细的log 输出:
```
The server does not support the client's requested TLS protocol versions: (TLSv1.3). You may need to configure the client to allow other protocols to be used. See: https://docs.gradle.org/6.8/userguide/build_environment.html#gradle_system_properties
Received fatal alert: protocol_version
......
 [ERROR] [system.err] javax.net.ssl|DEBUG|24|Build operations Thread 2|2023-02-09 10:41:02.495 UTC|SSLSocketInputRecord.java:213|READ: TLSv1.2 alert, length = 2
 [ERROR] [system.err] javax.net.ssl|DEBUG|24|Build operations Thread 2|2023-02-09 10:41:02.495 UTC|SSLSocketInputRecord.java:458|Raw read (
 [ERROR] [system.err]   0000: 02 46                                              .F
 [ERROR] [system.err] )
 [ERROR] [system.err] javax.net.ssl|DEBUG|24|Build operations Thread 2|2023-02-09 10:41:02.495 UTC|SSLSocketInputRecord.java:249|READ: TLSv1.2 alert, length = 2
 [ERROR] [system.err] javax.net.ssl|DEBUG|24|Build operations Thread 2|2023-02-09 10:41:02.496 UTC|Alert.java:232|Received alert message (
 [ERROR] [system.err] "Alert": {
 [ERROR] [system.err]   "level"      : "fatal",
 [ERROR] [system.err]   "description": "protocol_version"
 [ERROR] [system.err] }
 [ERROR] [system.err] )
 [ERROR] [system.err] javax.net.ssl|ERROR|24|Build operations Thread 2|2023-02-09 10:41:02.497 UTC|TransportContext.java:313|Fatal (PROTOCOL_VERSION): 
 Received fatal alert: protocol_version
```
🆗 至此为止看判断是与服务端协商支持的tls 版本不一致导致的问题，将客户端的tls 版本设置为`TLSv1.2`后，再次尝试编译，果然就成功了🚀。

## 6. Ref

* [不同版本JDK默认的TLS](https://stackoverflow.com/questions/21245796/javax-net-ssl-sslhandshakeexception-remote-host-closed-connection-during-handsh)

* [Nginx 同一个IP上配置多个HTTPS主机](http://www.ttlsa.com/web/multiple-https-host-nginx-with-a-ip-configuration/?spm=a2c4g.11186623.0.0.56581aafh9FPcP)

* [cURL and the TLS SNI extension](http://www.sigexec.com/posts/curl-and-the-tls-sni-extension/)

* [jsse.enableSNIExtension](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html)

* [如何修复“SSL Handshake Failed”错误](https://www.jiyik.com/tm/xwzj/network_287.html)

[^1]: [handshake_failure SO 链接](https://stackoverflow.com/questions/6353849/received-fatal-alert-handshake-failure-through-sslhandshakeexception)
