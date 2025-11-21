---
layout: post
title: "golang如何正确关闭tcp连接"
date: 2021-10-09
categories: [golang]
---

> 

## Read方法返回EOF错误

假设我们有两个demo程序——server和client。

client主动连接上server后不做任何操作，直接关闭net.conn对象。用于模拟主动关闭端。伪代码如下：
```go
conn, _ := net.Dial("tcp", "127.0.0.1:8081")
conn.Close()

```
server在accept新连接后，在新连接的处理函数中调用Read方法，Read返回io.EOF后不调用Close方法，直接退出处理函数，释放连接对象。伪代码如下：
```go
func handleConn(conn net.Conn) {
    buf := make([]byte, 1024)
    n, err := conn.Read(buf)
    log.Println(n, err)
    //conn.Close()
}
```
启动server后，再启动client，server打印出0 EOF。

用netstat查看连接情况：
```
$netstat -an | grep 8081

tcp4       0      0  127.0.0.1.8081         127.0.0.1.62871        CLOSE_WAIT
tcp4       0      0  127.0.0.1.62871        127.0.0.1.8081         FIN_WAIT_2
tcp46      0      0  *.8081                 *.*                    LISTEN
```
* client处于FIN_WAIT_2状态，说明client发送了FIN，并收到了对应的ACK。

* server处于CLOSE_WAIT状态，说明server收到了FIN，并发送了对应的ACK。

用wireshark抓包：
再测试一遍，发现client发送了FIN，server回复了对应的ACK。但是server并没有发送FIN。与netstat显示的状态相符合。
修改server代码，在Read返回EOF后，调用conn.Close()
重新测试，再使用netstat和wireshark分析，发现server也发送了FIN，两端都正常关闭。

## Write方法返回broken pipe错误

修改server代码。伪代码如下：
```go
	buf := make([]byte, 1024)
	time.Sleep(5 * time.Second)
	n, err := conn.Write(buf)
	log.Println(n, err)
	time.Sleep(5 * time.Second)
	n, err = conn.Write(buf)
	log.Println(n, err)
```
server输出如下：
```
2019/04/17 16:08:17 1024 <nil>
2019/04/17 16:08:18 0 write tcp 127.0.0.1:8081->127.0.0.1:64124: write: broken pipe
```
server的第一次Sleep 5秒是为了确保在第一次Write之前client已关闭连接。

用netstat观察：

我们发现在5秒内，server处于CLOSE_WAIT状态，client处于FIN_WAIT_2状态。

5秒之后，两端都进入完全关闭状态。

用wireshark抓包：

发现5秒后，server向client发送第一次1024字节数据后，client向server回复了RST包。

10秒后，server并不会再发送第二次的1024字节数据。

server的第二次Sleep 5秒是为了确保在第一次Write之后，server接收到了RST包。如果去掉第二次的Sleep，可能出现server连续发送两次数据给client，client回复两次RST给server。

## 本端关闭后，本端继续调用Read或Write或Close方法

```go
127.0.0.1:63482->127.0.0.1:8081: use of closed network connection
127.0.0.1:63448->127.0.0.1:8081: use of closed network connection
```
会一直报如下错误，这是由fd_mutex.go中的mutexClosed标志决定的，当文件描述符被关闭后，该标志会被设置，之后所有io操作都会返回错误。


## 结论

* Read方法返回EOF错误，表示本端感知到对端已经关闭连接（本端已接收到对端发送的FIN）。此后如果本端不调用Close方法，只释放本端的连接对象，则连接处于非完全关闭状态（CLOSE_WAIT）。即文件描述符发生泄漏。

* Write方法返回broken pipe错误，表示本端感知到对端已经关闭连接（本端已接收到对端发送的RST）。此后本端可不调用Close方法。连接处于完全关闭状态。

* 由于golang里net.conn内部对文件描述符的所有io操作都有状态保护，所以即使在对端或本端关闭了连接之后，依然可以任意次数调用Read、Write、Close方法