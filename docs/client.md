# SSH 客户端

## 简介

OpenSSH 的客户端是二进制程序 ssh。它在 Linux/Unix 系统的位置是`/usr/local/bin/ssh`。

Linux 系统一般都自带 ssh，如果没有就需要安装。

```bash
# Ubuntu 和 Debian
$ sudo apt install openssh-client

# CentOS 和 Fedora
$ sudo dnf install openssh-clients
```

安装以后，可以使用`-V`参数输出版本号，查看一下是否安装成功。

```bash
$ ssh -V
```

## 基本用法

ssh 最常见的用途就是登录服务器，这要求服务器安装并正在运行 SSH 服务器软件。

ssh 登录服务器的命令如下。

```bash
$ ssh hostname
```

上面命令中，`hostname`是主机名，它可以是域名，也可能是 IP 地址或局域网内部的主机名。不指定用户名的情况下，将使用客户端的当前用户名，作为远程服务器的登录用户名。如果要指定用户名，可以采用下面的语法。

```bash
$ ssh user@hostname
```

上面的命令中，用户名和主机名写在一起了，之间使用`@`分隔。

用户名也可以使用`ssh`的`-l`参数指定，这样的话，用户名和主机名就不用写在一起了。

```bash
$ ssh -l username host
```

ssh 默认连接服务器的22端口，`-p`参数可以指定其他端口。

```bash
$ ssh -p 8821 foo.com
```

上面命令连接服务器`foo.com`的8821端口。

## 连接流程

ssh 连接远程服务器后，首先有一个验证过程，验证远程服务器是否为陌生地址。

如果是第一次连接某一台服务器，命令行会显示一段文字，表示不认识这台机器，提醒用户确认是否需要连接。

```bash
The authenticity of host 'foo.com (192.168.121.111)' can't be established.
ECDSA key fingerprint is SHA256:Vybt22mVXuNuB5unE++yowF7lgA/9/2bLSiO3qmYWBY.
Are you sure you want to continue connecting (yes/no)?
```

上面这段文字告诉用户，`foo.com`这台服务器的指纹是陌生的，让用户选择是否要继续连接（输入 yes 或 no）。

所谓“服务器指纹”，指的是 SSH 服务器公钥的哈希值。每台 SSH 服务器都有唯一一对密钥，用于跟客户端通信，其中公钥的哈希值就可以用来识别服务器。

下面的命令可以查看某个公钥的指纹。

```bash
$ ssh-keygen -l -f /etc/ssh/ssh_host_ecdsa_key.pub
256 da:24:43:0b:2e:c1:3f:a1:84:13:92:01:52:b4:84:ff   (ECDSA)
```

上面的例子中，`ssh-keygen -l -f`命令会输出公钥`/etc/ssh/ssh_host_ecdsa_key.pub`的指纹。

ssh 会将本机连接过的所有服务器公钥的指纹，都储存在本机的`~/.ssh/known_hosts`文件中。每次连接服务器时，通过该文件判断是否为陌生主机（陌生公钥）。

在上面这段文字后面，输入`yes`，就可以将当前服务器的指纹也储存在本机`~/.ssh/known_hosts`文件中，并显示下面的提示。以后再连接的时候，就不会再出现警告了。

```bash
Warning: Permanently added 'foo.com (192.168.121.111)' (RSA) to the list of known hosts
```

然后，客户端就会跟服务器建立连接。接着，ssh 就会要求用户输入所要登录账户的密码。用户输入并验证密码正确以后，就能登录远程服务器的 Shell 了。

## 服务器密钥变更

服务器指纹可以防止有人恶意冒充远程主机。如果服务器的密钥发生变更（比如重装了 SSH 服务器），客户端再次连接时，就会发生公钥指纹不吻合的情况。这时，客户端就会中断连接，并显示一段警告信息。

```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that the RSA host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
77:a5:69:81:9b:eb:40:76:7b:13:04:a9:6c:f4:9c:5d.
Please contact your system administrator.
Add correct host key in /home/me/.ssh/known_hosts to get rid of this message.
Offending key in /home/me/.ssh/known_hosts:36
```

