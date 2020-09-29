# Apache2.4 安装流程

## 安装前准备工作

### 检查磁盘空间和环境

```shell
# 查看安装位置是否有足够的空间
df -h
# 查看是否有GCC编译环境
man gcc  (或者 gcc -v)
# 系统信息
lsb_release -a
getconf LONG_BIT
```

### 准备安装介质

httpd-2.4.39  
apr-1.5.2  
apr-util-1.6.0  
pcre-8.38
mod_wl_24.so

### 用户和目录规划

权限分离 中间件全部使用wlsoper安装 775权限
Apache用户属于wlsoper用户组, 对相应目录有读写权限即可

Apache安装目录: /tpsys/apache
介质存放目录: /tpsys/install
/tmp下如果有/tmp/_wl_proxy, 需要确保apache对该目录有读写权限

```shell
groupadd -g 500 wlsoper
useradd -u 500 -g wlsoper -G wheel -d /home/wlsoper wlsoper
useradd -u 8080 -g wlsoper -d /home/apache apache
echo "wlsoper_passwd" | passwd --stdin wlsoper
echo "apache_passwd" | passwd --stdin apache
```

## 安装Apache2.4

### 解压所有介质包

```shell
cd /tpsys/install

tar -zxvf httpd-2.4.39.tar.gz  
tar -zxvf apr-1.5.2.tar.gz  
tar -zxvf apr-util-1.6.0.tar.gz  
tar -zxvf pcre-8.38.tar.gz

mv apr-1.5.2 apr  
mv apr-util-1.6.0 apr-util  
mv pcre-8.38 pcre

cp -r apr apr-util pcre /tpsys/install/httpd-2.4.39/srclib/
```

### 编译

```shell
cd /tpsys/install/httpd-2.4.39/srclib/
# 编译pcre
cd pcre
./configure --prefix=/tpsys/apache/httpd/pcre
make
make install
# 编译apr
cd apr
./configure --prefix=/tpsys/apache/httpd/apr
make
make install
# 编译apr-util
cd apr-util
./configure --prefix=/tpsys/apache/httpd/apr-util --with-apr=/tpsys/apache/httpd/apr --with-expat=builtin
make
make install
```

若出现问题,使用`make clean`清楚编译缓存,重新编译即可,若报builtin相关错误,可以不带builtin参数

```shell
cd /tpsys/install/httpd-2.4.39
./configure --prefix=/tpsys/apache/httpd --enable-so --enable-mods-shared=all --with-apr=/tpsys/apache/httpd/apr
--with-apr-util=/tpsys/apache/httpd/apr-util --with-pcre=/tpsys/apache/httpd/pcre
make
make install
```

## 启停

```shell
cd /tpsys/apache/httpd/bin/
# 启动
./apachectl -k start
# 停止
./apachectl -k stop
# 重启
./apachectl -k restart
```

## 优化配置

修改的相关文件及时备份

### 自启动

```shell
vi /etc/rc.local
su - apache -c "/tpsys/apache/httpd/bin/httpd -k start"
```

### 端口

```shell
Listen 8080
```

### 代理

#### 反向代理

```shell
# 反向代理需要开启以下模块,在httpd.conf文件中解除相关配置
mod_proxy_connect.so
mod_proxy_ftp.so
mod_proxy_http.so
mod_proxy.so

# 反向代理配置,在httpd.conf文件添加
# 访问 http://IP:8080/life 时,反向代理到 http://10.1.1.1:7001/life
<VirtualHost *:8080>
    ProxyRequests Off
    ServerName IP
    <Proxy *>
        # 2.4版本
        Require all denied
        # 2.2版本用下面两句
        #Order deny,allow
        #Deny from all
    </Proxy>
    ProxyPass /life http://10.1.1.1:7001/life
    ProxyPassReverse /life http://10.1.1.1:7001/life
</VirtualHost
```

#### weblogic代理

将weblogic模块mod_wl_24.so上传到/tpsys/apache/httpd/modules/ 并赋权775

```shell
# weblogic代理配置,在httpd.conf文件添加
DocumentRoot "/tpsys/servers/applications/html_tp"
# 需要转发的文根
<Directory "/tpsys/servers/applications/html_tp">
# 本机IP
ServerName IP
<IfModule mod_weblogic.c>
    WeblogicCluster 10.1.1.1:8001,10.1.1.2:8001
    # 单节点的话就直接配置下面两行即可
    #WeblogicHost 10.1.1.1
    #WeblogicPort 8001
    # 当URL匹配到下面配置后面的表达式时,把请求转发给weblogic处理
    MatchExpression *.do
    MatchExpression *.jsp
    MatchExpression /life/general/servlet
</IfModule>
```

### SSL证书配置

#### 配置前检查OpenSSL版本,如果版本比较低就升级版本

```shell
rpm -qa | grep openssl
# 若需要SSL的话,在编译时加入 --enable-ssl 即可
```

#### 启用SSL

```shell
# 配置文件中取消下面两行注释
LoadModule ssl_module modules/mod_ssl.so
Include conf/extra/httpd-ssl.conf
```

#### 证书配置

暂未更新

## 安全加固

在httpd.conf中修改或新增相关配置

### 修改项

```shell
# 修改目录遍历漏洞
Options Indexes FollowSymLinks --> Options FollowSymLinks
# 修复远程拒绝服务漏洞,解除下面注释
LoadModule headers_module module/mod_headers.so
```

### 新增项

```shell
# 只允许GET和POST方法（实际上是禁用了PUT和DELETE等高危方法）
<Location "/">
    <LimitExcept GET POST>
        Deny from all
    </LimitExcept>
</Location>
# 关闭TRACE,防止TRACE方法被访问者恶意利用
TraceEnable Off
# 修复远程拒绝访问服务漏洞
SetEnvIf Range (,.*?){5,} bad-range=1
RequestHeader unset Range env=bad-range
# 限制http请求主体的大小
LimitRequestBody 102400
# 隐藏错误信息
ServerSignature Off
# 隐藏版本信息
ServerTokens Prod
```
