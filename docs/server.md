# SSH 服务器

## 简介

SSH 的架构是服务器/客户端模式，两端运行的软件是不一样的。OpenSSH 的客户端软件是 ssh，服务器软件是 sshd。本章介绍 sshd 的各种知识。

如果没有安装 sshd，可以用下面的命令安装。

```bash
# Debian
$ sudo aptitude install openssh-server

# Red Hat
$ sudo yum install openssh-server
```

一般来说，sshd 安装后会跟着系统一起启动。如果当前 sshd 没有启动，可以用下面的命令启动。

```bash
$ sshd
```

上面的命令运行后，如果提示“sshd re-exec requires execution with an absolute path”，就需要使用绝对路径来启动。这是为了防止有人出于各种目的，放置同名软件在`$PATH`变量指向的目录中，代替真正的 sshd。

```bash
# Centos、Ubuntu、OS X
$ /usr/sbin/sshd
```

上面的命令运行以后，sshd 自动进入后台，所以命令后面不需要加上`&`。

除了直接运行可执行文件，也可以通过 Systemd 启动 sshd。

```bash
# 启动
$ sudo systemctl start sshd.service

# 停止
$ sudo systemctl stop sshd.service

# 重启
$ sudo systemctl restart sshd.service
```

下面的命令让 sshd 在计算机下次启动时自动运行。

```bash
$ sudo systemctl enable sshd.service
```

## sshd 配置文件

sshd 的配置文件在`/etc/ssh`目录，主配置文件是`sshd_config`，此外还有一些安装时生成的密钥。

- `/etc/ssh/sshd_config`：配置文件
- `/etc/ssh/ssh_host_ecdsa_key`：ECDSA 私钥。
- `/etc/ssh/ssh_host_ecdsa_key.pub`：ECDSA 公钥。
- `/etc/ssh/ssh_host_key`：用于 SSH 1 协议版本的 RSA 私钥。
- `/etc/ssh/ssh_host_key.pub`：用于 SSH 1 协议版本的 RSA 公钥。
- `/etc/ssh/ssh_host_rsa_key`：用于 SSH 2 协议版本的 RSA 私钥。
- `/etc/ssh/ssh_host_rsa_key.pub`：用于 SSH 2 协议版本的 RSA 公钥。
- `/etc/pam.d/sshd`：PAM 配置文件。

注意，如果重装 sshd，上面这些密钥都会重新生成，导致客户端重新连接 ssh 服务器时，会跳出警告，拒绝连接。为了避免这种情况，可以在重装 sshd 时，先备份`/etc/ssh`目录，重装后再恢复这个目录。

配置文件`sshd_config`的格式是，每个命令占据一行。每行都是配置项和对应的值，配置项的大小写不敏感，与值之间使用空格分隔。

```bash
Port 2034
```

上面的配置命令指定，配置项`Port`的值是`2034`。`Port`写成`port`也可。

配置文件还有另一种格式，就是配置项与值之间有一个等号，等号前后的空格可选。

```bash
Port = 2034
```

配置文件里面，`#`开头的行表示注释。

```bash
# 这是一行注释
```

注意，注释只能放在一行的开头，不能放在一行的结尾。

```bash
Port 2034 # 此处不允许注释
```

上面的写法是错误的。

另外，空行等同于注释。

sshd 启动时会自动读取默认的配置文件。如果希望使用其他的配置文件，可以用 sshd 命令的`-f`参数指定。

```bash
$ sshd -f /usr/local/ssh/my_config
```

上面的命令指定 sshd 使用另一个配置文件`my_config`。

修改配置文件以后，可以用 sshd 命令的`-t`（test）检查有没有语法错误。

```bash
$ sshd -t
```

配置文件修改以后，并不会自动生效，必须重新启动 sshd。

```bash
$ sudo systemctl restart sshd.service
```

## sshd 密钥

sshd 有自己的一对或多对密钥。它使用密钥向客户端证明自己的身份。所有密钥都是公钥和私钥成对出现，公钥的文件名一般是私钥文件名加上后缀`.pub`。

