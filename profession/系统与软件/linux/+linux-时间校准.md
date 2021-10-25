# Linux 时间校准
***
## 命令
### 方式一
```
tzselect
export TZ='Asia/Shanghai'
source ~/.bashrc
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
timedatectl set-timezone 'Asia/Shanghai'

yum -y install ntp
/usr/sbin/ntpdate -u cn.pool.ntp.org
```

### 方式二
```shell script
date

yum install -y ntpdate
rm -rf /etc/localtime
cp /usr/share/zoneinfo/Asia/Shanghai/ /etc/localtime
ntpdate -u ntp.api.bz
date
hwclock -w
```

## 参考链接
- [Linux服务器时间不一致问题的解决](https://blog.csdn.net/zisefeizhu/article/details/81535299)
- [linux 修改服务器时间](https://blog.csdn.net/wangbailin2009/article/details/53332453)