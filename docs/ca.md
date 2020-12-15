# SSH 证书登录

SSH 是服务器登录工具，一般情况下都采用密码登录或密钥登录。

但是，SSH 还有第三种登录方法，那就是证书登录。某些情况下，它是更合理、更安全的登录方法，本文就介绍这种登录方法。

## 非证书登录的缺点

密码登录和密钥登录，都有各自的缺点。

密码登录需要输入服务器密码，这非常麻烦，也不安全，存在被暴力破解的风险。

密钥登录需要服务器保存用户的公钥，也需要用户保存服务器公钥的指纹。这对于多用户、多服务器的大型机构很不方便，如果有员工离职，需要将他的公钥从每台服务器删除。

## 证书登录是什么？

证书登录就是为了解决上面的缺点而设计的。它引入了一个证书颁发机构（Certificate Authority，简称 CA），对信任的服务器颁发服务器证书，对信任的用户颁发用户证书。

登录时，用户和服务器不需要提前知道彼此的公钥，只需要交换各自的证书，验证是否可信即可。

证书登录的主要优点有两个：（1）用户和服务器不用交换公钥，这更容易管理，也具有更好的可扩展性。（2）证书可以设置到期时间，而公钥没有到期时间。针对不同的情况，可以设置有效期很短的证书，进一步提高安全性。

## 证书登录的流程

SSH 证书登录之前，如果还没有证书，需要生成证书。具体方法是：（1）用户和服务器都将自己的公钥，发给 CA；（2）CA 使用服务器公钥，生成服务器证书，发给服务器；（3）CA 使用用户的公钥，生成用户证书，发给用户。

有了证书以后，用户就可以登录服务器了。整个过程都是 SSH 自动处理，用户无感知。

第一步，用户登录服务器时，SSH 自动将用户证书发给服务器。

第二步，服务器检查用户证书是否有效，以及是否由可信的 CA 颁发。证实以后，就可以信任用户。

第三步，SSH 自动将服务器证书发给用户。

第四步，用户检查服务器证书是否有效，以及是否由信任的 CA 颁发。证实以后，就可以信任服务器。

第五步，双方建立连接，服务器允许用户登录。

## 生成 CA 的密钥

证书登录的前提是，必须有一个 CA，而 CA 本质上就是一对密钥，跟其他密钥没有不同，CA 就用这对密钥去签发证书。

虽然 CA 可以用同一对密钥签发用户证书和服务器证书，但是出于安全性和灵活性，最好用不同的密钥分别签发。所以，CA 至少需要两对密钥，一对是签发用户证书的密钥，假设叫做`user_ca`，另一对是签发服务器证书的密钥，假设叫做`host_ca`。

使用下面的命令，生成`user_ca`。

```bash
# 生成 CA 签发用户证书的密钥
$ ssh-keygen -t rsa -b 4096 -f ~/.ssh/user_ca -C user_ca
```

上面的命令会在`~/.ssh`目录生成一对密钥：`user_ca`（私钥）和`user_ca.pub`（公钥）。

这个命令的各个参数含义如下。

- `-t rsa`：指定密钥算法 RSA。
- `-b 4096`：指定密钥的位数是4096位。安全性要求不高的场合，这个值可以小一点，但是不应小于1024。
- `-f ~/.ssh/user_ca`：指定生成密钥的位置和文件名。
- `-C user_ca`：指定密钥的识别字符串，相当于注释，可以随意设置。

使用下面的命令，生成`host_ca`。

```bash
# 生成 CA 签发服务器证书的密钥
$ ssh-keygen -t rsa -b 4096 -f host_ca -C host_ca
```

上面的命令会在`~/.ssh`目录生成一对密钥：`host_ca`（私钥）和`host_ca.pub`（公钥）。

现在，`~/.ssh`目录应该至少有四把密钥。

- `~/.ssh/user_ca`
- `~/.ssh/user_ca.pub`
- `~/.ssh/host_ca`
- `~/.ssh/host_ca.pub`

## CA 签发服务器证书

有了 CA 以后，就可以签发服务器证书了。

签发证书，除了 CA 的密钥以外，还需要服务器的公钥。一般来说，SSH 服务器（通常是`sshd`）安装时，已经生成密钥`/etc/ssh/ssh_host_rsa_key`了。如果没有的话，可以用下面的命令生成。

