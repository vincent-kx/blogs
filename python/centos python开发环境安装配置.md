# centos python开发环境安装配置

1. 需先确定服务器上已安装 zlib-devel,openssl-devel 库，没有的话需要使用sudo yum安装

2. 如果需要使用mysql-python之类的package，那么必须先安装一些基础的依赖，如：

   > yum install python-devel.x86_64
   >
   > yum install mysql-devel

   具体应该安装哪一个包可以使用yum search mysql | grep devel命令来先查找

3. 下载python源码包、setuptools最新源码包、pip最新源码包，使用tar -xvf或者unzip 命令分别解压到某个目录，如python-2.7.12，切换到解压目录的python目录下执行以下命令

   >./configure --prefix=/home/user_test/local/python (这种是安装在用户自己的目录下的，这样的user_test用户以后安装python的其他包的时候可以无需root权限)
   >
   >make install

4. 切换到setuptools解压目录，执行以下安装命令

   >/home/user_test/local/python/bin/python setup.py install

5. 切换到pip解压目录，执行以下安装命令

   >/home/user_test/local/python/bin/python setup.py install

6. 如果需要安装MySQLdb包则执行以下命令，其他包也类似

   > pip install mysql-python

以上为指定目录的安装方式，如果采用默认的安装方式的话则需要root权限，以上各吧包的安装可以分别先通过

yum search pkg-name | grep keyword 和 pip search pkg-name | grep keyword来查找需要安装的包具体信息，查找到之后再执行安装，如：

> sudo yum search pip
>
> sudo yum install python2-pip.noarch
>
> pip search mysql-python
>
> sudo pip install MySQL-python