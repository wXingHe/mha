# MHA--mysql高可用搭建

> 参考链接: 
          
    [https://www.cnblogs.com/xuanzhi201111/p/4231412.html](https://www.cnblogs.com/xuanzhi201111/p/4231412.html)

    [http://joelhy.github.io/2015/02/06/mysql-mha/](http://joelhy.github.io/2015/02/06/mysql-mha/)

## 一.mha搭建准备

* **说明**:目前MHA主要支持一主多从的架构，要搭建MHA,要求一个复制集群中必须最少有三台数据库服务器，一主二从，即一台充当master，一台充当备用master，另外一台充当从库，至少需要三台服务器.
另外,按照mha的结构分还分为管理节点(MHA Manager,该服务器可以和数据服务器公用一台主机,也可以独立,我用的是虚拟机,就独立出来了),和数据节点(MHA Node)

* 在所有节点都要安装MHA node所需的perl模块（DBD:mysql），可以通过yum安装，如果没epel源,先安装epel源

```
yum install perl-DBD-MySQL -y
```

* 在所有的节点安装MHA node(包括管理节点和数据节点,需要梯子)

> **下载:**

```
curl http://www.mysql.gr.jp/frame/modules/bwiki/index.php\?plugin\=attach\&pcmd\=open\&file\=mha4mysql-node-0.56-0.el6.noarch.rpm\&refer\=matsunobu -o mha4mysql-node-0.56-0.el6.noarch.rpm
```

> **安装**

```
yum localinstall -y mha4mysql-node-0.56-0.el6.noarch.rpm
```

* 在管理节点安装MHA node(需要梯子)

> **下载:**

```
curl http://www.mysql.gr.jp/frame/modules/bwiki/index.php\?plugin\=attach\&pcmd\=open\&file\=mha4mysql-node-0.56-0.el6.noarch.rpm\&refer\=matsunobu -o mha4mysql-node-0.56-0.el6.noarch.rpm
```

> **安装**

```	
yum localinstall -y mha4mysql-manager-0.56-0.el6.noarch.rpm

```

## 二.SSH配置:
> MHA Manager内部使用SSH连接到各个MySQL服务器，最新从库节点上的MHA Node也需要使用SSH (scp)把relay log文件发给各个从库节点，故需要各台服务器见需要配置SSH密钥登录方式。

```
ssh-keygen -t rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub "root@192.168.1.110"
ssh-copy-id -i ~/.ssh/id_rsa.pub "root@192.168.1.111"
ssh-copy-id -i ~/.ssh/id_rsa.pub "root@192.168.1.112"
ssh-copy-id -i ~/.ssh/id_rsa.pub "root@192.168.1.113"
```

## 三.管理节点的配置(/etc/mha/mha.conf)

```
[server default]
manager_workdir=/usr/local/mha #mha的工作目录
master_binlog_dir='/usr/local/mysql/var' #master数据库存放binlog目录
remote_workdir=/tmp  #binlog临时保存目录
repl_password=123456  #MySQL复制的用户名
repl_user=wang  #MySQL复制的密码
ssh_user=root #ssh登录名
password=root #MHA监控所有MySQL节点的密码
user=root #MHA监控所有MySQL节点的用户名
master_ip_failover_script=/etc/mha/master_ip_failover #自动Failover脚本
master_ip_online_change_script=/etc/mha/master_ip_online_change #设置手动切换脚本
report_script=/usr/local/bin/send_report #当mha检测到master故障时切换master发送邮件的脚本
[server1]
hostname=192.168.1.110 #数据库地址
[server2]
candidate_master=1  #候选主库
check_repl_delay=0  #忽略延迟大小
hostname=192.168.1.111
[server3]
hostname=192.168.1.112

```
> 配置中提到的脚本:

* master_ip_failover

```
#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '192.168.33.6/24';
my $key = 'wvip';
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ( $command eq "stop" || $command eq "stopssh" ) {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```
* master_ip_online_change

```shell
#!/usr/bin/env perl  
use strict;  
use warnings FATAL =>'all';  
  
use Getopt::Long;  
  
my $vip = '192.168.0.20/24';  # Virtual IP  
my $key = "wvip";  
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";  
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";  
my $exit_code = 0;  
my $orig_master_ssh_user='root';  
my $new_master_ssh_user='root';

my (  
  $command,              $orig_master_is_new_slave, $orig_master_host,  
  $orig_master_ip,       $orig_master_port,         $orig_master_user,  
  $orig_master_password,      $new_master_host,  
  $new_master_ip,        $new_master_port,          $new_master_user,  
  $new_master_password,   
);  
GetOptions(  
  'command=s'                => \$command,  
  'orig_master_is_new_slave' => \$orig_master_is_new_slave,  
  'orig_master_host=s'       => \$orig_master_host,  
  'orig_master_ip=s'         => \$orig_master_ip,  
  'orig_master_port=i'       => \$orig_master_port,  
  'orig_master_user=s'       => \$orig_master_user,  
  'orig_master_password=s'   => \$orig_master_password,  
  'orig_master_ssh_user=s'   => \$orig_master_ssh_user,  
  'new_master_host=s'        => \$new_master_host,  
  'new_master_ip=s'          => \$new_master_ip,  
  'new_master_port=i'        => \$new_master_port,  
  'new_master_user=s'        => \$new_master_user,  
  'new_master_password=s'    => \$new_master_password,  
  'new_master_ssh_user=s'    => \$new_master_ssh_user,  
);  
  
  
exit &main();  
  
sub main {  
  
#print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";  
  
if ( $command eq "stop" || $command eq "stopssh" ) {  
  
        # $orig_master_host, $orig_master_ip, $orig_master_port are passed.  
        # If you manage master ip address at global catalog database,  
        # invalidate orig_master_ip here.  
        my $exit_code = 1;  
        eval {  
            print "\n\n\n***************************************************************\n";  
            print "Disabling the VIP - $vip on old master: $orig_master_host\n";  
            print "***************************************************************\n\n\n\n";  
&stop_vip();  
            $exit_code = 0;  
        };  
        if ($@) {  
            warn "Got Error: $@\n";  
            exit $exit_code;  
        }  
        exit $exit_code;  
}  
elsif ( $command eq "start" ) {  
  
        # all arguments are passed.  
        # If you manage master ip address at global catalog database,  
        # activate new_master_ip here.  
        # You can also grant write access (create user, set read_only=0, etc) here.  
my $exit_code = 10;  
        eval {  
            print "\n\n\n***************************************************************\n";  
            print "Enabling the VIP - $vip on new master: $new_master_host \n";  
            print "***************************************************************\n\n\n\n";  
&start_vip();  
            $exit_code = 0;  
        };  
        if ($@) {  
            warn $@;  
            exit $exit_code;  
        }  
        exit $exit_code;  
}  
elsif ( $command eq "status" ) {  
        print "Checking the Status of the script.. OK \n";  
        `ssh $orig_master_ssh_user\@$orig_master_host \" $ssh_start_vip \"`;  
        exit 0;  
}  
else {  
&usage();  
        exit 1;  
}  
}  
  
