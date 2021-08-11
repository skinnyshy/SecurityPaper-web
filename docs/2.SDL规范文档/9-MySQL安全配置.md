
# 10. MySQL安全配置

作者：Jun6l3

协作：

---

数据库作为数据管理的平台，它的安全性首先由系统的内部安全和网络安全两部分来决定。对于系统管理员来说，首先要保证系统本身的安全，在安装MySQL数据库时，需要对基础环境进行较好的配置。

## 修改root用户口令，删除空口令

缺省安装的MySQL的root用户是空密码的，为了安全起见，必须修改为强密码，所谓的强密码，至少8位，由字母、数字和符号组成的不规律密码。使用MySQL自带的命令mysaladmin修改root密码，同时也可以登陆数据库，修改数据库mysql下的user表的字段内容，修改方法如下所示：

```shell
# /usr/local/mysql/bin/mysqladmin -u root password “upassword” //使用mysqladmin
#mysql> use mysql;
#mysql> update user set password=password('upassword') where user='root';
#mysql> flush privileges; //强制刷新内存授权表，否则用的还是在内存缓冲的口令
```

## 删除默认数据库和数据库用户

一般情况下，MySQL数据库安装在本地，并且也只需要本地的php脚本对mysql进行读取，所以很多用户不需要，尤其是默认安装的用户。MySQL初始化后会自动生成空用户和test库，进行安装的测试，这会对数据库的安全构成威胁，有必要全部删除，最后的状态只保留单个root即可，当然以后根据需要增加用户和数据库。

```shell
#mysql> show databases;
#mysql> drop database test; //删除数据库test
#use mysql;
#delete from db; //删除存放数据库的表信息，因为还没有数据库信息。
#mysql> delete from user where not (user='root') ; // 删除初始非root的用户
#mysql> delete from user where user='root' and password=''; //删除空密码的root，尽量重复操作
Query OK, 2 rows affected (0.00 sec)
#mysql> flush privileges; //强制刷新内存授权表。
```

## 改变默认mysql管理员帐号

系统mysql的管理员名称是root，而一般情况下，数据库管理员都没进行修改，这一定程度上对系统用户穷举的恶意行为提供了便利，此时修改为复杂的用户名，请不要在设定为admin或者administraror的形式，因为它们也在易猜的用户字典中。

```shell
mysql> update user set user="newroot" where user="root"; //改成不易被猜测的用户名
mysql> flush privileges;
```

## 关于密码的管理

密码是数据库安全管理的一个很重要因素，不要将纯文本密码保存到数据库中。如果你的计算机有安全危险，入侵者可以获得所有的密码并使用它们。相反，应使用MD5()、SHA1()或单向哈希函数。也不要从词典中选择密码，有专门的程序可以破解它们，请选用至少八位，由字母、数字和符号组成的强密码。在存取密码时，使用mysql的内置函数password（）的sql语句，对密码进行加密后存储。例如以下方式在users表中加入新用户。
```shell
#mysql> insert into users values (1,password(1234),'test');
```

### 使用独立用户运行msyql

绝对不要作为使用root用户运行MySQL服务器。这样做非常危险，因为任何具有FILE权限的用户能够用root创建文件(例如，~root/.bashrc)。mysqld拒绝使用root运行，除非使用–user=root选项明显指定。应该用普通非特权用户运行mysqld。正如前面的安装过程一样，为数据库建立独立的linux中的mysql账户，该账户用来只用于管理和运行MySQL。

要想用其它Unix用户启动mysqld，增加user选项指定`/etc/my.cnf`选项文件或服务器数据目录的`my.cnf`选项文件中的`[mysqld]`组的用户名。

```shell
#vim /etc/my.cnf
[mysqld]
user=mysql
```

该命令使服务器用指定的用户来启动，无论你手动启动或通过`mysqld_safe`或`mysql.server`启动，都能确保使用mysql的身份。也可以在启动数据库是，加上user参数。
```shell
# /usr/local/mysql/bin/mysqld_safe --user=mysql &
```

作为其它linux用户而不用root运行mysqld，你不需要更改user表中的root用户名，因为MySQL账户的用户名与linux账户的用户名无关。确保mysqld运行时，只使用对数据库目录具有读或写权限的linux用户来运行。

