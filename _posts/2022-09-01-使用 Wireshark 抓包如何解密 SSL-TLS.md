---
layout:         post
title:          使用 Wireshark 抓包如何解密 SSL-TLS
subtitle:		HTTPS 也不能保证 100% 安全
date:           2022-09-01
author:         Luke
cover:          '/assets/img/post-decrypt_ssl.png'
tags:           AWS, ALB, health check, HTTP header, HSTS
---

# 简单的操作步骤
[Decrypting SSL/TLS with Wireshark](https://resources.infosecinstitute.com/topic/decrypting-ssl-tls-traffic-with-wireshark/)  
适用于 Firefox、chrome 等浏览器，并不一定适用于 curl, apache jmeter 等工具  

1. `Windows + I` to open system settings  
2. search "env", choose "system env variables"  
3. "Advanced" -- "Env Variables" - New  
4. `SSLKEYLOGFILE`, choose a path/file to save the secrets.  
![SSLKEYLOGFILE](/assets/img/SSLKEYLOGFILE.png)      
5. Wireshark -- Edit -- Preferances -- Protocol -- TLS -- `(Pre)-Master-Secret` log filename, choosethe same path/file as "SSLKEYLOGFILE"  
![Master-Secret](/assets/img/Master-Secret.png)    
6. if you have the Wireshark capture in another computer with above settings, just copy the .pcap file and the "SSLKEYLOGFILE" file to new computer, then go ahead to modify your new computer Wireshark TLS settings, you will able to decrypt the .pcap as well.  
7. you should only share the secrets file to the person when you MUST.  

# Wireshark 一些相关的 display filter
tls.handshake.type == 1  // Client Hello, check the SNI for HTTP Host  
tls.handshake.extensions_server_name == "www.amazonaws.cn"  // filter the specify SNI HTTP Host  
http2.header.unescaped == "POST"        // filter the HTTP request method, eg. GET, POST  
http2.header.value == "/upload/uploadPic"   // this is the URL for upload  
mime_multipart                            // when upload, the file may split into multiple   packets    
http2.streamid == 300      // the specify stream id  
