# SSH 端口转发

## 简介

SSH 除了登录服务器，还有一大用途，就是作为加密通信的中介，充当两台服务器之间的通信加密跳板，使得原本不加密的通信变成加密通信。这个功能称为端口转发（port forwarding），又称 SSH 隧道（tunnel）。

端口转发有两个主要作用：

（1）将不加密的数据放在 SSH 安全连接里面传输，使得原本不安全的网络服务增加了安全性，比如通过端口转发访问 Telnet、FTP 等明文服务，数据传输就都会加密。

（2）作为数据通信的加密跳板，绕过网络防火墙。

端口转发有三种使用方法：动态转发，本地转发，远程转发。下面逐一介绍。

## 动态转发

动态转发指的是，本机与 SSH 服务器之间创建了一个加密连接，然后本机内部针对某个端口的通信，都通过这个加密连接转发。它的一个使用场景就是，访问所有外部网站，都通过 SSH 转发。

动态转发需要把本地端口绑定到 SSH 服务器。至于 SSH 服务器要去访问哪一个网站，完全是动态的，取决于原始通信，所以叫做动态转发。

```bash
$ ssh -D local-port tunnel-host -N
```

上面命令中，`-D`表示动态转发，`local-port`是本地端口，`tunnel-host`是 SSH 服务器，`-N`表示这个 SSH 连接只进行端口转发，不登录远程 Shell，不能执行远程命令，只能充当隧道。

举例来说，如果本地端口是`2121`，那么动态转发的命令就是下面这样。

```bash
$ ssh -D 2121 tunnel-host -N
```

注意，这种转发采用了 SOCKS5 协议。访问外部网站时，需要把 HTTP 请求转成 SOCKS5 协议，才能把本地端口的请求转发出去。

下面是 SSH 隧道建立后的一个使用实例。

```bash
$ curl -x socks5://localhost:2121 http://www.example.com
```

上面命令中，curl 的`-x`参数指定代理服务器，即通过 SOCKS5 协议的本地`2121`端口，访问`http://www.example.com`。

如果经常使用动态转发，可以将设置写入 SSH 客户端的用户个人配置文件（`~/.ssh/config`）。

```bash
DynamicForward tunnel-host:local-port
```

## 本地转发

本地转发（local forwarding）指的是，创建一个本地端口，将发往该端口的所有通信都通过 SSH 服务器，转发到指定的远程服务器的端口。这种情况下，SSH 服务器只是一个作为跳板的中介，用于连接本地计算机无法直接连接的远程服务器。本地转发是在本地计算机建立的转发规则。

它的语法如下，其中会指定本地端口（local-port）、SSH 服务器（tunnel-host）、远程服务器（target-host）和远程端口（target-port）。

```html
$ ssh -L -N -f local-port:target-host:target-port tunnel-host
```

上面命令中，有三个配置参数。

- `-L`：转发本地端口。
- `-N`：不发送任何命令，只用来建立连接。没有这个参数，会在 SSH 服务器打开一个 Shell。
- `-f`：将 SSH 连接放到后台。没有这个参数，暂时不用 SSH 连接时，终端会失去响应。

举例来说，现在有一台 SSH 服务器`tunnel-host`，我们想要通过这台机器，在本地`2121`端口与目标网站`www.example.com`的80端口之间建立 SSH 隧道，就可以写成下面这样。

```bash
$ ssh -L 2121:www.example.com:80 tunnel-host -N
```

然后，访问本机的`2121`端口，就是访问`www.example.com`的80端口。

```bash
$ curl http://localhost:2121
```

注意，本地端口转发采用 HTTP 协议，不用转成 SOCKS5 协议。

另一个例子是加密访问邮件获取协议 POP3。

```bash
$ ssh -L 1100:mail.example.com:110 mail.example.com
```

上面命令将本机的1100端口，绑定邮件服务器`mail.example.com`的110端口（POP3 协议的默认端口）。端口转发建立以后，POP3 邮件客户端只需要访问本机的1100端口，请求就会通过 SSH 服务器（这里是`mail.example.com`），自动转发到`mail.example.com`的110端口。

上面这种情况有一个前提条件，就是`mail.example.com`必须运行 SSH 服务器。否则，就必须通过另一台 SSH 服务器中介，执行的命令要改成下面这样。

```bash
$ ssh -L 1100:mail.example.com:110 other.example.com
```

上面命令中，本机的1100端口还是绑定`mail.example.com`的110端口，但是由于`mail.example.com`没有运行 SSH 服务器，所以必须通过`other.example.com`中介。本机的 POP3 请求通过1100端口，先发给`other.example.com`的22端口（sshd 默认端口），再由后者转给`mail.example.com`，得到数据以后再原路返回。

注意，采用上面的中介方式，只有本机到`other.example.com`的这一段是加密的，`other.example.com`到`mail.example.com`的这一段并不加密。

这个命令最好加上`-N`参数，表示不在 SSH 跳板机执行远程命令，让 SSH 只充当隧道。另外还有一个`-f`参数表示 SSH 连接在后台运行。

