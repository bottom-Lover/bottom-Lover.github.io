### Django+uwsgi+nginx搭建个人网站
用户环境：Ubuntu19桌面版；服务器环境：Ubuntu19服务器；程序IDE：pycharm-2019。
#### nginx服务器
##### 1.1 下载
Ubuntu中直接一条命令即可下载安装：
```
sudo apt install nginx
```
##### 1.2 启动与停止
```
nginx启动命令：systemctl start nginx   
nginx重启命令：systemctl restart nginx    
nginx停止命令：systemctl stop nginx
```
##### 1.3 测试nginx
当nginx启动之后，用客户机在浏览器中访问192.168.221.130（这是我的服务器ip地址，你访问的时候应该换成你的服务器ip），即可看到默认的nginx静态网页。    
如果浏览器中能看到nginx默认网页，则说明nginx安装成功。
#### uwsgi服务器
##### 2.1 下载
在Ubuntu终端中下载uwsgi，使用命令：
```
pip3 install uwsgi   
```
在这之前你首先要装上Python3、Python3-pip。
##### 2.2 测试
在你的Documents文件夹中创建一个文件，命名为：wsgiTest.py，使用vim wsgiTest.py编辑文件：   
```python
# !/usr/bin/python3
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"]
```
接下来我们启动 uWSGI 来运行一个 HTTP 服务器，将程序部署在HTTP端口 8080 上：
```
uwsgi --http :8080 --wsgi-file wsgiTest.py   
```
使用客户机在浏览器中访问192.168.221.130:8080即可看到一个Hello World的静态网页。此时说明uwsgi安装成功。
#### Django项目
##### 3.1 下载
使用pip命令下载Django：
```
pip3 install django
```
如果下载速度慢的话，建议使用国内镜像，其命令为：
```
pip3 install django -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```
-i参数后面跟的是国内镜像的地址，--trusted-host表示信任此地址
##### 3.2 创建项目
使用django-admin命令创建你的项目，项目名为django_web：
```
django-admin startproject django_web
```
##### 3.3 启动项目
使用django-admin运行该项目：
```
python3 manage.py runserver 8080
```
能打开网页说明此项目运行正常。  
使用uwsgi启动项目：
```
uwsgi --http :8080 --module django_web.uwgi
```
此命令与上一个命令效果一样。  
至此如果能正常访问项目则说明一切准备工作都已完成。接下来就是配置 nginx + uwsgi 托管项目。
##### 3.4 配置
在项目的根目录（与manage.py同级目录）下创建 wsgi.ini 文件，并编辑如下：
```
# uwsgi 配置文件
[uwsgi]
#端口
socket = 127.0.0.1:8000
# django项目绝对路径
chdir = /home/lin01/mrlinfile/projects/django_web
# 模块路径（项目名称.wsgi）可以理解为wsgi.py的位置
module = django_web.wsgi
# 允许主进程
master = true
#最多进程数
processes  = 4
# 退出时候回收pid文件
vacuum = true
```
在目录 /etc/nginx/conf.d/ 下创建文件 nginx.conf，编辑如下：
```
server {
    # the port your site will be served on
    listen      8080;
    # the domain name it will serve for
    server_name localhost; # substitute your machine's IP address or FQDN
    charset     utf-8;
 
    # max upload size
    client_max_body_size 75M;   # adjust to taste
    location /static {
        alias /home/lin01/mrlinfile/projects/django_web/static; # your Django project's static files - amend as required
    }
    location / {
        uwsgi_pass  127.0.0.1:8000;
        include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed
    }
```
##### 3.5 启动
启动uwsgi托管项目，其命令为：
```
# wsgi.ini配置文件的绝对路径
uwsgi --ini /home/lin01/mrlinfile/projects/django_web/wsgi.ini
```
启动完成后会提示：
```
[uWSGI] getting INI configuration from uwsgi.ini
```
表明uwsgi运行成功。  
此时重启nginx，此项目就算部署完成，重启命令为：
```
systemctl restart nginx
```
至此整个项目部署完成，通过客户机就可以访问服务器了。

***

#### 遇到的问题
在完成部署之后，打开浏览器访问，发现没有样式，网页长这个熊样：
![nocss](https://github.com/13633825898/13633825898.github.io/tree/master/images/nocss.jpg)

至于没有显示出样式，肯定是静态文件出问题了，要么就是根本就没创建静态文件目录，要么就是路径不对。

我呢，是两者都有问题。

现在来解决第一个问题。

第一步，在项目根目录下创建名为static的文件夹；

第二步，在settings里面设置
```
STATIC_ROOT = os.path.join(BASE_DIR, "static")
```
在项目的（不是APP的）urls.py里面设置
```
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... the rest of your URLconf goes here ...
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```
第三步，使用Python命令收集静态文件
```
python manage.py collectstatic
```
至此，第一个问题完美解决

现在解决第二个问题。

首先检查nginx目录下的conf.d/nginx.conf文件的配置，在下处查看静态文件夹位置是否跟你的实际位置一致。
```
# ...
location /static {
        alias /home/lin01/mrlinfile/projects/django_web/static; # your Django project's static files - amend as required
    }
# ...
```
如果不一致记得更改，如果一致，重启nginx服务即可。
不一致的改完此文件之后，还要再去另外一个地方检查，即nginx根目录下的nginx.conf，同样要在下处查看路径。
```
# ...
location /static {
        alias /home/lin01/mrlinfile/projects/django_web/static; # your Django project's static files - amend as required
    }
# ...
```
不一致的记得更改，一致的可以忽略。

（PS：这两个文件什么关系？）