上面这段文字的意思是，该主机的公钥指纹跟`~/.ssh/known_hosts`文件储存的不一样，必须处理以后才能连接。这时，你需要确认是什么原因，使得公钥指纹发生变更，到底是恶意劫持，还是管理员变更了 SSH 服务器公钥。

如果新的公钥确认可以信任，需要继续执行连接，你可以执行下面的命令，将原来的公钥指纹从`~/.ssh/known_hosts`文件删除。

```bash
$ ssh-keygen -R hostname
```

上面命令中，`hostname`是发生公钥变更的主机名。

除了使用上面的命令，你也可以手工修改`known_hosts`文件，将公钥指纹删除。

删除了原来的公钥指纹以后，重新执行 ssh 命令连接远程服务器，将新的指纹加入`known_hosts`文件，就可以顺利连接了。

## 执行远程命令

SSH 登录成功后，用户就进入了远程主机的命令行环境，所看到的提示符，就是远程主机的提示符。这时，你就可以输入想要在远程主机执行的命令。

另一种执行远程命令的方法，是将命令直接写在`ssh`命令的后面。

```bash
$ ssh username@hostname command
```

上面的命令会使得 SSH 在登录成功后，立刻在远程主机上执行命令`command`。

下面是一个例子。

```bash
$ ssh foo@server.example.com cat /etc/hosts
```

上面的命令会在登录成功后，立即远程执行命令`cat /etc/hosts`。

采用这种语法执行命令时，ssh 客户端不会提供互动式的 Shell 环境，而是直接将远程命令的执行结果输出在命令行。但是，有些命令需要互动式的 Shell 环境，这时就要使用`-t`参数。

```bash
# 报错
$ ssh remote.server.com emacs
emacs: standard input is not a tty

# 不报错
$ ssh -t server.example.com emacs
```

上面代码中，`emacs`命令需要一个互动式 Shell，所以报错。只有加上`-t`参数，ssh 才会分配一个互动式 Shell。

## 加密参数

SSH 连接的握手阶段，客户端必须跟服务端约定加密参数集（cipher suite）。

加密参数集包含了若干不同的加密参数，它们之间使用下划线连接在一起，下面是一个例子。

```bash
TLS_RSA_WITH_AES_128_CBC_SHA
```

它的含义如下。

- TLS：加密通信协议
- RSA：密钥交换算法
- AES：加密算法
- 128：加密算法的强度
- CBC：加密算法的模式
- SHA：数字签名的 Hash 函数

下面是一个例子，客户端向服务器发出的握手信息。

```http
Handshake protocol: ClientHello
    Version: TLS 1.2
    Random
        Client time: May 22, 2030 02:43:46 GMT
        Random bytes: b76b0e61829557eb4c611adfd2d36eb232dc1332fe29802e321ee871
    Session ID: (empty)
    Cipher Suites
        Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256”
        Suite: TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
        Suite: TLS_RSA_WITH_AES_128_GCM_SHA256
        Suite: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
        Suite: TLS_DHE_RSA_WITH_AES_128_CBC_SHA
        Suite: TLS_RSA_WITH_AES_128_CBC_SHA
        Suite: TLS_RSA_WITH_3DES_EDE_CBC_SHA
        Suite: TLS_RSA_WITH_RC4_128_SHA
    Compression methods
        Method: null
    Extensions
        Extension: server_name
            Hostname: www.feistyduck.com
        Extension: renegotiation_info
        Extension: elliptic_curves
            Named curve: secp256r1
            Named curve: secp384r1
        Extension: signature_algorithms
            Algorithm: sha1/rsa
            Algorithm: sha256/rsa
            Algorithm: sha1/ecdsa
            Algorithm: sha256/ecdsa”
```

上面的握手信息（ClientHello）之中，`Cipher Suites`字段就是客户端列出可选的加密参数集，服务器在其中选择一个自己支持的参数集。

服务器选择完毕之后，向客户端发出回应。

