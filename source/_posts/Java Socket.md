---
title: Java Socket
date: 2017/10/5 18:4:0
tags: [Java,Java Socket]
categories: [Java Socket]
---
打算将 Socket 编程再复习下，为后续学习 netty 做准备。		
<!--more-->
		
# 知识准备
学习 Socket 编程，一些前置知识必不可少。

- OSI 参考模型以及 TCP/IP 协议栈。		

- 网络编程三要素：协议、IP 和端口号。		

- 端口号是正在运行的程序的标识；有效的端口号范围：0 到 65535，其中 0 到 1024 为系统使用或保留。  

- TCP/IP 是对一组协议的统称，具体每一层都有很多协议。传输层的协议主要关注 TCP 和 UDP。

- UDP 的特点是无连接、速度快、不可靠，需要将数据打包，有大小限制。

- TCP 的特点是有连接（三次握手、四次挥手），速度相比 UDP 要慢，但要可靠，数据无限制。	

- Socket 编程，即为网络编程，也称为套接字编程。Socket 是网络上具有唯一标识的 IP 地址和端口号组合在一起构成。Socket 通信的两端都有 Socket，网络通信即为 Socket 间的通信，数据在两个 Socket 间在某种协议下通过 IO 流传输。  

# API 学习
关于 Java Socket 的源码都在 net 包下，其中有几个比较重要的类 `InetAddress` 和 `URL` 等等。		
```java
package com.nekolr;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.InetAddress;
import java.net.URL;

/**
 * @author nekolr
 */
public class APIDemo {

    public static void main(String[] args) throws Exception {

        /**
         * InetAddress 类
         * 用于标识网络上的硬件资源，标识网络层 IP 协议地址
         */

        // 获取本机 InetAddress 实例
        InetAddress address = InetAddress.getLocalHost();
        // 获取主机名
        String hostName = address.getHostName();
        // 获取主机 IP
        String ip = address.getHostAddress();

        /*
        *    byte 与十进制转换：192(10) -> -64(byte)
        *
        *    192(10) -> 11000000(2)
        *    由于 byte 范围为-128 ~ 127，对应二进制表示 11111111 ~ 01111111，11000000 用 byte 表示即为-64
        *
        * */

        // 获取主机 IP，如果是 IPV4，则是一个长度为 4 的 byte 数组
        byte []bytes = address.getAddress();
        // 根据主机名或 IP 地址字符串获取 InetAddress 实例
        InetAddress address1 = InetAddress.getByName("avalon");
        InetAddress address2 = InetAddress.getByName("192.168.229.1");
        // 根据 IP 获取 InetAddress 实例
        InetAddress address3 = InetAddress.getByAddress(bytes);


        /**
         * URL 类
         * 统一资源定位符，标识网络上的某个资源的地址
         */

        URL baidu = new URL("http://www.baidu.com");

        URL url = new URL(baidu, "/index.html?username=nekolr#test");
        // 获取协议，此处 http
        url.getProtocol();
        // 获取主机名，此处 www.baidu.com
        url.getHost();
        // 获取端口号，此处 -1
        url.getPort();
        // 获取资源地址，此处 /index.html
        url.getPath();
        // 获取资源名称，此处 /index.html?username=nekolr
        url.getFile();
        // 获取锚点，此处 test
        url.getRef();
        // 获取参数，此处 username=nekolr
        url.getQuery();


        /**
         * 使用 URL 打开资源
         */

        // 打开获取资源的输入流
        InputStream is = baidu.openStream();
        // 字节流转字符流
        InputStreamReader isr = new InputStreamReader(is, "utf-8");
        // 使用缓存
        BufferedReader br = new BufferedReader(isr);
        String line = br.readLine();
        while (line!=null){
            System.out.println(line);
            line = br.readLine();
        }
        br.close();
        isr.close();
        is.close();

        // 使用 URL 打开资源，使用 JDK1.7 新增的 AutoCloseable 接口自动关闭资源
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(baidu.openStream(), "utf-8"))) {
            String data;
            while((data = reader.readLine())!=null){
                System.out.println(data);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

# 简单 Socket 通信练习
Socket 通信基于 TCP 和 UDP 协议，针对这两个协议有不同的写法。		
![socket 通信模型 ](https://img.nekolr.com/images/2018/04/14/gm.jpg)
		
```java
package com.nekolr.socket;

import com.google.common.util.concurrent.ThreadFactoryBuilder;

import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.*;

/**
 * @author nekolr
 */
