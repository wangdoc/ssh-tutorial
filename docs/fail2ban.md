# Fail2Ban 教程

## 简介

Fail2Ban 是一个 Linux 系统的应用软件，用来防止系统入侵，主要是防止暴力破解系统密码。它是用 Python 开发的。

它主要通过监控日志文件（比如`/var/log/auth.log`、`/var/log/apache/access.log`等）来生效。一旦发现恶意攻击的登录请求，它会封锁对方的 IP 地址，使得对方无法再发起请求。

Fail2Ban 可以防止有人反复尝试 SSH 密码登录，但是如果 SSH 采用的是密钥登录，禁止了密码登录，就不需要 Fail2Ban 来保护。

Fail2Ban 的安装命令如下。

```bash
# ubuntu & Debian
$ sudo apt install fail2ban

# Fedora
$ sudo dnf install epel-release
$ sudo dnf install fail2ban

# Centos & Red hat
$ yum install fail2ban
```

安装后，使用下面的命令查看 Fail2Ban 的状态。

```bash
$ systemctl status fail2ban.service
```

如果没有启动，就启动 Fail2Ban。

```bash
$ sudo systemctl start fail2ban
```

重新启动 Fail2Ban。

```bash
$ sudo systemctl restart fail2ban
```

设置 Fail2Ban 重启后自动运行。

```bash
$ sudo systemctl enable fail2ban
```

## fail2ban-client

Fail2Ban 自带一个客户端 fail2ban-client，用来操作 Fail2Ban。

```bash
$ fail2ban-client
```

上面的命令会输出 fail2ban-client 所有的用法。

下面的命令查看激活的监控目标。

```bash
$ fail2ban-client status

Status
|- Number of jail:	1
`- Jail list:	sshd
```

下面的命令查看某个监控目标（这里是 sshd）的运行情况。

```bash
$ sudo fail2ban-client status sshd

Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  |- Total failed:     9
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   0.0.0.0
```

下面的命令输出一个简要的版本，包括所有监控目标被封的 IP 地址。

```bash
$ sudo fail2ban-client banned
[{'sshd': ['192.168.100.50']}, {'apache-auth': []}]
```

下面的命令可以解封某个 IP 地址。

```bash
$ sudo fail2ban-client set sshd unbanip 192.168.1.69
```

## 配置

### 主配置文件

Fail2Ban 主配置文件是在`/etc/fail2ban/fail2ban.conf`，可以新建一份副本`/etc/fail2ban/fail2ban.local`，修改都针对副本。

```bash
$ sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
```

下面是设置 Fail2Ban 的日志位置。

```bash
[Definition]
logtarget = /var/log/fail2ban/fail2ban.log
```

修改配置以后，需要重新启动`fail2ban.service`，让其生效。

### 封禁配置

Fail2Ban 封禁行为的配置文件是`/etc/fail2ban/jail.conf`。为了便于修改，可以把它复制一份`/etc/fail2ban/jail.local`，后面的修改都针对`jail.local`这个文件。

```bash
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

你也可以在目录`/etc/fail2ban/jail.d`里面，新建单独的子配置文件，比如`/etc/fail2ban/jail.d/sshd.local`。

同样地，修改配置以后，需要重新启动`fail2ban.service`，让其生效。

配置文件里面，`[DEFAULT]`标题行表示对于所有封禁目标生效。举例来说，如果封禁时间修改为1天，`/etc/fail2ban/jail.local`里面可以写成：

```bash
[DEFAULT]
bantime = 1d
```

如果某人被封时，对站长发送邮件通知，可以如下设置。

```bash
[DEFAULT]
destemail = yourname@example.com
sender = yourname@example.com

# to ban & send an e-mail with whois report to the destemail.
action = %(action_mw)s

# same as action_mw but also send relevant log lines
#action = %(action_mwl)s
```

如果配置写在其他标题行下，就表示只对该封禁目标生效，比如写在`[sshd]`下面，就表示只对 sshd 生效。

默认情况下，Fail2Ban 对各种服务都是关闭的，如果要针对某一项服务开启，需要在配置文件里面声明。

```bash
[sshd]
enabled = true
```

上面声明表示，Fail2Ban 对 sshd 开启。

### 配置项

下面是配置文件`jail.local`的配置项含义，所有配置项的格式都是`key=value`。

（1）bantime

封禁的时间长度，单位`m`表示分钟，`d`表示天，如果不写单位，则表示秒。Fail2Ban 默认封禁10分钟（10m 或 600）。

```bash
[DEFAULT]
bantime = 10m
```

（2）findtime

登录失败计算的时间长度，单位`m`表示分钟，`d`表示天，如果不写单位，则表示秒。Fail2Ban 默认封禁 10 分钟内登录 5 次失败的客户端。

```bash
[DEFAULT]

findtime = 10m
maxretry = 5
```

（3）maxretry

尝试登录的最大失败次数。

（4）destemail

接受通知的邮件地址。

```bash
[DEFAULT]
destemail = root@localhost
sender = root@<fq-hostname>
mta = sendmail
```

（5）sendername

通知邮件的“发件人”字段的值。

（6）mta

发送邮件的邮件服务，默认是`sendmail`。

（7）action

封禁时采取的动作。

```bash
[DEFAULT]
action = $(action_)s
```

上面的`action_`是默认动作，表示拒绝封禁对象的流量，直到封禁期结束。

下面是 Fail2Ban 提供的一些其他动作。

```bash
# ban & send an e-mail with whois report to the destemail.
action_mw = %(action_)s
            %(mta)s-whois[sender="%(sender)s", dest="%(destemail)s", protocol="%(protocol)s", chain="%(chain)s"]

# ban & send an e-mail with whois report and relevant log lines
# to the destemail.
action_mwl = %(action_)s
             %(mta)s-whois-lines[sender="%(sender)s", dest="%(destemail)s", logpath="%(logpath)s", chain="%(chain)s"]

# See the IMPORTANT note in action.d/xarf-login-attack for when to use this action
#
# ban & send a xarf e-mail to abuse contact of IP address and include relevant log lines
# to the destemail.
action_xarf = %(action_)s
             xarf-login-attack[service=%(__name__)s, sender="%(sender)s", logpath="%(logpath)s", port="%(port)s"]

# ban IP on CloudFlare & send an e-mail with whois report and relevant log lines
# to the destemail.
action_cf_mwl = cloudflare[cfuser="%(cfemail)s", cftoken="%(cfapikey)s"]
                %(mta)s-whois-lines[sender="%(sender)s", dest="%(destemail)s", logpath="%(logpath)s", chain="%(chain)s"]
```

（8）ignoreip

Fail2Ban 可以忽视的可信 IP 地址。多个 IP 地址之间使用空格分隔。

```bash
ignoreip = 127.0.0.1/8 192.168.1.10 192.168.1.20
```

（9）port

 指定要监控的端口。可以设为任何端口号或服务名称，比如`ssh`、`22`、`2200`等。

### ssh 配置

下面是 sshd 的设置范例。

```bash
[sshd]
enabled   = true
port = ssh
filter    = sshd
banaction = iptables
backend   = systemd
maxretry  = 5
findtime  = 1d
bantime   = 2w
ignoreip  = 127.0.0.1/8
```

首先需要注意，为了让 Fail2Ban 能够完整发挥作用，最好在`/etc/ssh/sshd_config`里面设置`LogLevel VERBOSE`，保证日志有足够的信息。

