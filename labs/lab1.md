# lab1:

安装Go 1.13，并解压到/usr/local路径

```
sudo tar -zxvf go1.13.6.linux-amd64.tar.gz -C /usr/local
```

添加环境变量，参考[这里](https://www.jianshu.com/p/c43ebab25484)

安装GIt，报错：

>Could not get lock /var/lib/dpkg/lock-frontend - open (11: Resource temporarly unavailable)

解决办法：

```
sudo rm /var/lib/dpkg/lock
sudo rm /var/lib/dpkg/lock-frontend
```

