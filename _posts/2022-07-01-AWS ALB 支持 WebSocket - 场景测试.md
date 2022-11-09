---
layout:         post
title:          AWS ALB 支持 WebSocket - 场景测试
subtitle:		    实现 client、server 之间的 双向、实时通信
date:           2022-07-01
author:         Luke
cover:          '/assets/img/IMG_20220701-094059445.png'
tags:           AWS, ALB, WebSocket, ws
---


- ALB 支持 WebSocket，不需要为 ALB Listener, target group 做特殊配置；ALB Listener 选择 http or https，client 访问 ws:// 或者 wss:// 即可
- 需要注意 ALB idle timeout 60s，WebSocket client-server 之间交互必须在 60 秒内有 keepalive(ping-pong)，不然就会被 ALB disconnect
- WebSocket 是基于 TCP 的协议，通过 HTTP header 中携带 Connection: Upgrade, Upgrade: websocket 来升级 WebSocket，报文细节参考文章结尾的抓包。
- WebSocket 实现了浏览器与服务器全双工通信，能更好的节省服务器资源和带宽并达到实时通讯效果。

**环境准备：**

- curl 无法直接访问 ws:// 或者 wss://，需要比较长的一串指令才可以；如果需要简单测试，可以安装 npm, wscat
- [安装 npm 可以参考](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html)
- npm install -g wscat // wscat 是一个封装好的工具，用来测试 WebSocket 接口，方便实用
- 快速测试 wscat 是否工作正常，然后可以做一个 AMI 防身
    wscat -c wss://ws.postman-echo.com/raw  
        Connected (press CTRL+C to quit)

**测试 WebSocket，双向连接：**

- 使用 wscat 直接做 server 和 client 测试


        [ec2-user@elb-target-az1c-WebScoket ~]$ wscat -l 8123  // 作为 server，监听 WebSocket on port 8123  
        Listening on port 8123 (press CTRL+C to quit)  
        Client connected  
        < hello world // 接收到来自 client msg  
        > hi nice to meet U      // 主动给 client 发送 msg  
        Disconnected (code: 1005, reason: "")   // client 端按下 Ctrl C 中断连接

        [ec2-user@Bastion-ConsumerVPC-ZHY ~]$ wscat -c ws://172.31.46.123:8123 // 作为 client，访问 ws 资源  
        Connected (press CTRL+C to quit)  
        > hello world  
        < hi nice to meet U

- client 是 VPC 内 EC2，在 R53 创建了 private hosted zone，为 ALB 添加一条 alias 记录，就可以实现 VPC 内自定义域名访问了


        [ec2-user@elb-target-az1c-WebScoket ~]$ wscat -l 8123
        Listening on port 8123 (press CTRL+C to quit)
        Client connected
        < hello server, this is a connection via the ALB
        > hi client, yes ALB support WebSocket natively
        Disconnected (code: 1006, reason: "")        // 被 ALB 中断的 ws connection

        [ec2-user@Bastion-ConsumerVPC-ZHY ~]$ wscat -c ws://mynxalb.com:8123
        Connected (press CTRL+C to quit)
        > hello server, this is a connection via the ALB
        < hi client, yes ALB support WebSocket natively
        Disconnected (code: 1006, reason: "")