## 禁止远程连接数据库

在命令行netstat -ant下看到，默认的3306端口是打开的，此时打开了mysqld的网络监听，允许用户远程通过帐号密码连接数本地据库，默认情况是允许远程连接数据的。为了禁止该功能，启动skip-networking，不监听sql的任何TCP/IP的连接，切断远程访问的权利，保证安全性。假如需要远程管理数据库，可通过安装PhpMyadmin来实现。假如确实需要远程连接数据库，至少修改默认的监听端口，同时添加防火墙规则，只允许可信任的网络的mysql监听端口的数据通过。

```shell
# vim /etc/my.cf
将#skip-networking注释去掉。
# /usr/local/mysql/bin/mysqladmin -u root -p shutdown //停止数据库
#/usr/local/mysql/bin/mysqld_safe --user=mysql & //后台用mysql用户启动mysql
```

## 限制连接用户的数量

数据库的某用户多次远程连接，会导致性能的下降和影响其他用户的操作，有必要对其进行限制。可以通过限制单个账户允许的连接数量来实现，设置`my.cnf`文件的mysqld中的`max_user_connections`变量来完成。GRANT语句也可以支持 资源控制选项来限制服务器对一个账户允许的使用范围。

```shell
#vim /etc/my.cnf
[mysqld]
max_user_connections 2
```

## 用户目录权限限制

默认的mysql是安装在/usr/local/mysql，而对应的数据库文件在/usr/local/mysql/var目录下，因此，必须保证该目录不能让未经授权的用户访问后把数据库打包拷贝走了，所以要限制对该目录的访问。确保mysqld运行时，只使用对数据库目录具有读或写权限的linux用户来运行。

```shell
# chown -R root  /usr/local/mysql/  //mysql主目录给root
# chown -R mysql.mysql /usr/local/mysql/var //确保数据库目录权限所属mysql用户
```

## 命令历史记录保护

数据库相关的shell操作命令都会分别记录在.bash_history，如果这些文件不慎被读取，会导致数据库密码和数据库结构等信息泄露，而登陆数据库后的操作将记录在.mysql_history文件中，如果使用update表信息来修改数据库用户密码的话，也会被读取密码，因此需要删除这两个文件，同时在进行登陆或备份数据库等与密码相关操作时，应该使用-p参数加入提示输入密码后，隐式输入密码，建议将以上文件置空。

```shell
# rm .bash_history .mysql_history  //删除历史记录
# ln -s /dev/null .bash_history   //将shell记录文件置空
# ln -s /dev/null .mysql_history  //将mysql记录文件置空
```

## 禁止MySQL对本地文件存取

在mysql中，提供对本地文件的读取，使用的是load data local infile命令，默认在5.0版本中，该选项是默认打开的，该操作令会利用MySQL把本地文件读到数据库中，然后用户就可以非法获取敏感信息了，假如你不需要读取本地文件，请务必关闭。

测试：首先在测试数据库下建立sqlfile.txt文件，用逗号隔开各个字段

```shell
# vi sqlfile.txt
1,sszng,111
2,sman,222
#mysql> load data local infile 'sqlfile.txt' into table users fields terminated by ','; //读入数据
#mysql> select * from users;

+--------+------------+----------+
| userid  | username   | password |
+--------+------------+----------+
|      1 | sszng       | 111      |
|      2 | sman        | 222      |
+--------+------------+----------+
```

成功的将本地数据插入数据中，此时应该禁止MySQL中用`“LOAD DATA LOCAL INFILE”`命令。网络上流传的一些攻击方法中就有用它`LOAD DATA LOCAL INFILE`的，同时它也是很多新发现的`SQL Injection`攻击利用的手段！黑客还能通过使用`LOAD DATALOCAL INFILE`装载`“/etc/passwd”`进一个数据库表，然后能用SELECT显示它，这个操作对服务器的安全来说，是致命的。可以在my.cnf中添加`local-infile=0`，或者加参数`local-infile=0`启动mysql。

```shell
#/usr/local/mysql/bin/mysqld_safe --user=mysql --local-infile=0 &
#mysql> load data local infile 'sqlfile.txt' into table users fields terminated by ',';
#ERROR 1148 (42000): The used command is not allowed with this MySQL version
```

