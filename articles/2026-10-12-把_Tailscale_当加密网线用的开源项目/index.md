# 把 Tailscale 当加密网线用的开源项目

> ✍️ **作者:** 小G  
> 📅 **发布时间:** 2026/10/9 08:17:06  
> 🔗 **微信原文:** https://mp.weixin.qq.com/s/iKZCQF7vlgfpI7SjhDS4Ww  
> 📦 **本地图片数:** 0 张 (已物理转存至本地 images/ 目录)

---



## 项目简介

Tailscale 好用，但有时候你只想干一件事：两台机器之间拉一条加密通道，传个文件、转个端口、开个临时 SSH。不想登录账号，不想改路由表，更不想给机器装一整套客户端。

Tailcat 把控制面整段拿掉了。一端起服务，打出一段 tc 开头的地址；另一端把这段地址贴进去。地址里已经带了公钥、打洞要用的信息和一把预共享密钥。怎么把地址交给对方，随你：微信、U 盘、当面念都行。

所以：
不用 Tailscale 账号
不用 root
不改 DNS、不改路由
就是个用户态程序

## 适用场景

两台临时机器要通一下，账号体系来不及搞。
内网盒子没有公网端口，又不想上整套组网。
把一个本地开发端口给同事看一眼，看完关掉。
文件不能走网盘，但有一段能见面或能私聊的时间。

不适合的情况也清楚：要长期稳定入口，却把无认证 SSH 的地址写进 DNS；拿官方限速 DERP 当大流量中转；假设明年命令行参数还长这样。

## 怎么用

## 两边管道 stdin / stdout

服务端先起，打出一次性地址：

$ tailcat
# Selected bootstrap relay region 302, San Francisco
# 🐈 Server listening with new address: tcomFwWCCcjS5nKNqAod034nWoJZW0LZqDhhC8U_dKdnDRYQ8uNGFpGQEu
(hangs, waiting...)

客户端：

$ echo hello | tailcat tcomFwWCCcjS5nKNqAod034nWoJZW0LZqDhhC8U_dKdnDRYQ8uNGFpGQEu

服务端那边就会印出 hello。

## 把本机端口掏出去

$ tailcat serve 8080,8443   # 或者 tailcat serve all
# 🐈 Server listening with new address: tcXXXXXXXXX

对面：

$ tailcat tcXXXXXXXXX 8080
GET / HTTP/1.1
Host: foo

HTTP/1.1 200 OK
....

流量转到本机对应端口。

## 把对面的端口接到本机

浏览器、数据库客户端这类既不认 SOCKS 也不吃标准输入的，用 forward：

$ tailcat serve 8080,3306
$ tailcat forward tcXXXXXXXXX 18080:8080 3306

本地端口写 0 就是让系统随便挑一个空闲端口，监听起来会把实际地址打出来。

对面如果开了 exit-node，还可以把本机端口转到它所在网上的别的机器：

$ tailcat serve exit-node
$ tailcat forward tcXXXXXXXXX \
    3001:172.23.52.30:3001 \
    17170:172.23.52.31:17170

默认只绑 127.0.0.1，日志也关着。要看网络细节，子命令前面加 --verbose。只有别的机器也要连这个转发口时，才用 --bind=0.0.0.0。Ctrl-C 停掉。

## 用浏览器打开对面的网站

$ tailcat serve 80
$ tailcat browse tcXXXXXXXXX

等价于 tailcat forward --open-browser &lt;地址&gt; 0:80：本机端口起来后打开 http://127.0.0.1:&lt;端口&gt;/，然后一直转，直到你打断。--open-browser 对任意一条单独的 forward 映射都有效。

## 用公钥管 SSH

认本机 authorized_keys、直接写的 OpenSSH 公钥行，或者 GitHub 账号上的钥匙：

$ tailcat serve --ssh-authorized-keys=~/.ssh/authorized_keys ssh

