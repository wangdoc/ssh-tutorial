# SSH 日志

SSH 在服务器端可以生成日志，记录登录当前服务器的情况。

SSH 日志是写在系统日志当中的，查看的时候需要从系统日志里面找到跟 SSH 相关的记录。

## journalctl 命令

如果系统使用 Systemd，可以使用`journalctl`命令查看日志。

```bash
$ journalctl -u ssh

Mar 25 20:25:36 web0 sshd[14144]: Accepted publickey for ubuntu from 10.103.160.144 port 59200 ssh2: RSA SHA256:l/zFNib1vJ+64nxLB4N9KaVhBEMf8arbWGxHQg01SW8
Mar 25 20:25:36 web0 sshd[14144]: pam_unix(sshd:session): session opened for user ubuntu by (uid=0)
Mar 25 20:39:12 web0 sshd[14885]: pam_unix(sshd:session): session closed for user ubuntu
...
```

上面示例中，返回的日志每一行就是一次登录尝试，按照从早到晚的顺序，其中包含了登录失败的尝试。`-u`参数是 Unit 单元的意思，`-u ssh`就是查看 SSH 单元，有的发行版需要写成`-u sshd`。

`-b0`参数可以查看自从上次登录后的日志。

```bash
$ journalctl -t ssh -b0
```

`-r`参数表示逆序输出，最新的在前面。

```bash
$ journalctl -t ssh -b0 -r
```

`since`和`until`参数可以指定日志的时间范围。

```bash
$ journalctl -u ssh --since yesterday # 查看昨天的日志
$ journalctl -u ssh --since -3d --until -2d # 查看三天前的日志
$ journalctl -u ssh --since -1h # 查看上个小时的日志
$ journalctl -u ssh --until "2022-03-12 07:00:00" # 查看截至到某个时间点的日志
```

下面的命令查看实时日志。

```bash
$ journalctl -fu ssh
```

## 其他命令

如果系统没有使用 Systemd，可以在`/var/log/auth.log`文件中找到 sshd 的日志。

```bash
$ sudo grep sshd /var/log/auth.log
```

下面的命令查看最后 500 行里面的 sshd 条目。

```bash
$ sudo tail -n 500 /var/log/auth.log | grep sshd
```

`-f`参数可以实时跟踪日志。

```bash
$ sudo tail -f -n 500 /var/log/auth.log | grep sshd
```

如果只是想看谁登录了系统，而不是深入查看所有细节，可以使用`lastlog`命令。

```bash
$ lastlog
```

## 日志设置

sshd 的配置文件`/etc/ssh/sshd_config`，可以调整日志级别。

```
LogLevel VERBOSE
```

如果为了调试，可以将日志调整为 DEBUG。

```
 LogLevel DEBUG
```