DSA 格式的密钥文件默认为`/etc/ssh/ssh_host_dsa_key`（公钥为`ssh_host_dsa_key.pub`），RSA 格式的密钥为`/etc/ssh/ssh_host_rsa_key`（公钥为`ssh_host_rsa_key.pub`）。如果需要支持 SSH 1 协议，则必须有密钥`/etc/ssh/ssh_host_key`。

如果密钥不是默认文件，那么可以通过配置文件`sshd_config`的`HostKey`配置项指定。默认密钥的`HostKey`设置如下。

```bash
# HostKey for protocol version 1
# HostKey /etc/ssh/ssh_host_key

# HostKeys for protocol version 2
# HostKey /etc/ssh/ssh_host_rsa_key
# HostKey /etc/ssh/ssh_host_dsa_ke
```

上面命令前面的`#`表示这些行都是注释，因为这是默认值，有没有这几行都一样。

如果要修改密钥，就要去掉行首的`#`，指定其他密钥。

```bash
HostKey /usr/local/ssh/my_dsa_key
HostKey /usr/local/ssh/my_rsa_key
HostKey /usr/local/ssh/my_old_ssh1_key
```

## sshd 配置项

以下是`/etc/ssh/sshd_config`文件里面的配置项。

**AcceptEnv**

`AcceptEnv`指定允许接受客户端通过`SendEnv`命令发来的哪些环境变量，即允许客户端设置服务器的环境变量清单，变量名之间使用空格分隔（`AcceptEnv PATH TERM`）。

**AllowGroups**

`AllowGroups`指定允许登录的用户组（`AllowGroups groupName`，多个组之间用空格分隔。如果不使用该项，则允许所有用户组登录。

**AllowUsers**

`AllowUsers`指定允许登录的用户，用户名之间使用空格分隔（`AllowUsers user1 user2`），也可以使用多行`AllowUsers`命令指定，用户名支持使用通配符。如果不使用该项，则允许所有用户登录。该项也可以使用`用户名@域名`的格式（比如`AllowUsers jones@example.com`）。

**AllowTcpForwarding**

`AllowTcpForwarding`指定是否允许端口转发，默认值为`yes`（`AllowTcpForwarding yes`），`local`表示只允许本地端口转发，`remote`表示只允许远程端口转发。

**AuthorizedKeysFile**

`AuthorizedKeysFile`指定储存用户公钥的目录，默认是用户主目录的`ssh/authorized_keys`目录（`AuthorizedKeysFile .ssh/authorized_keys`）。

**Banner**

`Banner`指定用户登录后，sshd 向其展示的信息文件（`Banner /usr/local/etc/warning.txt`），默认不展示任何内容。

**ChallengeResponseAuthentication**

`ChallengeResponseAuthentication`指定是否使用“键盘交互”身份验证方案，默认值为`yes`（`ChallengeResponseAuthentication yes`）。

从理论上讲，“键盘交互”身份验证方案可以向用户询问多重问题，但是实践中，通常仅询问用户密码。如果要完全禁用基于密码的身份验证，请将`PasswordAuthentication`和`ChallengeResponseAuthentication`都设置为`no`。

**Ciphers**

`Ciphers`指定 sshd 可以接受的加密算法（`Ciphers 3des-cbc`），多个算法之间使用逗号分隔。

**ClientAliveCountMax**

`ClientAliveCountMax`指定建立连接后，客户端失去响应时（超过指定时间，没有收到任何消息），服务器尝试连接（发送消息）的次数（`ClientAliveCountMax 8`）。

**ClientAliveInterval**

`ClientAliveInterval`指定允许客户端发呆的时间，单位为秒（`ClientAliveInterval 180`）。超过这个时间，服务器将发送消息以请求客户端的响应。如果为`0`，表示不向客户端发送消息，即连接不会自动断开。

**Compression**

`Compression`指定客户端与服务器之间的数据传输是否压缩。默认值为`yes`（`Compression yes`）

**DenyGroups**

`DenyGroups`指定不允许登录的用户组（`DenyGroups groupName`）。

**DenyUsers**

`DenyUsers`指定不允许登录的用户（`DenyUsers user1`），用户名之间使用空格分隔，也可以使用多行`DenyUsers`命令指定。

**FascistLogging**

