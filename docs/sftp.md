# sftp 命令

`sftp`是 SSH 提供的一个客户端应用程序，主要用来安全地访问 FTP。因为 FTP 是不加密协议，很不安全，`sftp`就相当于将 FTP 放入了 SSH。

下面的命令连接 FTP 主机。

```bash
$ sftp username@hostname
```

执行上面的命令，会要求输入 FTP 的密码。密码验证成功以后，就会出现 FTP 的提示符`sftp> `，下面是一个例子。

```bash
$ sftp USER@penguin.example.com
USER@penguin.example.com's password:
Connected to penguin.example.com.
sftp>
```

FTP 的提示符下面，就可以输入各种 FTP 命令了，这部分完全跟传统的 FTP 用法完全一样。

- `ls [directory]`：列出一个远程目录的内容。如果没有指定目标目录，则默认列出当前目录。
- `cd directory`：从当前目录改到指定目录。
- `mkdir directory`：创建一个远程目录。
- `rmdir path`：删除一个远程目录。
- `put localfile [remotefile]`：本地文件传输到远程主机。
- `get remotefile [localfile]`：远程文件传输到本地。
- `help`：显示帮助信息。
- `bye`：退出 sftp。
- `quit`：退出 sftp。
- `exit`：退出 sftp。

