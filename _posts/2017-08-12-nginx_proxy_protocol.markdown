---
layout: post 
title:  "elb + nginx proxy protocol"
date:   2017-08-12 22:54:00 +0800
excerpt_separator: <!--abstract-->
tags: nginx proxy_protocol aws elb
---

同事最近写了一款小工具，为了知道为什么要这么搞，有了以下的学习结论。<!--abstract-->

elb是aws提供的一种四层负载均衡解决方案。与LVS不同，可以通过ELB配置页面，设置前置HTTPS，从而后端nginx无需添加HTTPS相关授权配置。

elb的默认情况下，后端nginx看到的请求源，是ELB负载均衡集群的IP。

如果后端nginx希望获取真实请求源的IP地址，则需要通过在对应端口，开启nginx上的proxy protocol，也就是代理协议。

elb集群会将真实请求源IP，放在proxy protocol包头，传递给后端nginx，从而使后端服务可以通过 `$proxy_protocol_addr` 获取请求源IP。

但是nginx开启代理协议后，在修改nginx配置时，nginx本机无法通过curl命令请求本地提供的接口(HTTP协议)，需用proxy protocol与之交互(相当于使用proxy protocol协议包裹HTTP包)。

代理协议行以回车符和换行符 ("\r\n") 结束，且具有以下形式：

    PROXY_STRING + single space + INET_PROTOCOL + single space + CLIENT_IP + single space + PROXY_IP + single space + CLIENT_PORT +     single space + PROXY_PORT + "\r\n"
    实例：
    PROXY TCP4 198.51.100.22 203.0.113.7 35646 80\r\n

当nginx启用了代理协议，$proxy_protocol_addr变量将是真实的客户端IP。



***


参考资料
1. [using-proxy-protocol-nginx](https://chrislea.com/2014/03/20/using-proxy-protocol-nginx/)
2. [AWS ELB nginx 启用代理协议](http://www.ttlsa.com/nginx/aws-elb-nginx-enable-proxy-protocol/)