SSH 1 版本专用，指定日志输出全部 Debug 信息（`FascistLogging yes`）。

**HostKey**

`HostKey`指定 sshd 服务器的密钥，详见前文。

**KeyRegenerationInterval**

`KeyRegenerationInterval`指定 SSH 1 版本的密钥重新生成时间间隔，单位为秒，默认是3600秒（`KeyRegenerationInterval 3600`）。

**ListenAddress**

`ListenAddress`指定 sshd 监听的本机 IP 地址，即 sshd 启用的 IP 地址，默认是 0.0.0.0（`ListenAddress 0.0.0.0`）表示在本机所有网络接口启用。可以改成只在某个网络接口启用（比如`ListenAddress 192.168.10.23`），也可以指定某个域名启用（比如`ListenAddress server.example.com`）。

如果要监听多个指定的 IP 地址，可以使用多行`ListenAddress`命令。

```bash
ListenAddress 172.16.1.1
ListenAddress 192.168.0.1
```

**LoginGraceTime**

`LoginGraceTime`指定允许客户端登录时发呆的最长时间，比如用户迟迟不输入密码，连接就会自动断开，单位为秒（`LoginGraceTime 60`）。如果设为`0`，就表示没有限制。

**LogLevel**

`LogLevel`指定日志的详细程度，可能的值依次为`QUIET`、`FATAL`、`ERROR`、`INFO`、`VERBOSE`、`DEBUG`、`DEBUG1`、`DEBUG2`、`DEBUG3`，默认为`INFO`（`LogLevel INFO`）。

**MACs**

`MACs`指定sshd 可以接受的数据校验算法（`MACs hmac-sha1`），多个算法之间使用逗号分隔。

**MaxAuthTries**

`MaxAuthTries`指定允许 SSH 登录的最大尝试次数（`MaxAuthTries 3`），如果密码输入错误达到指定次数，SSH 连接将关闭。

**MaxStartups**

`MaxStartups`指定允许同时并发的 SSH 连接数量（MaxStartups）。如果设为`0`，就表示没有限制。

这个属性也可以设为`A:B:C`的形式，比如`MaxStartups 10:50:20`，表示如果达到10个并发连接，后面的连接将有50%的概率被拒绝；如果达到20个并发连接，则后面的连接将100%被拒绝。

**PasswordAuthentication**

`PasswordAuthentication`指定是否允许密码登录，默认值为`yes`（`PasswordAuthentication yes`），建议改成`no`（禁止密码登录，只允许密钥登录）。

**PermitEmptyPasswords**

`PermitEmptyPasswords`指定是否允许空密码登录，即用户的密码是否可以为空，默认为`yes`（`PermitEmptyPasswords yes`），建议改成`no`（禁止无密码登录）。

**PermitRootLogin**

`PermitRootLogin`指定是否允许根用户登录，默认为`yes`（`PermitRootLogin yes`），建议改成`no`（禁止根用户登录）。

还有一种写法是写成`prohibit-password`，表示 root 用户不能用密码登录，但是可以用密钥登录。

```bash
PermitRootLogin prohibit-password
```

**PermitUserEnvironment**

`PermitUserEnvironment`指定是否允许 sshd 加载客户端的`~/.ssh/environment`文件和`~/.ssh/authorized_keys`文件里面的`environment= options`环境变量设置。默认值为`no`（`PermitUserEnvironment no`）。

**Port**

`Port`指定 sshd 监听的端口，即客户端连接的端口，默认是22（`Port 22`）。出于安全考虑，可以改掉这个端口（比如`Port 8822`）。

配置文件可以使用多个`Port`命令，同时监听多个端口。

```bash
Port 22
Port 80
Port 443
Port 8080
```

上面的示例表示同时监听4个端口。

**PrintMotd**

`PrintMotd`指定用户登录后，是否向其展示系统的 motd（Message of the day）的信息文件`/etc/motd`。该文件用于通知所有用户一些重要事项，比如系统维护时间、安全问题等等。默认值为`yes`（`PrintMotd yes`），由于 Shell 一般会展示这个信息文件，所以这里可以改为`no`。

**PrintLastLog**