如果经常使用本地转发，可以将设置写入 SSH 客户端的用户个人配置文件（`~/.ssh/config`）。

```bash
Host test.example.com
LocalForward client-IP:client-port server-IP:server-port
```

## 远程转发

远程转发指的是在远程 SSH 服务器建立的转发规则。

它跟本地转发正好反过来。建立本地计算机到远程 SSH 服务器的隧道以后，本地转发是通过本地计算机访问远程 SSH 服务器，而远程转发则是通过远程 SSH 服务器访问本地计算机。它的命令格式如下。

```bash
$ ssh -R remote-port:target-host:target-port -N remotehost
```

上面命令中，`-R`参数表示远程端口转发，`remote-port`是远程 SSH 服务器的端口，`target-host`和`target-port`是目标服务器及其端口，`remotehost`是远程 SSH 服务器。

远程转发主要针对内网的情况。下面举两个例子。

第一个例子是内网某台服务器`localhost`在 80 端口开了一个服务，可以通过远程转发将这个 80 端口，映射到具有公网 IP 地址的`my.public.server`服务器的 8080 端口，使得访问`my.public.server:8080`这个地址，就可以访问到那台内网服务器的 80 端口。

```bash
$ ssh -R 8080:localhost:80 -N my.public.server
```

上面命令是在内网`localhost`服务器上执行，建立从`localhost`到`my.public.server`的 SSH 隧道。运行以后，用户访问`my.public.server:8080`，就会自动映射到`localhost:80`。

第二个例子是本地计算机`local`在外网，SSH 跳板机和目标服务器`my.private.server`都在内网，必须通过 SSH 跳板机才能访问目标服务器。但是，本地计算机`local`无法访问内网之中的 SSH 跳板机，而 SSH 跳板机可以访问本机计算机。

由于本机无法访问内网 SSH 跳板机，就无法从外网发起 SSH 隧道，建立端口转发。必须反过来，从 SSH 跳板机发起隧道，建立端口转发，这时就形成了远程端口转发。跳板机执行下面的命令，绑定本地计算机`local`的`2121`端口，去访问`my.private.server:80`。

```bash
$ ssh -R 2121:my.private.server:80 -N local
```

上面命令是在 SSH 跳板机上执行的，建立跳板机到`local`的隧道，并且这条隧道的出口映射到`my.private.server:80`。

显然，远程转发要求本地计算机`local`也安装了 SSH 服务器，这样才能接受 SSH 跳板机的远程登录。

执行上面的命令以后，跳板机到`local`的隧道已经建立了。然后，就可以从本地计算机访问目标服务器了，即在本机执行下面的命令。

```bash
$ curl http://localhost:2121
```

本机执行上面的命令以后，就会输出服务器`my.private.server`的 80 端口返回的内容。

如果经常执行远程端口转发，可以将设置写入 SSH 客户端的用户个人配置文件（`~/.ssh/config`）。

```bash
Host remote-forward
  HostName test.example.com
  RemoteForward remote-port target-host:target-port
```

完成上面的设置后，执行下面的命令就会建立远程转发。

```bash
$ ssh -N remote-forward

# 等同于
$ ssh -R remote-port:target-host:target-port -N test.example.com
```

## 实例

下面看两个端口转发的实例。

### 简易 VPN

VPN 用来在外网与内网之间建立一条加密通道。内网的服务器不能从外网直接访问，必须通过一个跳板机，如果本机可以访问跳板机，就可以使用 SSH 本地转发，简单实现一个 VPN。

```bash
$ ssh -L 2080:corp-server:80 -L 2443:corp-server:443 tunnel-host -N
```

上面命令通过 SSH 跳板机，将本机的`2080`端口绑定内网服务器的`80`端口，本机的`2443`端口绑定内网服务器的`443`端口。

### 两级跳板

端口转发可以有多级，比如新建两个 SSH 隧道，第一个隧道转发给第二个隧道，第二个隧道才能访问目标服务器。

首先，在本机新建第一级隧道。

```bash
$ ssh -L 7999:localhost:2999 tunnel1-host
```

上面命令在本地`7999`端口与`tunnel1-host`之间建立一条隧道，隧道的出口是`tunnel1-host`的`localhost:2999`，也就是`tunnel1-host`收到本机的请求以后，转发给自己的`2999`端口。

然后，在第一台跳板机（`tunnel1-host`）执行下面的命令，新建第二级隧道。

```bash
$ ssh -L 2999:target-host:7999 tunnel2-host -N
```

上面命令将第一台跳板机`tunnel1-host`的`2999`端口，通过第二台跳板机`tunnel2-host`，连接到目标服务器`target-host`的`7999`端口。

最终效果就是，访问本机的`7999`端口，就会转发到`target-host`的`7999`端口。

## 参考链接

- [An Illustrated Guide to SSH Tunnels](https://solitum.net/posts/an-illustrated-guide-to-ssh-tunnels/), Scott Wiersdorf
- [An Excruciatingly Detailed Guide To SSH](https://grahamhelton.com/blog/ssh-cheatsheet/), Graham Helton