```bash
$ sudo ssh-keygen -f /etc/ssh/ssh_host_rsa_key -b 4096 -t rsa
```

上面命令会在`/etc/ssh`目录，生成`ssh_host_rsa_key`（私钥）和`ssh_host_rsa_key.pub`（公钥）。然后，需要把服务器公钥`ssh_host_rsa_key.pub`，复制或上传到 CA 所在的服务器。

上传以后，CA 就可以使用密钥`host_ca`为服务器的公钥`ssh_host_rsa_key.pub`签发服务器证书。

```bash
$ ssh-keygen -s host_ca -I host.example.com -h -n host.example.com -V +52w ssh_host_rsa_key.pub
```

上面的命令会生成服务器证书`ssh_host_rsa_key-cert.pub`（服务器公钥名字加后缀`-cert`）。这个命令各个参数的含义如下。

- `-s`：指定 CA 签发证书的密钥。
- `-I`：身份字符串，可以随便设置，相当于注释，方便区分证书，将来可以使用这个字符串撤销证书。
- `-h`：指定该证书是服务器证书，而不是用户证书。
- `-n host.example.com`：指定服务器的域名，表示证书仅对该域名有效。如果有多个域名，则使用逗号分隔。用户登录该域名服务器时，SSH 通过证书的这个值，分辨应该使用哪张证书发给用户，用来证明服务器的可信性。
- `-V +52w`：指定证书的有效期，这里为52周（一年）。默认情况下，证书是永远有效的。建议使用该参数指定有效期，并且有效期最好短一点，最长不超过52周。
- `ssh_host_rsa_key.pub`：服务器公钥。

生成证书以后，可以使用下面的命令，查看证书的细节。

```bash
$ ssh-keygen -L -f ssh_host_rsa_key-cert.pub
```

最后，为证书设置权限。

```bash
$ chmod 600 ssh_host_rsa_key-cert.pub
```

## CA 签发用户证书

下面，再用 CA 签发用户证书。这时需要用户的公钥，如果没有的话，客户端可以用下面的命令生成一对密钥。

```bash
$ ssh-keygen -f ~/.ssh/user_key -b 4096 -t rsa
```

上面命令会在`~/.ssh`目录，生成`user_key`（私钥）和`user_key.pub`（公钥）。

然后，将用户公钥`user_key.pub`，上传或复制到 CA 服务器。接下来，就可以使用 CA 的密钥`user_ca`为用户公钥`user_key.pub`签发用户证书。

```bash
$ ssh-keygen -s user_ca -I user@example.com -n user -V +1d user_key.pub
```

上面的命令会生成用户证书`user_key-cert.pub`（用户公钥名字加后缀`-cert`）。这个命令各个参数的含义如下。

- `-s`：指定 CA 签发证书的密钥
- `-I`：身份字符串，可以随便设置，相当于注释，方便区分证书，将来可以使用这个字符串撤销证书。
- `-n user`：指定用户名，表示证书仅对该用户名有效。如果有多个用户名，使用逗号分隔。用户以该用户名登录服务器时，SSH 通过这个值，分辨应该使用哪张证书，证明自己的身份，发给服务器。
- `-V +1d`：指定证书的有效期，这里为1天，强制用户每天都申请一次证书，提高安全性。默认情况下，证书是永远有效的。
- `user_key.pub`：用户公钥。

生成证书以后，可以使用下面的命令，查看证书的细节。

```bash
$ ssh-keygen -L -f user_key-cert.pub
```

最后，为证书设置权限。

```bash
$ chmod 600 user_key-cert.pub
```

## 服务器安装证书

CA 生成服务器证书`ssh_host_rsa_key-cert.pub`以后，需要将该证书发回服务器，可以使用下面的`scp`命令，将证书拷贝过去。

```bash
$ scp ~/.ssh/ssh_host_rsa_key-cert.pub root@host.example.com:/etc/ssh/
```

然后，将下面一行添加到服务器配置文件`/etc/ssh/sshd_config`。

```bash
HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub
```

上面的代码告诉 sshd，服务器证书是哪一个文件。

重新启动 sshd。

```bash
$ sudo systemctl restart sshd.service
# 或者
$ sudo service sshd restart
```

## 服务器安装 CA 公钥

为了让服务器信任用户证书，必须将 CA 签发用户证书的公钥`user_ca.pub`，拷贝到服务器。