`PrintLastLog`指定是否打印上一次用户登录时间，默认值为`yes`（`PrintLastLog yes`）。

**Protocol**

`Protocol`指定 sshd 使用的协议。`Protocol 1`表示使用 SSH 1 协议，建议改成`Protocol 2`（使用 SSH 2 协议）。`Protocol 2,1`表示同时支持两个版本的协议。

**PubkeyAuthentication**

`PubkeyAuthentication`指定是否允许公钥登录，默认值为`yes`（`PubkeyAuthentication yes`）。

**QuietMode**

SSH 1 版本专用，指定日志只输出致命的错误信息（`QuietMode yes`）。

**RSAAuthentication**

`RSAAuthentication`指定允许 RSA 认证，默认值为`yes`（`RSAAuthentication yes`）。

**ServerKeyBits**

`ServerKeyBits`指定 SSH 1 版本的密钥重新生成时的位数，默认是768（`ServerKeyBits 768`）。

**StrictModes**

`StrictModes`指定 sshd 是否检查用户的一些重要文件和目录的权限。默认为`yes`（`StrictModes yes`），即对于用户的 SSH 配置文件、密钥文件和所在目录，SSH 要求拥有者必须是根用户或用户本人，用户组和其他人的写权限必须关闭。

**SyslogFacility**

`SyslogFacility`指定 Syslog 如何处理 sshd 的日志，默认是 Auth（`SyslogFacility AUTH`）。

**TCPKeepAlive**

`TCPKeepAlive`指定系统是否应向客户端发送 TCP keepalive 消息（`TCPKeepAlive yes`）。

**UseDNS**

`UseDNS`指定用户 SSH 登录一个域名时，服务器是否使用 DNS，确认该域名对应的 IP 地址包含本机（`UseDNS yes`）。打开该选项意义不大，而且如果 DNS 更新不及时，还有可能误判，建议关闭。

**UseLogin**

`UseLogin`指定用户认证内部是否使用`/usr/bin/login`替代 SSH 工具，默认为`no`（`UseLogin no`）。

**UserPrivilegeSeparation**

`UserPrivilegeSeparation`指定用户认证通过以后，使用另一个子线程处理用户权限相关的操作，这样有利于提高安全性。默认值为`yes`（`UsePrivilegeSeparation yes`）。

**VerboseMode**

SSH 2 版本专用，指定日志输出详细的 Debug 信息（`VerboseMode yes`）。

**X11Forwarding**

`X11Forwarding`指定是否打开 X window 的转发，默认值为 no（`X11Forwarding no`）。

修改配置文件以后，可以使用下面的命令验证，配置文件是否有语法错误。

```bash
$ sshd -t
```

新的配置文件生效，必须重启 sshd。

```bash
$ sudo systemctl restart sshd
```

## sshd 的命令行配置项

sshd 命令有一些配置项。这些配置项在调用时指定，可以覆盖配置文件的设置。

（1）`-d`

`-d`参数用于显示 debug 信息。

```bash
$ sshd -d
```

（2）`-D`

`-D`参数指定 sshd 不作为后台守护进程运行。

```bash
$ sshd -D
```

（3）`-e`

`-e`参数将 sshd 写入系统日志 syslog 的内容导向标准错误（standard error）。

（4）`-f`

`-f`参数指定配置文件的位置。

（5）`-h`

`-h`参数用于指定密钥。

```bash
$ sshd -h /usr/local/ssh/my_rsa_key
```

（6）`-o`

`-o`参数指定配置文件的一个配置项和对应的值。

```bash
$ sshd -o "Port 2034"
```

配置项和对应值之间，可以使用等号。

```bash
$ sshd -o "Port = 2034"
```

如果省略等号前后的空格，也可以不使用引号。

```bash
$ sshd -o Port=2034
```

`-o`参数可以多个一起使用，用来指定多个配置关键字。

（7）`-p`

`-p`参数指定 sshd 的服务端口。

```bash
$ sshd -p 2034
```

上面命令指定 sshd 在`2034`端口启动。

`-p`参数可以指定多个端口。

```bash
$ sshd -p 2222 -p 3333
```

（8）`-t`

`-t`参数检查配置文件的语法是否正确。