多个来源逗号隔开。user@github 会在启动前拉一次 https://github.com/user.keys。哪个来源找不到、拉失败、或里面没有合法公钥，进程直接起不来。command=、from= 这类选项不认（内置 SSH 没实现）。光写 tailcat serve ssh、不带 --ssh-authorized-keys 也会失败；觉得「能进隧道就够了」就用 no-auth-ssh。

## 不认证的 SSH

Linux / macOS / Windows 可以明确关掉客户端认证，身份就靠这条加密隧道。

$ tailcat serve no-auth-ssh

**注意**
这种模式下，**地址就是钥匙**。谁拿到地址，谁就能以跑服务的那个用户身份进 shell。只私下传，别往外贴，也别写进 DNS TXT。要用域名访问 SSH，必须另做认证：隧道上 --allow，SSH 上 --ssh-authorized-keys，或两个一起。

客户端：

$ tailcat ssh tcXXXXXXXXX
$ tailcat ssh tcXXXXXXXXX ls -la

## 每个连接跑一条命令

跟 inetd 差不多。exec 来一个连接就起一次命令，连接当 stdin/stdout。命令写在 -- 后面：

$ tailcat serve exec -- /usr/bin/fortune

命令的 stderr 打到服务端进程上。环境变量里有对端节点密钥 $TAILCAT_PEER_KEY（格式跟 --allow 一样）和对端 $TAILCAT_REMOTE_ADDR。

跟 ssh / no-auth-ssh 一起用时，这条命令相当于 OpenSSH 的 ForceCommand：每次会话只跑它（客户端要 PTY 就给 PTY），没有 shell、不能自己指定命令、也没有 SFTP。客户端原来想跑的命令（如果有）在 $SSH_ORIGINAL_COMMAND。

$ tailcat serve --ssh-authorized-keys=alice@github ssh -- ./deploy.sh
$ tailcat serve no-auth-ssh -- git-upload-pack /srv/repo.git

## 传文件

收文件：开个投递箱，把打出来的地址给人。

$ tailcat recv ~/inbox
$ tailcat cp report.pdf tcXXXXXXXXX:

tailcat cp 其实是调系统自带的 scp，连接从 tailcat 里走，进度条还在，-r 能传目录。投递箱只写：对面不能列目录、不能读回来、也不能动已有文件。

往外提供文件：默认只读 serve 当前目录，也可以指定目录并开读写。

$ tailcat serve files
$ tailcat serve --files=/pub:rw files
$ tailcat ls -l tcXXXXXXXXX
$ tailcat cp tcXXXXXXXXX:report.pdf .

tailcat ls 直接说 SFTP，机器上没装 OpenSSH 也能用。路径被关在所 serve 的目录里（Go 的 os.Root），.. 和符号链接出不去。协议是 SFTP，系统自带的 sftp / scp 配上经 tailcat 的 ProxyCommand 也能用（tailcat cp 和 tailcat ssh 就是这么干的）。ssh 和 no-auth-ssh 同样提供 SFTP，权限跟 shell 一样。

传输不压缩。SFTP 自己没有压缩，这边的 SSH 传输层也没有（Go 那套 SSH 没做；传输层压缩历史上出过安全问题，TLS 后来也砍了）。要压就先自己压再传。

## 例子

## 用 DNS 挂受保护的 SSH

不用做端口转发，也不用 port knocking。按名字从外面 SSH 进来，服务端没有对公网开放的入站端口；WireGuard 先认人，SSH 才看得到包。

**注意**
下面的 --allow 不能省。TXT 把地址公开了，「拿得到地址」不再说明任何事，服务端必须自己认客户端。这里只放行一把客户端节点密钥。没有 --allow（也没有 SSH 层的 --ssh-authorized-keys），谁读到 TXT 谁就能连。

客户端先做一把身份钥匙，把公钥给服务端：

client$ tailcat genkey --client --key=client-default
# wrote file to ~/.config/tailcat/keys/client-default.private.json
nodekey:cfb6bfa77a0654d7450947fd6acef17d2cd848da1d30b2540b13dac272ddfd16

