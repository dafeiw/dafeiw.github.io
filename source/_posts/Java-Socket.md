---
title: Java Socket
date: 2019-07-16 10:50:32
tags:
- Java
- Web
categories: Web
---

# 1. What is Socket

Two programs running over network use socket to communicate with each other. A socket is an endpoint of a network connection. Two applications can communicate with each other by sending and receiving streams over a connection. You need to know the IP address as well as the port number of the socket of the other application. In Java, Socket includes ServerSocket and Socket classes in java.net package. ServerSocket implements the server side. Socket implements client side.

ServerSocket is different from Socket. The role of a server socket is to wait for connection requests from clients. Once the server socket gets a connection request, it creates a Socket instance to handle the communication with the client.

On the client side, the client knows the hostname of the server and the port number on which the server is listening. The client also needs to identify itself to the server so it binds to a local port number. This is usually assigned by the system. 
<!--more-->
The server accepts the connection. Upon acceptance, the server gets a new socket and also has its remote endpoint set to the address and port of the client. It needs a new socket so that it can continue to listen to the original socket for connection request. 

On the client side, if the connection is accepted, a socket is successfully created and the client can use the socket to communicate with the server.

Every TCP connection can be uniquely identified by a combination of an IP address and a port number. 

If you are trying to the Web, the URL class and related classes (URLConnection, URLEncorder) are probably more appropriate than the socket classes. URLs use sockets as part of the underlying implementation. 

# 2. Socket

## 1) Create ServerSocket. 
ServerSocket has 5 constructors. The most convenient one is ServerSocket(int port). It only needs a port number.

## 2) Call accept() method
The accept method is a blocking method. This method will only return when there is a connection request and its return value is an instance of the Socket class.

## 3) Use the returned Socket

The returned socket is used to create Reader and Writer to receive and send data. 

Once you create/get an instance of the Socket class, you can use it to send and receive streams of bytes. 

```
public class Server {
    public static void main(String[] args) {
        try {
            ServerSocket server = new ServerSocket(8080);
            Socket socket = server.accept(); // block here
            BufferedReader is = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String line = is.readLine();
            System.out.println("Received from client: " + line);
            PrintWriter pw = new PrintWriter(socket.getOutputStream());
            pw.println("Received data: " + line);
            pw.flush();
            pw.close();
            is.close();
            socket.close();
            server.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class Client {
    public static void main(String[] args) {
        String msg = "Client Data";
        try {
            Socket socket = new Socket("127.0.0.1", 8080);
            PrintWriter pw = new PrintWriter(socket.getOutputStream());
            BufferedReader is = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            pw.println(msg);
            pw.flush();
            String line = is.readLine();
            System.out.println("Received from server: " + line);
            pw.close();
            is.close();
            socket.close();
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

# 3. NioSocket

ServerSocketChannel and SocketChannle replace ServerSocket and Socket.

In the standard IO API you work with byte streams and character streams. In NIO you work with channels and buffers. Data is always read from a channel into a buffer, or written from a buffer to a channel.

ServerSocketChanel is created by its static factory method open(). Each ServerSocketChanel has one ServerSocket which can be retrieved by socket() method. You don't use ServerSocket to listen for incoming requests. Instead, you use ServerSocket to bind a port number. You can use configureBlocking method to configure blocking mode. If it's non-blocking mode, you can call register method to register Selector.

Selector is created by its static factory method open(). Use register method to register selector to ServerSocketChannel or SocketChannel. After registration, you cna use select method to waiting for requests. The selector method accepts a long type parameter. If you pass 0 or no parameter, select method will block until connection. 

NioServer is created by the following steps:
1) Create ServerSocketChannel
2) Create Selector and register it on ServerSocketChannel
3) Invoke select method of Selector and waiting for connection
4) SelectionKey collection is returned upon connection
5) Use SelectionKey to get Channel and Selector and type of operation.

```
public class NioServer {

    public static void main(String[] args) throws IOException {
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.socket().bind(new InetSocketAddress((8080)));
        ssc.configureBlocking(false);
        Selector selector = Selector.open();
        ssc.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            if (selector.select(3000) == 0) {
                System.out.println("Timeout...");
                continue;
            }
            System.out.println("Process request...");
            Iterator<SelectionKey> keyIter = selector.selectedKeys().iterator();

            while (keyIter.hasNext()) {
                SelectionKey key = keyIter.next();
                if (key.isAcceptable()) {
                    // handle accept
                    SocketChannel sc = ((ServerSocketChannel)key.channel()).accept();
                    sc.configureBlocking(false);
                    sc.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocate(1024));
                }
                if (key.isReadable()) {
                        // handle
                }
                keyIter.remove();
            }

        }
    }
}
```