```http
Handshake protocol: ServerHello
    Version: TLS 1.2
    Random
        Server time: Mar 10, 2059 02:35:57 GMT”
        Random bytes: 8469b09b480c1978182ce1b59290487609f41132312ca22aacaf5012
    Session ID: 4cae75c91cf5adf55f93c9fb5dd36d19903b1182029af3d527b7a42ef1c32c80
    Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    Compression method: null
    Extensions
        Extension: server_name
        Extension: renegotiation_info”
```

上面的回应信息（ServerHello）中，`Cipher Suite`字段就是服务器最终选定的加密参数。

## ssh 命令行配置项

ssh 命令有很多配置项，修改它的默认行为。

**-c**

`-c`参数指定加密算法。

```bash
$ ssh -c blowfish,3des server.example.com
# 或者
$ ssh -c blowfish -c 3des server.example.com
```

上面命令指定使用加密算法`blowfish`或`3des`。

**-C**

`-C`参数表示压缩数据传输。

```bash
$ ssh -C server.example.com
```

**-D**

`-D`参数指定本机的 Socks 监听端口，该端口收到的请求，都将转发到远程的 SSH 主机，又称动态端口转发，详见《端口转发》一章。

```bash
$ ssh -D 1080 server
```

上面命令将本机 1080 端口收到的请求，都转发到服务器`server`。

**-f**

`-f`参数表示 SSH 连接在后台运行。

**-F**

`-F`参数指定配置文件。

```bash
$ ssh -F /usr/local/ssh/other_config
```

上面命令指定使用配置文件`other_config`。

**-i**

`-i`参数用于指定私钥，意为“identity_file”，默认值为`~/.ssh/id_dsa`（DSA 算法）和`~/.ssh/id_rsa`（RSA 算法）。注意，对应的公钥必须存放到服务器，详见《密钥登录》一章。

```bash
$ ssh -i my-key server.example.com
```

**-J**

`-J`指定跳转服务器。假定本地无法直接与 SSH 服务器通信，就可以通过`—J`指定跳转服务器。

```bash
$ ssh -J root@J1,root@J2 root@S1
```

上面示例中，本机先通过 J1，再通过 J2，登陆到 S1 服务器。

**-l**

`-l`参数指定远程登录的账户名。

```bash
$ ssh -l sally server.example.com
# 等同于
$ ssh sally@server.example.com
```

**-L**

`-L`参数设置本地端口转发，详见《端口转发》一章。

```bash
$ ssh  -L 9999:targetServer:80 user@remoteserver
```

上面命令中，所有发向本地`9999`端口的请求，都会经过`remoteserver`发往 targetServer 的 80 端口，这就相当于直接连上了 targetServer 的 80 端口。

**-m**

`-m`参数指定校验数据完整性的算法（message authentication code，简称 MAC）。

```bash
$ ssh -m hmac-sha1,hmac-md5 server.example.com
```

上面命令指定数据校验算法为`hmac-sha1`或`hmac-md5`。

**-N**

`-N`参数用于端口转发，表示建立的 SSH 只用于端口转发，不能执行远程命令，这样可以提供安全性，详见《端口转发》一章。

**-o**

`-o`参数用来指定一个配置命令。

```bash
$ ssh -o "Keyword Value"
```

举例来说，配置文件里面有如下内容。

```bash
User sally
Port 220
```

通过`-o`参数，可以把上面两个配置命令从命令行传入。

```bash
$ ssh -o "User sally" -o "Port 220" server.example.com
```

使用等号时，配置命令可以不用写在引号里面，但是等号前后不能有空格。

```bash
$ ssh -o User=sally -o Port=220 server.example.com
```

**-p**

`-p`参数指定 SSH 客户端连接的服务器端口。

```bash
$ ssh -p 2035 server.example.com
```

上面命令连接服务器的2035端口。

**-q**

`-q`参数表示安静模式（quiet），不向用户输出任何警告信息。

```bash
$ ssh –q foo.com
root’s password:
```

上面命令使用`-q`参数，只输出要求用户输入密码的提示。

**-R**

`-R`参数指定远程端口转发，详见《端口转发》一章。

```bash
$ ssh -R 9999:targetServer:902 local
```