抓包结果：


    [ec2-user@Bastion-ConsumerVPC-ZHY ~]$ tshark -r websocket.pcap -V -O websocket,http
    Frame 1: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
    Ethernet II, Src: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0), Dst: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96)
    Internet Protocol Version 4, Src: 172.31.30.194 (172.31.30.194), Dst: 52.82.112.71 (52.82.112.71)
    Transmission Control Protocol, Src Port: 45364 (45364), Dst Port: 8123 (8123), Seq: 0, Len: 0

    Frame 2: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
    Ethernet II, Src: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96), Dst: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0)
    Internet Protocol Version 4, Src: 52.82.112.71 (52.82.112.71), Dst: 172.31.30.194 (172.31.30.194)
    Transmission Control Protocol, Src Port: 8123 (8123), Dst Port: 45364 (45364), Seq: 0, Ack: 1, Len: 0

    Frame 3: 66 bytes on wire (528 bits), 66 bytes captured (528 bits)
    Ethernet II, Src: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0), Dst: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96)
    Internet Protocol Version 4, Src: 172.31.30.194 (172.31.30.194), Dst: 52.82.112.71 (52.82.112.71)
    Transmission Control Protocol, Src Port: 45364 (45364), Dst Port: 8123 (8123), Seq: 1, Ack: 1, Len: 0

    Frame 4: 291 bytes on wire (2328 bits), 291 bytes captured (2328 bits)
    Ethernet II, Src: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0), Dst: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96)
    Internet Protocol Version 4, Src: 172.31.30.194 (172.31.30.194), Dst: 52.82.112.71 (52.82.112.71)
    Transmission Control Protocol, Src Port: 45364 (45364), Dst Port: 8123 (8123), Seq: 1, Ack: 1, Len: 225
    Hypertext Transfer Protocol
        GET / HTTP/1.1\r\n
            [Expert Info (Chat/Sequence): GET / HTTP/1.1\r\n]
                [Message: GET / HTTP/1.1\r\n]
                [Severity level: Chat]
                [Group: Sequence]
            Request Method: GET
            Request URI: /
            Request Version: HTTP/1.1
        Sec-WebSocket-Version: 13\r\n
        Sec-WebSocket-Key: MwRD/oe9OfmxI7F7bqdUaw==\r\n
        Connection: Upgrade\r\n
        Upgrade: websocket\r\n
        Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits\r\n
        Host: mynxalb.com:8123\r\n
        \r\n
        [Full request URI: http://mynxalb.com:8123/]
        [HTTP request 1/1]

    Frame 5: 66 bytes on wire (528 bits), 66 bytes captured (528 bits)
    Ethernet II, Src: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96), Dst: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0)
    Internet Protocol Version 4, Src: 52.82.112.71 (52.82.112.71), Dst: 172.31.30.194 (172.31.30.194)
    Transmission Control Protocol, Src Port: 8123 (8123), Dst Port: 45364 (45364), Seq: 1, Ack: 226, Len: 0

    Frame 6: 232 bytes on wire (1856 bits), 232 bytes captured (1856 bits)
    Ethernet II, Src: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96), Dst: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0)
    Internet Protocol Version 4, Src: 52.82.112.71 (52.82.112.71), Dst: 172.31.30.194 (172.31.30.194)
    Transmission Control Protocol, Src Port: 8123 (8123), Dst Port: 45364 (45364), Seq: 1, Ack: 226, Len: 166
    Hypertext Transfer Protocol
        HTTP/1.1 101 Switching Protocols\r\n
            [Expert Info (Chat/Sequence): HTTP/1.1 101 Switching Protocols\r\n]
                [Message: HTTP/1.1 101 Switching Protocols\r\n]
                [Severity level: Chat]
                [Group: Sequence]
            Request Version: HTTP/1.1
            Status Code: 101
            Response Phrase: Switching Protocols
        Date: Thu, 30 Jun 2022 08:18:45 GMT\r\n
        Connection: upgrade\r\n
        Upgrade: websocket\r\n
        Sec-WebSocket-Accept: LpQh8kaihmGtS0YZ6+blfDgly7g=\r\n
        \r\n
        [HTTP response 1/1]
        [Time since request: 0.013851000 seconds]
        [Request in frame: 4]

    Frame 7: 66 bytes on wire (528 bits), 66 bytes captured (528 bits)
    Ethernet II, Src: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0), Dst: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96)
    Internet Protocol Version 4, Src: 172.31.30.194 (172.31.30.194), Dst: 52.82.112.71 (52.82.112.71)
    Transmission Control Protocol, Src Port: 45364 (45364), Dst Port: 8123 (8123), Seq: 226, Ack: 167, Len: 0

    Frame 8: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
    Ethernet II, Src: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0), Dst: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96)
    Internet Protocol Version 4, Src: 172.31.30.194 (172.31.30.194), Dst: 52.82.112.71 (52.82.112.71)
    Transmission Control Protocol, Src Port: 45364 (45364), Dst Port: 8123 (8123), Seq: 226, Ack: 167, Len: 8
    WebSocket
        1... .... = Fin: True
        .000 .... = Reserved: 0x00
        .... 0001 = Opcode: Text (1)
        1... .... = Mask: True
        .000 0010 = Payload length: 2
        Masking-Key: 222c2f98
        Payload
            Text: 4a45
        Unmask Payload
            [Text unmask: hi]

    Frame 9: 66 bytes on wire (528 bits), 66 bytes captured (528 bits)
    Ethernet II, Src: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96), Dst: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0)
    Internet Protocol Version 4, Src: 52.82.112.71 (52.82.112.71), Dst: 172.31.30.194 (172.31.30.194)
    Transmission Control Protocol, Src Port: 8123 (8123), Dst Port: 45364 (45364), Seq: 167, Ack: 234, Len: 0

    Frame 10: 71 bytes on wire (568 bits), 71 bytes captured (568 bits)
    Ethernet II, Src: 06:06:53:d4:f0:96 (06:06:53:d4:f0:96), Dst: 06:b2:ad:cd:3e:e0 (06:b2:ad:cd:3e:e0)
    Internet Protocol Version 4, Src: 52.82.112.71 (52.82.112.71), Dst: 172.31.30.194 (172.31.30.194)
    Transmission Control Protocol, Src Port: 8123 (8123), Dst Port: 45364 (45364), Seq: 167, Ack: 234, Len: 5
    WebSocket
        1... .... = Fin: True
        .000 .... = Reserved: 0x00
        .... 0001 = Opcode: Text (1)
        0... .... = Mask: False
        .000 0011 = Payload length: 3
        Payload
            Text: yes