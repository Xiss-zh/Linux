# CentOS7基本优化

### 1、系统安装配置网卡名称为ethx

```shell
# 在安装界面tab键增加如下命令
net.ifnames=0 biosdevname=0
```

![image-20230504172931650](D:\Github\Linux\png\image-20230504172931650.png)



### 2、安装过程中的一些基本截图

![image-20230504173429632](D:\Github\Linux\png\image-20230504173429632.png)

![image-20230504173535034](D:\Github\Linux\png\image-20230504173535034.png)

![image-20230504173639242](D:\Github\Linux\png\image-20230504173639242.png)

![image-20230504173619330](D:\Github\Linux\png\image-20230504173619330.png)

### 3、配置阿里yum源和epel源

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

### 4、关闭防火墙和selinux

```shell
# 关闭防火墙
systemctl disable --now firewalld

# 关闭selinux
setenforce 0 && sed -ri 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 查看selinux是否关闭
getenforce
```

### 5、安装常用工具与更新系统

```
yum install -y vim wget unzip net-tools telnet
```

```shell
# 设置update不更新内核
cat >> /etc/yum.conf << EOF 

exclude=kernel*

EOF

# 更新centos系统
yum -y update
```



### 6、禁用NetworkManager

```shell
# centos7默认使用network管理网络，禁用NetworkManager减少干扰，比如会干扰k8s的pod之间网络
systemctl disable --now NetworkManager
```

### 7、调整网卡配置

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

### 8、配置时间同步

```shell
# 使用chrony时间同步
yum -y install chrony

# 增加时间国内时间同步源
vim /etc/chrony.conf

# 禁用默认时间源，增加国内时间源
server ntp.aliyun.com iburst
server cn.ntp.org.cn iburst

#运行chrony服务并开机自启
systemctl enable --now chronyd

# 查看同步进度
chronyc sources –v

# 同步硬件时间
hwclock --show
hwclock --systohc
```

### 9、基本内核优化

```shell
cat >> /etc/sysctl.conf << EOF
# 当物理内存使用到90%时,才使用swap交换分区，默认70%
vm.swappiness = 10

# 强制 Linux 系统最低保留的空闲内存512M
vm.min_free_kbytes = 524288

# 设置系统中所允许文件描述符的数量限制1024*1024
fs.file-max = 1048576

# 设置Linux下进程数量的限制
kernel.pid_max = 32768

EOF
```

### 10、优化limit限制

```shell
# 修改limit限制
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
echo "UsePAM yes" >> /etc/ssh/sshd_config

#centos7.3以下
echo "UseLogin yes" >> /etc/ssh/sshd_config


# 重启生效
reboot

# 查看limit生效
ulimit -a


# 查看最大句柄数量限制
cat /proc/sys/fs/file-max

# 查看系统总的进程数量
cat /proc/sys/kernel/pid_max

# 查看当前系统打开的句柄数量和最大句柄数量限制
cat /proc/sys/fs/file-nr
```

### 11、vim优化

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

### 12、其它优化

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



```shell
# 设置开机脚本执行权限
chmod 744 /etc/rc.d/rc.local
```

### 12、升级内核

```shell
# 导入仓库源
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm

# 查看最新长期支持内核
yum --enablerepo="elrepo-kernel" list --showduplicates | sort -r | grep kernel-lt.x86_64

# 查看最新稳定版本内核
yum --enablerepo="elrepo-kernel" list --showduplicates | sort -r | grep kernel-ml.x86_64

# 安装长期支持版本内核
yum --enablerepo=elrepo-kernel install kernel-lt-devel kernel-lt -y

# 查看系统上的所有可用内核：
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

# 设置新内核启动
grub2-set-default 0

# 重启系统
reboot

# 查询当前内核版本
uname -r
```

![image-20230504180710134](D:\Github\Linux\png\image-20230504180710134.png)