```bash
$ scp ~/.ssh/user_ca.pub root@host.example.com:/etc/ssh/
```

上面的命令，将 CA 签发用户证书的公钥`user_ca.pub`，拷贝到 SSH 服务器的`/etc/ssh`目录。

然后，将下面一行添加到服务器配置文件`/etc/ssh/sshd_config`。

```bash
TrustedUserCAKeys /etc/ssh/user_ca.pub
```

上面的做法是将`user_ca.pub`加到`/etc/ssh/sshd_config`，这会产生全局效果，即服务器的所有账户都会信任`user_ca`签发的所有用户证书。

另一种做法是将`user_ca.pub`加到服务器某个账户的`~/.ssh/authorized_keys`文件，只让该账户信任`user_ca`签发的用户证书。具体方法是打开`~/.ssh/authorized_keys`，追加一行，开头是`@cert-authority principals="..."`，然后后面加上`user_ca.pub`的内容，大概是下面这个样子。

```bash
@cert-authority principals="user" ssh-rsa AAAAB3Nz...XNRM1EX2gQ==
```

上面代码中，`principals="user"`指定用户登录的服务器账户名，一般就是`authorized_keys`文件所在的账户。

重新启动 sshd。

```bash
$ sudo systemctl restart sshd.service
# 或者
$ sudo service sshd restart
```

至此，SSH 服务器已配置为信任`user_ca`签发的证书。

## 客户端安装证书

客户端安装用户证书很简单，就是从 CA 将用户证书`user_key-cert.pub`复制到客户端，与用户的密钥`user_key`保存在同一个目录即可。

## 客户端安装 CA 公钥

为了让客户端信任服务器证书，必须将 CA 签发服务器证书的公钥`host_ca.pub`，加到客户端的`/etc/ssh/ssh_known_hosts`文件（全局级别）或者`~/.ssh/known_hosts`文件（用户级别）。

具体做法是打开`ssh_known_hosts`或`known_hosts`文件，追加一行，开头为`@cert-authority *.example.com`，然后将`host_ca.pub`文件的内容（即公钥）粘贴在后面，大概是下面这个样子。

```bash
@cert-authority *.example.com ssh-rsa AAAAB3Nz...XNRM1EX2gQ==
```

上面代码中，`*.example.com`是域名的模式匹配，表示只要服务器符合该模式的域名，且签发服务器证书的 CA 匹配后面给出的公钥，就都可以信任。如果没有域名限制，这里可以写成`*`。如果有多个域名模式，可以使用逗号分隔；如果服务器没有域名，可以用主机名（比如`host1,host2,host3`）或者 IP 地址（比如`11.12.13.14,21.22.23.24`）。

然后，就可以使用证书，登录远程服务器了。

```bash
$ ssh -i ~/.ssh/user_key user@host.example.com
```

上面命令的`-i`参数用来指定用户的密钥。如果证书与密钥在同一个目录，则连接服务器时将自动使用该证书。

## 废除证书

废除证书的操作，分成用户证书的废除和服务器证书的废除两种。

服务器证书的废除，用户需要在`known_hosts`文件里面，修改或删除对应的`@cert-authority`命令的那一行。

用户证书的废除，需要在服务器新建一个`/etc/ssh/revoked_keys`文件，然后在配置文件`sshd_config`添加一行，内容如下。

```bash
RevokedKeys /etc/ssh/revoked_keys
```

`revoked_keys`文件保存不再信任的用户公钥，由下面的命令生成。

```bash
$ ssh-keygen -kf /etc/ssh/revoked_keys -z 1 ~/.ssh/user1_key.pub
```

上面命令中，`-z`参数用来指定用户公钥保存在`revoked_keys`文件的哪一行，这个例子是保存在第1行。

如果以后需要废除其他的用户公钥，可以用下面的命令保存在第2行。

```bash
$ ssh-keygen -ukf /etc/ssh/revoked_keys -z 2 ~/.ssh/user2_key.pub
```

## 参考链接

- [SSH Emergency Access](https://smallstep.com/blog/ssh-emergency-access/), Carl Tashian
- [Using OpenSSH Certificate Authentication](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/sec-using_openssh_certificate_authentication), Red Hat Enterprise Linux Deployment Guide
- [How to SSH Properly](https://gravitational.com/blog/how-to-ssh-properly/), Gus Luxton