上面命令需在跳板服务器执行，指定本地计算机`local`监听自己的 9999 端口，所有发向这个端口的请求，都会转向 targetServer 的 902 端口。

**-t**

`-t`参数在 ssh 直接运行远端命令时，提供一个互动式 Shell。

```bash
$ ssh -t server.example.com emacs
```

**-v**

`-v`参数显示详细信息。

```bash
$ ssh -v server.example.com
```

`-v`可以重复多次，表示信息的详细程度，比如`-vv`和`-vvv`。

```bash
$ ssh -vvv server.example.com
# 或者
$ ssh -v -v -v server.example.com
```

上面命令会输出最详细的连接信息。

**-V**

`-V`参数输出 ssh 客户端的版本。

```bash
$ ssh –V
ssh: SSH Secure Shell 3.2.3 (non-commercial version) on i686-pc-linux-gnu
```

上面命令输出本机 ssh 客户端版本是`SSH Secure Shell 3.2.3`。

**-X**

`-X`参数表示打开 X 窗口转发。

```bash
$ ssh -X server.example.com
```

**-1，-2**

`-1`参数指定使用 SSH 1 协议。

`-2`参数指定使用 SSH 2 协议。

```ssh
$ ssh -2 server.example.com
```

**-4，-6**

`-4`指定使用 IPv4 协议，这是默认值。

```bash
$ ssh -4 server.example.com
```

`-6`指定使用 IPv6 协议。

```bash
$ ssh -6 server.example.com
```

## 客户端配置文件

### 位置

SSH 客户端的全局配置文件是`/etc/ssh/ssh_config`，用户个人的配置文件在`~/.ssh/config`，优先级高于全局配置文件。

除了配置文件，`~/.ssh`目录还有一些用户个人的密钥文件和其他文件。下面是其中一些常见的文件。

- `~/.ssh/id_ecdsa`：用户的 ECDSA 私钥。
- `~/.ssh/id_ecdsa.pub`：用户的 ECDSA 公钥。
- `~/.ssh/id_rsa`：用于 SSH 协议版本2 的 RSA 私钥。
- `~/.ssh/id_rsa.pub`：用于SSH 协议版本2 的 RSA 公钥。
- `~/.ssh/identity`：用于 SSH 协议版本1 的 RSA 私钥。
- `~/.ssh/identity.pub`：用于 SSH 协议版本1 的 RSA 公钥。
- `~/.ssh/known_hosts`：包含 SSH 服务器的公钥指纹。

### 主机设置

用户个人的配置文件`~/.ssh/config`，可以按照不同服务器，列出各自的连接参数，从而不必每一次登录都输入重复的参数。下面是一个例子。

```bash
Host remoteserver
     HostName remote.example.com
     User neo
     Port 2112

Host *
     Port 2222
```

上面代码中，`Host remoteserver`表示，下面的设置只对主机`remoteserver`生效。`remoteserver`只是一个别名，具体的主机由`HostName`命令指定，`User`和`Port`这两项分别表示用户名和端口。`HostName`、`User`、`Port`这三项前面的缩进并不是必需的，只是为了视觉上易于识别针对不同主机的设置。

后面的`Host *`表示对所有主机生效，`*`是一个通配符，比如`Host *.edu`表示只对一级域名为`.edu`的主机生效。这条命令下面的`Port 2222`表示所有主机的默认连接端口都是2222，这样就不用在登录时特别指定端口了。

注意，当`Host *`与`Host remoteserver`下面有同一项设定时（比如两者都有`Port`设定），第一个出现的值生效。在本例中，连接`remoteserver`时，默认端口将是2112，而不是2222，如果`Host *`放在配置文件的顶部，那么默认端口将是2222。

以后，登录`remote.example.com`时，只要执行`ssh remoteserver`命令，就会自动套用 config 文件里面指定的参数。

```bash
$ ssh remoteserver
# 等同于
$ ssh -p 2112 neo@remote.example.com
```

###  配置命令的语法

ssh 客户端配置文件的每一行，就是一个配置命令。配置命令与对应的值之间，可以使用空格，也可以使用等号。