public class TcpSocket {

    static class TcpServer implements Runnable {

        private int port = 8888;

        @Override
        public void run() {
            try (
                    // 创建服务端 Socket，监听端口
                    ServerSocket serverSocket = new ServerSocket(port)
            ) {
                // 打开监听，等待客户端的连接（在连接到来之前一直阻塞）
                Socket socket = serverSocket.accept();

                try (
                        // 获取输入流（获取客户端的消息）
                        InputStream is = socket.getInputStream();
                        InputStreamReader isr = new InputStreamReader(is, "utf-8");
                        BufferedReader reader = new BufferedReader(isr)
                ) {
                    String line;
                    while ((line = reader.readLine()) != null) {
                        System.out.println("客户端" + socket.getInetAddress().getHostAddress() + "发送消息：" + line);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }

            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class TcpClient implements Runnable {

        private String host = "localhost";

        private int port = 8888;

        @Override
        public void run() {
            try (
                    // 创建客户端 Socket，指定主机名和端口号
                    Socket socket = new Socket(host, port)
            ) {
                try (
                        // 获取输出流（向服务端发送消息）
                        OutputStream os = socket.getOutputStream();
                        OutputStreamWriter osw = new OutputStreamWriter(os, "utf-8");
                        BufferedWriter writer = new BufferedWriter(osw)
                ) {
                    int i = 0;
                    while (true) {
                        i++;
                        writer.write("hello" + i);
                        writer.newLine();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }

            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        TcpServer tcpServer = new TcpServer();
        TcpClient tcpClient = new TcpClient();

        ThreadFactory factory = new ThreadFactoryBuilder().setNameFormat("thread-pool").build();
        ExecutorService executor = new ThreadPoolExecutor(2, 2,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(1024), factory, new ThreadPoolExecutor.AbortPolicy());

        executor.execute(tcpServer);
        executor.execute(tcpClient);
    }
}

```
		
```java
package com.nekolr.socket;

import com.google.common.util.concurrent.ThreadFactoryBuilder;

import java.io.IOException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.util.concurrent.*;

/**
 * @author nekolr
 */
public class UdpSocket {

    static class UdpServer implements Runnable {
        /**
         * 客户端 port
         */
        private int clientPort = 10088;

        /**
         * 服务端 port
         */
        private int serverPort = 10086;

        private String host = "localhost";

        @Override
        public void run() {
            try (
                    // 创建服务端 Socket
                    DatagramSocket socket = new DatagramSocket(serverPort);
            ) {

                // 创建数据报用于发送消息
                byte[] bytes = "我是服务端，消息为：i'm server".getBytes();
                DatagramPacket sendPacket = new DatagramPacket(bytes, bytes.length, InetAddress.getByName(host), clientPort);
                // 发送消息
                socket.send(sendPacket);

                bytes = new byte[1024];
                DatagramPacket receivePacket = new DatagramPacket(bytes, bytes.length);
                // 接收消息（阻塞）
                socket.receive(receivePacket);

                System.out.println("服务端接收的消息为：" + new String(bytes, "utf-8"));
            } catch (SocketException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class UdpClient implements Runnable {
        /**
         * 客户端 port
         */
        private int clientPort = 10088;
        /**
         * 服务端 port
         */
        private int serverPort = 10086;

        private String host = "localhost";

        @Override
        public void run() {
            try (
                    // 创建客户端 Socket
                    DatagramSocket socket = new DatagramSocket(clientPort);
            ) {
                // 创建数据报用于发送消息
                byte[] bytes = "我是客户端，消息为：i'm client".getBytes();
                DatagramPacket sendPacket = new DatagramPacket(bytes, bytes.length, InetAddress.getByName(host), serverPort);
                // 发送消息
                socket.send(sendPacket);

                bytes = new byte[1024];
                DatagramPacket receivePacket = new DatagramPacket(bytes, bytes.length);
                // 接收消息（阻塞）
                socket.receive(receivePacket);

                System.out.println("客户端接收的消息为：" + new String(bytes, "utf-8"));

            } catch (SocketException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        UdpServer udpServer = new UdpServer();
        UdpClient udpClient = new UdpClient();

        ThreadFactory factory = new ThreadFactoryBuilder().setNameFormat("thread-pool").build();
        ExecutorService executor = new ThreadPoolExecutor(2, 2,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(1024), factory, new ThreadPoolExecutor.AbortPolicy());

        executor.execute(udpServer);
        executor.execute(udpClient);
    }
}

```