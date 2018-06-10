每天一个Linux命令系列

细系统的学习linux常用命令，坚持每天一个命令，主要参考"每天一个Linux系列"，”开源世界之旅“，”鸟叔的私房菜“，加入自己的理解。

## GUI? CLI?

### 定义

**GUI**，Graphical User Interface，图形用户界面。用户界面的所有元素图形化，主要使用鼠标作为输入工具，点击图标执行程序，使用按钮、菜单、对话框等进行交互，追求易用，看起来比较美

**CLI**，Command Line Interface，命令行界面。用户界面字符化，使用键盘作为输入工具，输入命令、选项、参数执行程序，追求高效，看起来比较酷

### 缩写习惯

最常见的缩写，取每个单词的首字母，如

| cd   | Change Directory            |
| ---- | --------------------------- |
| dd   | Disk Dump                   |
| df   | Disk Free                   |
| du   | Disk Usage                  |
| pwd  | Print Working Directory     |
| ps   | Processes Status            |
| PS   | Prompt Strings              |
| su   | Substitute User             |
| rc   | Run Command                 |
| Tcl  | Tool Command Language       |
| cups | Common Unix Printing System |
| apt  | Advanced Packaging Tool     |
| bg   | BackGround                  |
| ping | Packet InterNet Grouper     |

如果首字母后为“h”，通常保留

| chsh  | CHange SHell       |
| ----- | ------------------ |
| chmod | CHange MODe        |
| chown | CHange OWNer       |
| chgrp | CHange GRouP       |
| bash  | Bourne Again SHell |
| zsh   | Z SHell            |
| ksh   | Korn SHell         |
| ssh   | Secure SHell       |

如果只有一个单词，通常取每个音节的首字母：

| cp   | CoPy   |
| ---- | ------ |
| ln   | LiNk   |
| ls   | LiSt   |
| mv   | MoVe   |
| rm   | ReMove |

对于目录，通常使用前几个字母作为缩写：

| bin  | BINaries              |
| ---- | --------------------- |
| dev  | DEVices               |
| etc  | ETCetera              |
| lib  | LIBrary               |
| var  | VARiable              |
| proc | PROCesses             |
| sbin | Superuser BINaries    |
| tmp  | TeMPorary             |
| usr  | Unix Shared Resources |

这种缩写的其它情况

| diff   | DIFFerences        |
| ------ | ------------------ |
| cal    | CALendar           |
| cat    | CATenate           |
| ed     | EDitor             |
| exec   | EXECute            |
| tab    | TABle              |
| regexp | REGular EXPression |

## 命令选项，从a到z

Linux 命令的选项繁复庞杂，让人眼花缭乱。不过这些选项往往具有相对固定的涵义，熟悉了它们，记忆便不再困难

- -a

  all : 全部，所有 (ls , lsattr , uname)archive : 存档 (cp , rsync)append : 附加 (tar -A , 7z)

- -b

  blocksize : 块大小，带参数 (du , df)batch : 批处理模式 (交互模式的程序通常拥有此选项，如 top -b)

- -c

  commands : 执行命令，带参数 (bash , ksh , python)create : 创建 (tar)

- -d

  debug : 调试delete : 删除directory : 目录 (ls)

- -e

  execute : 执行，带参数 (xterm , perl)edit : 编辑exclude : 排除

- -f

  force : 强制，不经确认(cp , rm ,mv)file : 文件，带参数 (tar)configuration file : 指定配置文件(有些守护进程拥有此选项，如 ssh , lighttpd)

- -h

  --help : 帮助human readable : 人性化显示(ls , du , df)headers : 头部

- -i

  interactive : 交互模式，提示(rm , mv)include : 包含

- -k

  keep : 保留kill

- -l

  long listing format : 长格式(ls)list : 列表load : 读取 (gcc , emacs)

- -m

  message : 消息 (cvs)manual : 手册 (whereis)create home : 创建 home 目录 (usermod , useradd)

- -n

  number : 行号、编号 (cat , head , tail , pstree , lspci)no : (useradd , make)

- -o

  output : 输出 (cc , sort)options : 选项 (mount)

- -p

  port : 端口，带参数 (很多网络工具拥有此选项，如 ssh , lftp )protocol : 协议，带参数passwd : 密码，带参数

- -q

  quiet : 静默

- -r

  reverse : 反转recursive : 递归 (cp , rm , chmod -R)

- -s

  silent : 安静size : 大小，带参数subject

- -t

  tagtype : 类型 (mount)

- -u

  user : 用户名、UID，带参数

- -v

  verbose : 冗长version : 版本

- -w

  width : 宽度warning : 警告

- -x

  exclude : 排除 (tar , zip)

- -y

  yes

- -z

  zip : 启用压缩 (bzip , tar , zcat , zip , cvs)

## let's begin

[每天一个Linux命令](http://www.cnblogs.com/peida/archive/2012/12/05/2803591.html)