```bash
Compression yes
# 等同于
Compression = yes
```

`#`开头的行表示注释，会被忽略。空行等同于注释。

### 主要配置命令

下面是 ssh 客户端的一些主要配置命令，以及它们的范例值。

- `AddressFamily inet`：表示只使用 IPv4 协议。如果设为`inet6`，表示只使用 IPv6 协议。
- `BindAddress 192.168.10.235`：指定本机的 IP 地址（如果本机有多个 IP 地址）。
- `CheckHostIP yes`：检查 SSH 服务器的 IP 地址是否跟公钥数据库吻合。
- `Ciphers blowfish,3des`：指定加密算法。
- `Compression yes`：是否压缩传输信号。
- `ConnectionAttempts 10`：客户端进行连接时，最大的尝试次数。
- `ConnectTimeout 60`：客户端进行连接时，服务器在指定秒数内没有回复，则中断连接尝试。
- `DynamicForward 1080`：指定动态转发端口。
- `GlobalKnownHostsFile /users/smith/.ssh/my_global_hosts_file`：指定全局的公钥数据库文件的位置。
- `Host server.example.com`：指定连接的域名或 IP 地址，也可以是别名，支持通配符。`Host`命令后面的所有配置，都是针对该主机的，直到下一个`Host`命令为止。
- `HostKeyAlgorithms ssh-dss,ssh-rsa`：指定密钥算法，优先级从高到低排列。
- `HostName myserver.example.com`：在`Host`命令使用别名的情况下，`HostName`指定域名或 IP 地址。
- `IdentityFile keyfile`：指定私钥文件。
- `LocalForward 2001 localhost:143`：指定本地端口转发。
- `LogLevel QUIET`：指定日志详细程度。如果设为`QUIET`，将不输出大部分的警告和提示。
- `MACs hmac-sha1,hmac-md5`：指定数据校验算法。
- `NumberOfPasswordPrompts 2`：密码登录时，用户输错密码的最大尝试次数。
- `PasswordAuthentication no`：指定是否支持密码登录。不过，这里只是客户端禁止，真正的禁止需要在 SSH 服务器设置。
- `Port 2035`：指定客户端连接的 SSH 服务器端口。
- `PreferredAuthentications publickey,hostbased,password`：指定各种登录方法的优先级。
- `Protocol 2`：支持的 SSH 协议版本，多个版本之间使用逗号分隔。
- `PubKeyAuthentication yes`：是否支持密钥登录。这里只是客户端设置，还需要在 SSH 服务器进行相应设置。
- `RemoteForward 2001 server:143`：指定远程端口转发。
- `SendEnv COLOR`：SSH 客户端向服务器发送的环境变量名，多个环境变量之间使用空格分隔。环境变量的值从客户端当前环境中拷贝。
- `ServerAliveCountMax 3`：如果没有收到服务器的回应，客户端连续发送多少次`keepalive`信号，才断开连接。该项默认值为3。
- `ServerAliveInterval 300`：客户端建立连接后，如果在给定秒数内，没有收到服务器发来的消息，客户端向服务器发送`keepalive`消息。如果不希望客户端发送，这一项设为`0`，即客户端不会主动断开连接。
- `StrictHostKeyChecking yes`：`yes`表示严格检查，服务器公钥为未知或发生变化，则拒绝连接。`no`表示如果服务器公钥未知，则加入客户端公钥数据库，如果公钥发生变化，不改变客户端公钥数据库，输出一条警告，依然允许连接继续进行。`ask`（默认值）表示询问用户是否继续进行。
- `TCPKeepAlive yes`：客户端是否定期向服务器发送`keepalive`信息。
- `User userName`：指定远程登录的账户名。
- `UserKnownHostsFile /users/smith/.ssh/my_local_hosts_file`：指定当前用户的`known_hosts`文件（服务器公钥指纹列表）的位置。
- `VerifyHostKeyDNS yes`：是否通过检查 SSH 服务器的 DNS 记录，确认公钥指纹是否与`known_hosts`文件保存的一致。

