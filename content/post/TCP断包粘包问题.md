+++
date = "2016-10-26T10:32:46+08:00"
draft = false
title = "TCP粘包问题：分包"

tags = ["network"]
+++

Update 2017-01-17
---

From Muduo：

TCP 是**“字节流”**协议，其本身没有“消息包”的概念，因此“粘包问题”是个伪命题。但对利用 TCP 进行通信的应用层程序来说，分包是其基本需求。

分包指的是在发送一个消息（message）或者一帧（frame）数据时，通过一定的处理，令接收方能从字节流中识别并截取（还原）出一个个消息包。

对于短连接的 TCP 服务，分包不是问题。只要发送方主动关闭连接，就表示一条消息发送完毕，接收方 read() 返回0，从而得知消息结尾。

对于长连接的 TCP 服务，分包有4种方法：

1. 消息长度固定（亦即是提前确定包长度，适合定长消息包）；
2. 使用特殊的字符或字符串作为消息的边界，例如 HTTP 协议的 headers 以“\r\n”为字段的分隔符；
3. 在每条消息的头部加一个长度字段，最常见的做法；
4. 利用消息本身的格式来分包，例如 XML 格式的消息中<root>...</root>的配对，或者json格式中的{...}的配对。解析这种消息格式通常会用到状态机。

粘包问题
---
一个完整的消息可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送。粘包是指发送方发送的若干包数据到接收方接收时粘成一包，从接收缓冲区看，后一包数据的头紧接着前一包数据的尾。

粘包问题是由 TCP 是面向字节流协议因此没有消息边界所引起的。而 UDP 是面向数据报的协议，所以不存在拆包粘包问题。

存在以下特殊情况：

1. 如果发送数据无结构，如文件传输，这样发送方只管发送，接收方只管接收存储就 ok，不用考虑粘包；
2. 如果利用 TCP 短连接时，不会出现粘包问题；
3. 当发送数据**存在一定结构，并且需要维护长连接时**，则需要考虑粘包问题；

问题原因
---
出现拆包粘包现象的原因既可能由发送方造成，也可能由接收方造成:

1. 要发送的数据大于TCP发送缓冲区剩余空间大小，发生拆包；
2. 待发送数据大于MSS（最大报文长度），TCP在传输前进行拆包；
3. 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去，造成粘包;
4. 接收方没能及时地接收缓冲区的数据，造成粘包;

解决方法
---
解决粘包的方法，是由应用层进行**分包处理**，本质上就是由**应用层**来维护消息和消息的边界（即定义自己的会话层和表示层协议）。

本文处理办法：

1. 发送方在每次发送消息时将数据报长度写入一个int32作为包头一并发送出去, 称之为Encode；
2. 接受方则先读取一个int32的长度的消息长度信息, 再根据长度读取相应长的byte数据, 称之为Decode；
 
        //codec.go
    
        package codec
        
        import (
            "bufio"
            "bytes"
            "encoding/binary"
        )
        
        func Encode(message string) ([]byte, error) {
            // 读取消息的长度
            var length int32 = int32(len(message))
            var pkg *bytes.Buffer = new(bytes.Buffer)
            // 写入消息头
            err := binary.Write(pkg, binary.LittleEndian, length)
            if err != nil {
                return nil, err
            }
            // 写入消息实体
            err = binary.Write(pkg, binary.LittleEndian, []byte(message))
            if err != nil {
                return nil, err
            }
        
            return pkg.Bytes(), nil
        }
        
        func Decode(reader *bufio.Reader) (string, error) {
            // 读取消息的长度
            lengthByte, _ := reader.Peek(4)
            lengthBuff := bytes.NewBuffer(lengthByte)
            var length int32
            err := binary.Read(lengthBuff, binary.LittleEndian, &length)
            if err != nil {
                return "", err
            }
            if int32(reader.Buffered()) < length+4 {
                return "", err
            }
        
            // 读取消息真正的内容
            pack := make([]byte, int(4+length))
            _, err = reader.Read(pack)
            if err != nil {
                return "", err
            }
            return string(pack[4:]), nil
        }