# CentOS7基本优化

### 1、安装配置网卡名称为ethx

```
# 在
net.ifnames=0 biosdevname=0
```



### 1、配置阿里yum源和epel源

```shell
# 备份源
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/

# 下载阿里源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

# 删除出现 Couldn't resolve host 'mirrors.cloud.aliyuncs.com' 信息
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

# 配置扩展源
curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo

# 重新生成缓存
yum makecache
```

### 2、关闭防火墙和selinux

```shell
# 关闭防火墙
systemctl disable --now firewalld

# 关闭selinux
setenforce 0 && sed -ri 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 查看selinux是否关闭
getenforce
```

### 3、安装常用工具

```
yum install -y vim wget unzip net-tools telnet
```

```shell
# 设置update不更新内核
cat >> /etc/yum.conf << EOF 

exclude=kernel*

EOF

# 更新centos系统
yum update
```



### 4、禁用NetworkManager

```shell
# centos7默认使用network管理网络，禁用NetworkManager减少干扰，比如会干扰k8s的pod之间网络
systemctl disable --now NetworkManager
```

### 5、调整网卡配置

```shell
# 删除多余配置，做系统模板特别是uuid和mac不能保留
vim /etc/sysconfig/network-scripts/ifcfg-eth0


TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=x.x.x.x
PREFIX=24
GATEWAY=x.x.x.x
DNS1=114.114.114.114
DNS2=223.5.5.5

# 重启网卡生效
systemctl restart network
```

### 6、配置时间同步

```shell
# 使用chrony时间同步
yum install chrony

# 增加时间国内时间同步源
vim /etc/chrony.conf

# 禁用默认时间源，增加国内时间源
server ntp.aliyun.com iburst
server cn.ntp.org.cn iburst

#运行chrony服务并开机自启
systemctl enable chronyd --now

# 查看同步进度
chronyc sources –v

# 同步硬件时间
hwclock --show
hwclock --systohc
```



```shell
# 设置history保留1000条
cat >> /etc/bashrc << EOF
# 保留1000条命令记录
export HISTSIZE=1000
export HISTFILESIZE=1000
EOF
```

```shell
# 设置支持中文字符集
sed -i "s/en_US.UTF-8/zh_CN.UTF-8/g" /etc/locale.conf
```



```
chmod 744 /etc/rc.d/rc.local
```



```shell
cat >> /etc/sysctl.conf << EOF
# 当物理内存使用到90%时,才使用swap交换分区，默认70%
vm.swappiness = 10

# 强制 Linux 系统最低保留的空闲内存512M
vm.min_free_kbytes = 524288

# 设置系统中所允许文件描述符的数量限制1024*1024
fs.file-max = 1048576

EOF
```

```shell
# 查看最大句柄数量限制
cat /proc/sys/fs/file-max

# 查看系统总的进程数，默认32768已经够用
cat /proc/sys/kernel/pid_max

# 查看当前系统打开的句柄数量和最大句柄数量限制
cat /proc/sys/fs/file-nr
```

```shell
# 修改用户limit限制
cat >> /etc/security/limits.conf << EOF
# 不限制最大锁定内存地址空间
*    soft    memlock    unlimited
*    hard    memlock    unlimited

# 不限制 CPU 时间
*    soft    cpu    unlimited
*    hard    cpu    unlimited

# 设置进程的最大数目
*    soft    nproc    65536
*    hard    nproc    65536

# 设置打开文件的最大数目
*    soft    nofile    65536
*    hard    nofile    65536


# 默认栈空间大小100M
*    soft    stack   102400
*    hard    stack   102400

# 不限制内核文件的大小
*    soft    core    unlimited
*    hard    core    unlimited
EOF


```

```shell
# 设置systemd service的资源限制
# 配置系统实例
sed -i "s?#DefaultLimitCORE=?DefaultLimitCORE=infinity?g" /etc/systemd/system.conf
sed -i "s?#DefaultLimitNOFILE=?DefaultLimitNOFILE=65536?g" /etc/systemd/system.conf
sed -i "s?#DefaultLimitNPROC=?DefaultLimitNPROC=65536?g" /etc/systemd/system.conf

# 配置用户实例
sed -i "s?#DefaultLimitCORE=?DefaultLimitCORE=infinity?g" /etc/systemd/user.conf
sed -i "s?#DefaultLimitNOFILE=?DefaultLimitNOFILE=65536?g" /etc/systemd/user.conf
sed -i "s?#DefaultLimitNPROC=?DefaultLimitNPROC=65536?g" /etc/systemd/user.conf


## 对于用户limit限制，需调整sshd配置,当我们用ssh客户端工具连接后,才会生效
# centos7.4以上
echo "UsePAM yes" >>/etc/ssh/sshd_config

#centos7.3以下
echo "UseLogin yes" >>/etc/ssh/sshd_config

systemctl restart sshd

# 重启生效
reboot
# 查看limit生效
ulimit -a



```



```shell
cat >> /etc/vimrc << EOF
" 解决粘贴问题
set paste
" 设置tab键为4个空格
set ts=4
set expandtab
%retab!
" 设置光标所在行的标识线
set cul
" 设置光标匹配的括号
set showmatch

EOF
```

