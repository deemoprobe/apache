# 1. Apache2.4 安装流程

## 1.1. 安装前准备工作

### 1.1.1. 检查磁盘空间和环境

```shell
# 查看安装位置是否有足够的空间
df -h
# 查看是否有GCC编译环境
man gcc  (或者 gcc -v)
# 系统信息
lsb_release -a
getconf LONG_BIT
```

### 1.1.2. 准备安装介质

httpd-2.4.39  
apr-1.5.2  
apr-util-1.6.0  
pcre-8.38  
mod_wl_24.so  
openssl-1.1.0c

### 1.1.3. 用户和目录规划

权限分离 中间件全部使用admin安装 775权限  
Apache用户属于admin用户组, 对相应目录有读写权限即可

Apache安装目录: /app/apache  
介质存放目录: /app/install  

```shell
groupadd -g 500 admin
useradd -u 500 -g admin -G wheel -d /home/admin admin
useradd -u 8080 -g admin -d /home/apache apache
echo "admin_passwd" | passwd --stdin admin
echo "apache_passwd" | passwd --stdin apache
```

## 1.2. 安装Apache2.4

### 1.2.1. 解压所有介质包

```shell
cd /app/install

tar -zxvf httpd-2.4.39.tar.gz  
tar -zxvf apr-1.5.2.tar.gz  
tar -zxvf apr-util-1.6.0.tar.gz  
tar -zxvf pcre-8.38.tar.gz

mv apr-1.5.2 apr  
mv apr-util-1.6.0 apr-util  
mv pcre-8.38 pcre

cp -r apr apr-util pcre /app/install/httpd-2.4.39/srclib/
```

### 1.2.2. 编译

```shell
cd /app/install/httpd-2.4.39/srclib/
# 编译pcre
cd pcre
./configure --prefix=/app/apache/httpd/pcre
make
make install
# 编译apr
cd apr
./configure --prefix=/app/apache/httpd/apr
make
make install
# 编译apr-util
cd apr-util
./configure --prefix=/app/apache/httpd/apr-util --with-apr=/app/apache/httpd/apr --with-expat=builtin
make
make install

# 检查OpenSSL版本,如果版本比较低(低于1.1.0)就升级版本,否则编译httpd时加上--enable-ssl会报错
openssl version
cd /app/install
tar -zxvf openssl-1.1.0c.tar.gz
cd openssl-1.1.0c
./config --prefix=/usr/local/openssl
make
make install
mv /usr/bin/openssl /usr/bin/openssl.bak
ln -sf /usr/lcoal/openssl/bin/openssl /usr/bin/openssl
cp /etc/ld.so.conf /etc/ld.so.conf.bak
echo "/usr/local/openssl/lib" >> /etc/ld.so.conf
ldconfig -v
```

若出现问题,使用`make clean`清楚编译缓存,重新编译即可,若报builtin相关错误,可以不带builtin参数

```shell
cd /app/install/httpd-2.4.39
./configure --prefix=/app/apache/httpd --enable-so --enable-ssl --enable-proxy
--enable-proxy-http --enable-mods-shared=all --with-apr=/app/apache/httpd/apr
--with-apr-util=/app/apache/httpd/apr-util --with-pcre=/app/apache/httpd/pcre
--with-ssl=/usr/local/openssl
make
make install
```

## 1.3. 启停

```shell
cd /app/apache/httpd/bin/
# 启动
./apachectl -k start
# 停止
./apachectl -k stop
# 重启
./apachectl -k restart
```

## 1.4. 优化配置

修改的相关文件及时备份

### 1.4.1. 自启动

```shell
vi /etc/rc.local
su - apache -c "/app/apache/httpd/bin/httpd -k start"
```

### 1.4.2. 设置apache用户权限

```shell
visudo
Cmnd_Alias APACHE=/bin/kill,/bin/netstat,/app/apache/httpd/bin/*,/app/applications/apache/script/*sh
```

### 1.4.3. 配置脚本

```shell
cd /app/application/apache/script
vi start.sh
sudo -u admin /app/apache/httpd/bin/httpd -k start
vi stop.sh
sudo -u admin /app/apache/httpd/bin/httpd -k stop
vi restart.sh
sudo -u admin /app/apache/httpd/bin/httpd -k restart
```

### 1.4.4. 端口

```shell
Listen 8080
```

### 1.4.5. 代理

#### 1.4.5.1. 反向代理

```shell
# 反向代理需要开启以下模块,在httpd.conf文件中解除相关配置
mod_proxy_connect.so
mod_proxy_ftp.so
mod_proxy_http.so
mod_proxy.so
mod_rewrite.so

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

#### 1.4.5.2. weblogic代理

将weblogic模块mod_wl_24.so上传到/app/apache/httpd/modules/ 并赋权775

```shell
# weblogic代理配置,在httpd.conf文件添加
DocumentRoot "/app/servers/applications/html_tp"
# 需要转发的文根
<Directory "/app/servers/applications/html_tp">
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

### 1.4.6. SSL证书配置

#### 1.4.6.1. 启用SSL

```shell
# 配置文件中取消下面两行注释
LoadModule ssl_module modules/mod_ssl.so
Include conf/extra/httpd-ssl.conf
```

#### 1.4.6.2. 证书配置

暂未更新

## 1.5. 安全加固

在httpd.conf中修改或新增相关配置

### 1.5.1. 修改项

```shell
# 修改目录遍历漏洞
Options Indexes FollowSymLinks --> Options FollowSymLinks
# 修复远程拒绝服务漏洞,解除下面注释
LoadModule headers_module module/mod_headers.so
```

### 1.5.2. 新增项

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

## 1.6. 问题总结

1. apr-util编译报错

    要把apr放在apr-util之前编译

2. 加载weblogic模块后,语法测试错误

    模块版本错误,不能使用低于mod_wl_24.so的模块

3. 代理配置成功后,无法访问资源

    查看文根是否正确,能否在配置的路径下找到相关资源

4. 编译时出错`aclocal-1.15' is missing on your system`

    ```shell
    #进入configure所在目录,执行下面自动编译修复语句
    autoreconf -f -i
    # 然后再依次执行./configure, make, make install
    ```

5. 编译pcre时报错`rm: cannot remove 'libtoolT': No such file or directory`

    ```shell
    vi configure
    Change the line
    $RM "$cfgfile"
    to
    $RM -f "$cfgfile"
    This will resolve the error
    ```