服务端生成钉在最近 DERP 区域上的服务端钥匙，然后只给这个客户端开 22：

server$ tailcat genkey --key=default --fixed-region
server$ tailcat serve --allow=nodekey:cfb6bf...ddfd16 22

DNS：

my-server.example.com. 300 IN TXT "tailcat=tcXXXXXXXXX"

客户端这边就一句：

client$ tailcat ssh my-server.example.com

有 client-default 就会自动用，不用再加参数。别人的握手直接丢掉，连「这台机器上有没有 SSH」都探不到。

多一层保险：tailcat ssh 连域名目标前会先试探——用一把刚生成的客户端密钥、不带任何 SSH 凭据，模拟陌生人登录。如果服务端也放行，说明读 TXT 的人一样进得去，于是拒绝连接并说明原因。第一次测自己的机器就能把配错抓出来。真要公开服务，或想少两次往返，用 --skip-dns-safety-check。

为什么要 --fixed-region：genkey 时就把最近的 DERP 区域写进地址和密钥文件，以后重启还绑同一区域，DNS 里那条地址才一直有效。默认是 --region=auto，启动再选，一次性用没问题；写进 DNS 的应该钉死。也可以 --region=&lt;名字&gt; 指定；--region=list 看有哪些。

DERP map 以后变了客户端怎么更稳，还没做完：https://github.com/tailscale/tailcat/issues/7[1]

## 自己架 DERP

不一定要用 Tailscale 的中继。自己跑 derper[2]（要有带 TLS 证书的主机名，它可以自己走 Let’s Encrypt），genkey 时把主机名当 region（多个就逗号隔开）：

server$ tailcat genkey --key=default --region=derp.example.com
server$ tailcat serve 22

地址里会带上你的中继主机名，客户端不用加参数，也不会去碰 Tailscale 的 DERP map 或中继，限速全是你自己的。要是有一串中继，可以自己提供 DERP map JSON，两边用 --derpmap-url 指过去。

## 当 Go 库用

最小服务端：隧道里任意 TCP 端口都回一句话，并打印地址。Server 零值会补默认：临时密钥、默认 DERP map 里最近的区域、log.Printf。不想说话就把 Logf 设成 logger.Discard。

package main

import (
"fmt"
"log"
"net"

"github.com/tailscale/tailcat"
)

funcmain() {
	s := &amp;tailcat.Server{
		OnTCP: func(port uint16)func(net.Conn) {
returnfunc(c net.Conn) {
				fmt.Fprintf(c, "hello from port %v\n", port)
				c.Close()
			}
		},
	}
if err := s.Start(); err != nil {
		log.Fatal(err)
	}
	fmt.Println(s.TailcatAddr())
select {}
}

最小客户端：第一个参数当地址去拨。Client 零值只要把 Server 设成地址就行（tailcat.NewClient 就是干这个的），隧道在第一次 dial 时才建。

package main

import (
"context"
"io"
"log"
"os"

"github.com/tailscale/tailcat"
)

funcmain() {
	cl := tailcat.NewClient(tailcat.Addr(os.Args[1]))
defer cl.Close()
	c, err := cl.DialTCPPort(context.Background(), 80)
if err != nil {
		log.Fatal(err)
	}
	io.Copy(os.Stdout, c)
}

UDP 每个客户端流是一条已连接的 packet conn，数据报边界和两端地址都保留。载荷别超过 tailcat.MaxUDPPayload（1232 字节），否则会撞 IPv6 隧道 MTU。出口节点用 OnUDPForward 和 DialUDP；双向转发用 ProxyPacketConns。服务端空闲 UDP 流默认两分钟关掉（tailcat.DefaultUDPIdleTimeout），用 Server.UDPIdleTimeout 改。

## 项目地址

https://github.com/tailscale/tailcat

## 参考资料

[1] 

https://github.com/tailscale/tailcat/issues/7: https://github.com/tailscale/tailcat/issues/7
[2] 

自己跑 derper: https://github.com/tailscale/tailscale/tree/main/cmd/derper#derp