`--local-infile=0`选项启动mysqld从服务器端禁用所有LOAD DATA LOCAL命令，假如需要获取本地文件，需要打开，但是建议关闭。

## MySQL服务器权限控制
MySQL权限系统的主要功能是证实连接到一台给定主机的用户，并且赋予该用户在数据库上的SELECT、INSERT、UPDATE和DELETE等权限（详见user超级用户表）。它的附加的功能包括有匿名的用户并对于MySQL特定的功能例如LOAD DATA INFILE进行授权及管理操作的能力。

管理员可以对user，db，host等表进行配置，来控制用户的访问权限，而user表权限是超级用户权限。只把user表的权限授予超级用户如服务器或数据库主管是明智的。对其他用户，你应该把在user表中的权限设成’N’并且仅在特定数据库的基础上授权。你可以为特定的数据库、表或列授权，FILE权限给予你用LOAD DATA INFILE和SELECT … INTO OUTFILE语句读和写服务器上的文件，任何被授予FILE权限的用户都能读或写MySQL服务器能读或写的任何文件。(说明用户可以读任何数据库目录下的文件，因为服务器可以访问这些文件）。 FILE权限允许用户在MySQL服务器具有写权限的目录下创建新文件，但不能覆盖已有文件在user表的File_priv设置Y或N。，所以当你不需要对服务器文件读取时，请关闭该权限。

```shell
#mysql> load data infile 'sqlfile.txt' into table loadfile.users fields terminated by ',';
Query OK, 4 rows affected (0.00 sec) //读取本地信息sqlfile.txt'
Records: 4  Deleted: 0  Skipped: 0  Warnings: 0
#mysql> update user set File_priv='N' where user='root'; //禁止读取权限
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
mysql> flush privileges; //刷新授权表
Query OK, 0 rows affected (0.00 sec)
#mysql> load data infile 'sqlfile.txt' into table users fields terminated by ','; //重登陆读取文件
#ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES) //失败
# mysql> select * from loadfile.users into outfile 'test.txt' fields terminated by ',';
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

为了安全起见，随时使用**SHOW GRANTS**语句检查查看谁已经访问了什么。然后使用REVOKE语句删除不再需要的权限。

## 使用chroot方式来控制MySQL的运行目录

Chroot是linux中的一种系统高级保护手段，它的建立会将其与主系统几乎完全隔离，也就是说，一旦遭到什么问题，也不会危及到正在运行的主系统。这是一个非常有效的办法，特别是在配置网络服务程序的时候。

## 关闭对Web访问的支持

如果不打算让Web访问使用MySQL数据库，没有提供诸如PHP这样的Web语言的时候，重新设置或编译你的PHP，取消它们对MySQL的默认支持。假如服务器中使用php等web程序，试试用Web形式非法的请求，如果得到任何形式的MySQL错误，立即分析原因，及时修改Web程序，堵住漏洞，防止MySQL暴露在web面前。

对于Web的安全检查，在MySQL官方文档中这么建议，对于web应用，至少检查以下清单：

* 试试用Web形式输入单引号和双引号(''和"")。如果得到任何形式的MySQL错误，立即分析原因。

* 试试修改动态URL，可以在其中添加`%22(")`、`%23(#)`和`%27(')`。

* 试试在动态URL中修改数据类型，使用前面示例中的字符，包括数字和字符类型。你的应用程序应足够安全，可以防范此类修改和类似攻击。 

* 试试输入字符、空格和特殊符号，不要输入数值字段的数字。你的应用程序应在将它们传递到MySQL之前将它们删除或生成错误。将未经过检查的值传递给MySQL是很危险的！

* 将数据传给MySQL之前先检查其大小。

* 用管理账户之外的用户名将应用程序连接到数据库。不要给应用程序任何不需要的访问权限。

## 数据库备份策略
一般可采用本地备份和网络备份的形式，可采用MySQL本身自带的mysqldump的方式和直接复制备份形式，
直接拷贝数据文件最为直接、快速、方便，但缺点是基本上不能实现增量备份。为了保证数据的一致性，需要在备份文件前，执行以下 SQL 语句：`FLUSH TABLES WITH READ LOCK`；也就是把内存中的数据都刷新到磁盘中，同时锁定数据表，以保证拷贝过程中不会有新的数据写入。这种方法备份出来的数据恢复也很简单，直接拷贝回原来的数据库目录下即可。

使用mysqldump可以把整个数据库装载到一个单独的文本文件中。这个文件包含有所有重建您的数据库所需要的SQL命令。这个命令取得所有的模式（Schema，后面有解释）并且将其转换成DDL语法（CREATE语句，即数据库定义语句），取得所有的数据，并且从这些数据中创建INSERT语句。这个工具将您的数据库中所有的设计倒转。因为所有的东西都被包含到了一个文本文件中。这个文本文件可以用一个简单的批处理和一个合适SQL语句导回到MySQL中。

使用 mysqldump进行备份非常简单，如果要备份数据库” nagios_db_backup ”，使用命令，同时使用管道gzip命令对备份文件进行压缩，建议使用异地备份的形式，可以采用Rsync等方式，将备份服务器的目录挂载到数据库服务器，将数据库文件备份打包在，通过crontab定时备份数据：

```shell
#!/bin/sh
time=`date +"("%F")"%R`
$/usr/local/mysql/bin/mysqldump -u nagios -pnagios nagios | gzip >/home/sszheng/nfs58/nagiosbackup/nagios_backup.$time.gz
# crontab -l
# m h  dom mon dow   command
00 00 * * * /home/sszheng/shnagios/backup.sh

恢复数据使用命令：
gzip -d nagios_backup.\(2008-01-24\)00\:00.gz
nagios_backup.(2008-01-24)00:00
#mysql –u root -p nagios  <  /home/sszheng/nfs58/nagiosbackup/nagios_backup.\(2008-01-24\)12\:00
```

## 启用SSL连接
mysql默认未启用SSL连接，使用wireshakr抓包可以查看执行的SQL语句和执行结果，在站库分离、主从复制、主从同步等复杂网络下，导致数据库执行过程可能会被嗅探。
### 安装时启动SSL
在MySQL5.7安装初始化阶段，比之前版本多了一步操作，而这个操作就是安装SSL的。
```shell
shell> bin/mysqld --initialize --user=mysql    # MySQL 5.7.6 and up
shell> bin/mysql_ssl_rsa_setup                 # MySQL 5.7.6 and up
```
当运行完这个命令后，默认会在data_dir目录下生成以下pem文件，这些文件就是用于启用SSL功能的：
```shell
[root mysql_data]# ll *.pem
-rw------- 1 mysql mysql 1675 Jun 12 17:22 ca-key.pem         #CA私钥
-rw-r--r-- 1 mysql mysql 1074 Jun 12 17:22 ca.pem             #自签的CA证书，客户端连接也需要提供
-rw-r--r-- 1 mysql mysql 1078 Jun 12 17:22 client-cert.pem    #客户端连接服务器端需要提供的证书文件
-rw------- 1 mysql mysql 1675 Jun 12 17:22 client-key.pem     #客户端连接服务器端需要提供的私钥文件
-rw------- 1 mysql mysql 1675 Jun 12 17:22 private_key.pem    #私钥/公钥对的私有成员
-rw-r--r-- 1 mysql mysql 451 Jun 12 17:22  public_key.pem     #私钥/公钥对的共有成员
-rw-r--r-- 1 mysql mysql 1078 Jun 12 17:22 server-cert.pem    #服务器端证书文件
-rw------- 1 mysql mysql 1675 Jun 12 17:22 server-key.pem     #服务器端私钥文件
```
本地进入MySQL命令行，可以看到如下变量值：
```shell
root> mysql -h 10.126.xxx.xxx -udba -p
```
查看SSL开启情况
```shell
dba:(none)> show global variables like '%ssl%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| have_openssl  | YES             |
| have_ssl      | YES             |    #已经开启了SSL
| ssl_ca        | ca.pem          |
| ssl_capath    |                 |
| ssl_cert      | server-cert.pem |
| ssl_cipher    |                 |
| ssl_crl       |                 |
| ssl_crlpath   |                 |
| ssl_key       | server-key.pem  |
+---------------+-----------------+
```
查看dba连接的方式
```shell
dba:(none)> \s
--------------
/usr/local/mysql/bin/mysql  Ver 14.14 Distrib 5.7.18, for linux-glibc2.5 (x86_64) using  EditLine wrapper
Connection id:          2973
Current database:
Current user:           dba@10.126.xxx.xxx
SSL:                    Cipher in use is DHE-RSA-AES256-SHA #表示该dba用户是采用SSL连接到mysql服务器上的，如果不是ssl，那么会显示“Not in use“
Current pager:          more
Using outfile:          ''
Using delimiter:        ;
Server version:         5.7.18-log MySQL Community Server (GPL)
Protocol version:       10
Connection:             10.126.126.160 via TCP/IP
Server characterset:    utf8
Db     characterset:    utf8
Client characterset:    utf8
Conn.  characterset:    utf8
TCP port:               3306
Uptime:                 2 hours 35 min 48 sec
```
* 如果用户是采用本地localhost或者sock连接数据库，那么不会使用SSL方式了。

### 安装后启动SSL
* 关闭MySQL服务
* 运行mysql_ssl_rsa_setup 命令
* 到data_dir目录下修改.pem文件的所属权限用户为mysql
```shell
chown -R mysql.mysql *.pem
```
* 启动MySQL服务

### 强制某用户必须使用SSL连接数据库
修改已存在用户 
```shell
ALTER USER 'dba'@'%' REQUIRE SSL;
```
新建必须使用SSL用户
```shell
grant select on *.* to 'dba'@'%' identified by 'xxx' REQUIRE SSL;
```
对于上面强制使用ssl连接的用户，如果不是使用ssl连接的就会报错，像下面这样：
```shell
[root]# /usr/local/mysql/bin/mysql -udba -p -h10.126.xxx.xxx --ssl=0
Enter password: 
ERROR 1045 (28000): Access denied for user 'dba'@'10.126.xxx.xxx' (using password: YES)
```

## Mysqld安全相关启动选项

下列mysqld选项影响安全：

* `--allow-suspicious-udfs`

  该选项控制是否可以载入主函数只有xxx符的用户定义函数。默认情况下，该选项被关闭，并且只能载入至少有辅助符的UDF。这样可以防止从未包含合法UDF的共享对象文件载入函数。

* `--local-infile[={0|1}]`

  如果用–local-infile=0启动服务器，则客户端不能使用LOCAL in LOAD DATA语句。

* `--old-passwords`

  强制服务器为新密码生成短(pre-4.1)密码哈希。当服务器必须支持旧版本客户端程序时，为了保证兼容性这很有用。

* `(OBSOLETE) --safe-show-database`

  在以前版本的MySQL中，该选项使SHOW DATABASES语句只显示用户具有部分权限的数据库名。在MySQL 5.1中，该选项不再作为现在的 默认行为使用，有一个SHOW DATABASES权限可以用来控制每个账户对数据库名的访问。

* `--safe-user-create`

  如果启用，用户不能用GRANT语句创建新用户，除非用户有mysql.user表的INSERT权限。如果你想让用户具有授权权限来创建新用户，你应给用户授予下面的权限：`mysql> GRANT INSERT(user) ON mysql.user TO 'user_name'@'host_name';`

  这样确保用户不能直接更改权限列，必须使用GRANT语句给其它用户授予该权限。

* `--secure-auth`

  不允许鉴定有旧(pre-4.1)密码的账户。

* `--skip-grant-tables`

  这个选项导致服务器根本不使用权限系统。这给每个人以完全访问所有的数据库的权力！（通过执行`mysqladmin flush-privileges`或`mysqladmin eload`命令，或执行`FLUSH PRIVILEGES`语句，你能告诉一个正在运行的服务器再次开始使用授权表。）

* `--skip-name-resolve`

  主机名不被解析。所有在授权表的Host的列值必须是IP号或localhost。

* `--skip-networking`

  在网络上不允许TCP/IP连接。所有到mysqld的连接必须经由Unix套接字进行。

* `--skip-show-database`

  使用该选项，只允许有SHOW DATABASES权限的用户执行SHOW DATABASES语句，该语句显示所有数据库名。不使用该选项，允许所有用户执行SHOW DATABASES，但只显示用户有SHOW DATABASES权限或部分数据库权限的数据库名。请注意全局权限指数据库的权限。