# A simple system call that enable the VIP on the new master  
sub start_vip() {  
`ssh $new_master_ssh_user\@$new_master_host \" $ssh_start_vip \"`;  
}  
# A simple system call that disable the VIP on the old_master  
sub stop_vip() {  
`ssh $orig_master_ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;  
}  
  
sub usage {  
print  
"Usage: master_ip_failover –command=start|stop|stopssh|status –orig_master_host=host –orig_master_ip=ip –orig_master_port=po  
rt –new_master_host=host –new_master_ip=ip –new_master_port=port\n";  
}
```

* send_report

```shell
#!/usr/bin/perl

#  Copyright (C) 2011 DeNA Co.,Ltd.
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;
use warnings FATAL => 'all';
use Mail::Sender;
use Getopt::Long;

#new_master_host and new_slave_hosts are set only when recovering master succeeded
my ( $dead_master_host, $new_master_host, $new_slave_hosts, $subject, $body );

my $smtp='smtp服务器';
my $mail_from='显示发信人';
my $mail_user='发信用户';
my $mail_pass='发信用户密码';
#my $mail_to=['to1@qq.com','to2@qq.com'];
my $mail_to='收件人';

GetOptions(
  'orig_master_host=s' => \$dead_master_host,
  'new_master_host=s'  => \$new_master_host,
  'new_slave_hosts=s'  => \$new_slave_hosts,
  'subject=s'          => \$subject,
  'body=s'             => \$body,
);

# Do whatever you want here
mailToContacts($smtp,$mail_from,$mail_user,$mail_pass,$mail_to,$subject,$body);

sub mailToContacts {
        my ($smtp, $mail_from, $mail_user, $mail_pass, $mail_to, $subject, $msg ) = @_;
        open my $DEBUG, ">/var/log/mha/manager.log"
                or die "Can't open the debug    file:$!\n";
        my $sender = new Mail::Sender {
                ctype           => 'text/plain;charset=utf-8',
                encoding        => 'utf-8',
                smtp            => $smtp,
                from            => $mail_from,
                auth            => 'LOGIN',
                TLS_allowed     => '0',
                authid          => $mail_user,
                authpwd         => $mail_pass,
                to              => $mail_to,
                subject         => $subject,
                debug           => $DEBUG
        };
        $sender->MailMsg(
                {
                        msg => $msg,
                        debug => $DEBUG
                }
        ) or print $Mail::Sender::Error;
        return 1;
}

exit 0;

```


## 四.开始运行

* 测试节点间的SSH登录(反馈信息显示All SSH connection tests passed successfully.才是SSH登录配置正确，否则需要根据错误信息需要配置。)

```
masterha_check_ssh  --conf=/etc/mha/mha.conf
```

* 检查复制配置(注意最后一句为MySQL Replication Health is OK.，如果是MySQL Replication Health is NOT OK!，则需要根据反馈的错误信息修改配置)

```
masterha_check_repl --conf=/etc/mha/mha.conf
```

* 启动MHA Manager(启动后可去/usr/log/mha/manager.log查看启动日志来查看相应报错来解决问题)

```
nohup masterha_manager --conf=/etc/mha/app1.cnf > /usr/log/mha/manager.log 2>&1 &
```

* 停止MHA Manager
```
 masterha_stop --conf=/etc/mha/mha.conf
```

## 五.常用命令( 参考:[https://www.kancloud.cn/devops-centos/centos-linux-devops/385182](https://www.kancloud.cn/devops-centos/centos-linux-devops/385182))

* 1.校验ssh等效验证

```
masterha_check_ssh --conf=/etc/mha/app1.cnf
```

* 2.校验mysql复制
```angular2html
masterha_check_repl --conf=/etc/mha/app1.cnf
```

* 3.启动mha监控，在master故障时开启自动转移

```
nohup masterha_manager --conf=/etc/mha/app1.cnf > /tmp/mha_manager.log 2>&1 &
```

* 4.检查启动的状态

```angular2html
masterha_check_status --conf=/etc/mha/app1.cnf
```

* 5.停止mha

```angular2html
masterha_stop  --conf=/etc/mha/app1.cnf
```

* 6.多次failover
 
 MHA在每次failover切换后会在管理目录生成文件app1.failover.complete ，下次在切换的时候
 如果由于间隔时间太短导致切换不成功，应手动清理掉。

```angular2html
rm -rf /var/log/mha/app1/app1.failover.complete
```

或者通过加上参数--ignore_last_failover来忽略

* 7.手工failover

手工failover场景，适用于在master死掉，而masterha_manager未开启情形，如下，指定--master_state=dead

```angular2html
masterha_master_switch --conf=/etc/mha/app1.cnf --master_state=dead --dead_master_host=192.168.0.230 --dead_master_port=3306 --new_master_host=192.168.0.235 --new_master_port=3306
```

可选 --ignore_last_failover

* 8.手动在线主从切换,如下，指定--master_state=alive

```angular2html
masterha_master_switch --conf=/etc/mha/app1.cnf --master_state=alive --new_master_host=192.168.0.230 --new_master_port=3306 --orig_master_is_new_slave 
```
可选 --interactive=0

可选 --running_updates_limit=10000

--orig_master_is_new_slave
表明在切换时原master变为新master的slave节点

--interactive=0
零交互

--running_updates_limit=10000
切换时候选master如果有延迟的话，mha切换不能成功，加上此参数表示延迟在此时间范围内都可切换（单位为s），
但是切换的时间长短是由recover时relay日志的大小决定




