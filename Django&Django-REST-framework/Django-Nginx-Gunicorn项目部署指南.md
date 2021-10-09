## Django-Nginx-Gunicorn项目部署指南

使用前必看：

> 本指南对应部署环境
>
> 服务器环境：Ubuntu 20.04
>
> python版本：3.8
>
> mysql版本：8.0.26
>
> Nginx版本：
>
> Gunicorn版本：

太长不看版本：

1. django配置好静态资源
2. mysql数据库配置好，并执行数据库迁移
3. 在项目文件夹下顺序执行如下命令：

```shell
sudo -H pip install --upgrade pip
sudo -H pip install pipenv

pipenv install django djangorestframework gunicorn pymysql ...

#######

配置gunicorn文件:
----------------- 
# /etc/systemd/system/gunicorn.service
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=你的liunx用户名
Group=你的liunx用户组（默认和用户名相同）
WorkingDirectory=【项目文件夹地址】#（即BASE_DIR的路径）
ExecStart=【虚拟环境文件夹的路径】/bin/gunicorn --access-logfile 【日志文件路径，不写入日志文件的话，使用'-'】 --workers 3 --bind unix:【项目文件夹地址】/【项目名称】.sock 【项目名称】.wsgi:application

[Install]
WantedBy=multi-user.target
--------------------
#######

sudo systemctl start gunicorn
sudo systemctl enable gunicorn

#######

配置nginx文件：
--------------------
# /etc/nginx/sites-enabled/project
server {
    listen 80; # 该端口号为后续外部ip访问的端口号
    server_name 【你的服务器ip或域名】;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ { # static可改为你在django中设置的静态URL前缀
        root /【你的项目地址】;
    }

    location / {
        include proxy_params;
        		   # 即之前在gunicorn中配置的sock
        proxy_pass http://unix:【项目文件夹地址】/【项目名称】.sock; 
    }
}
-----------------------
# /etc/nginx/nginx.conf
... 省略其他配置
http {

    # 从虚拟host文件导入配置(gunicorn所使用的配置)
    include /etc/nginx/sites-enabled/*;

	# ...其他配置

    # 使用server{ …… }写额外的服务器的配置（会覆盖虚拟host配置）
    # 如果目前的conf文件中有server{ …… }字段，将其注释或删除
}

--------------------
####### 

sudo ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled
sudo systemctl restart nginx

# 如果遇到端口问题，使用ufw命令开放相应端口
sudo ufw allow 'Nginx Full'
```

### 1 Django项目配置

将django项目文件夹放置于确定的目录（一般为`/home/...`目录下），称该文件夹目录为`BASE_DIR`，以便接下来各种配置引用。

修改`setting.py`，主要有以下字段：

- 静态资源配置：

  > 静态资源，指需要前端提供的业务代码，如js、css等网页端程序代码，需要后端在运行时加载，需要为其在服务器上单独划分一个存储空间。

  ```python
  # Static files (CSS, JavaScript, Images)
  # https://docs.djangoproject.com/en/3.2/howto/static-files/
  
  import ...
  import os
  
  # 确定访问静态资源使用的URL后缀
  STATIC_URL = '/static/'
  # 确定静态资源在系统的存储目录（Ubuntu目录为例）
  STATIC_ROOT = os.path.join(BASE_DIR, "static/") # static可改为你需要的目录
  ```

- URL索引配置

  刚才配置了静态资源，为了使其能被正确路由，在项目配置文件夹的`urls.py`中修改：

  ```python
  import ...
  from django.conf.urls.static import static
  
  
  urlpatterns = [
      path(...)
      # 结尾增加static索引行
  ]+ static(settings.STATIC_URL,document_root = settings.STATIC_ROOT)
  
  ```

  设置好静态资源后，使用

  ```shell
  python manage.py collectstatic
  ```

  将静态资源拷贝到对应的存储地址。

- 开放host地址

  将`setting.py`中的`ALLOWED_HOSTS`字段改为你的服务器域地址，以允许你的服务器访问django（也可以改为`'*'`以允许所有ip）

  ```python
  # 添加域名或服务器IP地址
  # ALLOWED_HOSTS = [ 'example.com', '203.0.113.5']
  # 如果是子域名，在域名前加'.'
  # ALLOWED_HOSTS = ['.example.com', '203.0.113.5']
  ALLOWED_HOSTS = ['【域名或ip】', '【域名或ip】', . . .]
  ```



### 2 服务器虚拟环境配置

为了防止同一服务器跑多个项目等情况下，python环境冲突，和本地测试一样，需要为django项目新建虚拟环境。在liunx下，常用的有`virtualenv`和`pipenv`，虚拟环境搭建方法如下：

- 使用`virtualenv`:

  安装`virtualenv`

  ```shell
  # 如果是python3，使用pip3
  sudo -H pip install --upgrade pip
  sudo -H pip install virtualenv
  ```

  进入项目文件夹`BASE_DIR`，然后运行命令

  ```shell
  # 项目目录 user@host:~/BASE_DIR$
  virtualenv xxxxenv
  # 虚拟环境名称可以自定义
  ```

  **使用该方法创建的虚拟环境，环境目录在`/BASE_DIR/【虚拟环境名称】`下**，要激活该虚拟环境，使用：

  ```shell
  source 【虚拟环境名称】/bin/activate
  ```

  如果你的命令行标识变为`(【虚拟环境名称】) user@host:~/ $`

  则说明虚拟环境启用成功了。

  安装完成虚拟环境后，直接使用`pip`安装需要的包。

- 使用`pipenv`:

  安装`pipenv`

  ```shell
  # 如果是python3，使用pip3
  sudo -H pip install --upgrade pip
  sudo -H pip install pipenv
  ```

  进入项目文件夹`BASE_DIR`，然后运行命令：

  ```python
  pipenv install
  
  # 若想指定python版本，加入如下参数
  pipenv install --python 3.6 #指定使用Python3.6的虚拟环境
  pipenv install --two        #使用系统的Python2在创建虚拟环境
  pipenv install --three      #使用系统的Python3在创建虚拟环境
  
  #注意：以上三个参数只能在未创建虚拟环境时使用。它们还具有破坏性，会删除当前的虚拟环境，然后用适当版本的虚拟环境替代。
  ```

  使用该方法创建的虚拟环境，环境目录在`~/.local/share/virtualenvs`下，文件夹由项目文件夹和一串字符命名。

  **如果想要获得虚拟环境的位置，在项目文件夹下运行命令**

  ```
  pipenv --venv
  ```

  即可获得。

  新建项目环境后，会在项目文件夹下新建Pipfile文件，其中由虚拟环境的相关配置，可以在此修改pip获取包的远程位置

  ```python
  [[source]]
  # 默认url为“https://pypi.org/simple”
  # 在此修改以更改pip源地址
  url = "https://pypi.tuna.tsinghua.edu.cn/simple/"
  verify_ssl = true
  name = "pypi"
  
  # 储存你已经安装的包名称和目录
  [packages]
  pymysql = "*"
  
  [dev-packages]
  
  # 储存python版本等其他信息
  [requires]
  python_version = "3.8"
  ```

  在项目文件夹下，使用命令行`pipenv shell`即可启动虚拟环境，启动后的命令行表示类似于`virtualenv`，启用后项目文件夹下会多出文件`Pipfile.lock`。

  安装完成虚拟环境后，使用以下命令安装包：

  ```shell
  # 可以空格分隔多个不同的包名称
  pipenv install django gunicorn ... 
  ```

提示：需要在虚拟环境中安装的python基本包：

```
django
djangorestframework
gunicorn
pymysql
...
【其他你在项目中使用到的外部包】
```

### 3 服务器数据库配置

使用mysql配置数据库

安装mysql服务端，使用如下命令

```shell
sudo apt-get install mysql-server
```

安装完成后，使用`sudo mysql`进入数据库，修改root账户默认密码

> 如果忘记了设置密码，或者以后忘记了root密码，可参考以下步骤使用debian进入数据库：
>
> 进入目录`/etc/mysql`，查看文件`sudo cat debian.cnf`
>
> ```mysql
> [client]
> host     = localhost
> # 使用这个用户名密码登录
> user     = debian-sys-maint
> password = Q5j2ifwUnliOh9Wz
> socket   = /var/run/mysqld/mysqld.sock
> ```
>
> 使用其中的账号密码登录即可`mysql -u debian-sys-maint -p`

进入数据库后，查看user表的配置

```mysql
use mysql
select user,host,authentication_string from user;
```

```
+------------------+-----------+------------------------------------------------+
| user             | host      | authentication_string                          |
+------------------+-----------+------------------------------------------------+
| debian-sys-maint | localhost | $A$005$5d....                                  |
| mysql.infoschema | localhost | $A$005$TH....                                  |
| mysql.session    | localhost | $A$005$THISISACOMBINATION....                  |
| mysql.sys        | localhost | $A$005$THISISACOMBINATION....                  |
| root             | localhost | *6BB4837....                                   |
+------------------+-----------+------------------------------------------------+
```

会列出当前的用户、host地址和加密密码字符串，可以通过以下方法修改root用户的密码

mysql8.0以上使用：

```mysql
update user set authentication_string='' where user='root';

# 如果执行该sql语句时出现`ERROR 1396 (HY000)`错误，请先执行上面一行将密码字符串清空
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '【你的密码】'
# 注意用户的host字段和之前查表得到的host字段保持一致（一般情况下是`localhost`，但不一定）
```

mysql5.7可以使用：

```mysql
update user set authentication_string=password("【你的密码】") where user='root';
```

修改完密码后，退出数据库，然后使用新的root账户登录。

```shell
mysql -uroot -p
```

创建新的数据库，采用utf-8字符编码，数据库名称和django配置文件中的数据库名称保持一致。

```mysql
CREATE DATABASE 【数据库名称】 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_520_ci;
```

回到django项目文件夹，在`BASE_DIR`下运行manage.py命令

```shell
python manage.py makemigrations
python manage.py migrate
```

注意，如果覆写了django的User类，在liunx上运行数据库迁移时，可能遇到命名空间问题

`ValueError: Related model 'users.XXX' cannot be resolved`，此时可以执行如下命令解决：

```shell
python manage.py makemigrations --empty 类名
python manage.py makemigrations
python manage.py migrate 类名
python manage.py migrate
```

### 4 Gunicorn服务配置

安装好`gunicorn`后，进入项目文件夹，启动gunicorn测试服务

```shell
gunicorn --bind 0.0.0.0:8000 【项目名称】.wsgi
```

此时访问服务器对应ip和url，如果一切正常，则说明gunicorn可以正常运行。

测试结束后，可以**Ctrl+C**结束gunicorn的运行。

为了使得gunicorn能在服务器上持续运行，我们需要为其配置liunx服务，使得当gunicorn中断后，系统能为我们自动重启gunicorn

进入文件夹`/etc/systemd/system`，新建一个`gunicorn.service`的文件，并打开，写如下内容：

```shell
# /etc/systemd/system/gunicorn.service
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=你的liunx用户名
Group=你的liunx用户组（默认和用户名相同）
WorkingDirectory=【项目文件夹地址】#（即BASE_DIR的路径）
ExecStart=【虚拟环境文件夹的路径】/bin/gunicorn --access-logfile 【日志文件路径，不写入日志文件的话，使用'-'】 --workers 3 --bind unix:【项目文件夹地址】/【项目名称】.sock 【项目名称】.wsgi:application

[Install]
WantedBy=multi-user.target
```

配置完成后，使用如下命令行启动服务：

```shell
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

使用命令`sudo systemctl status gunicorn`查看服务是否成功启动。如果成功，此时检测项目文件夹，应该可以找到与上面配置的名称相同的`【项目名称】.sock`文件。

如果后续修改了项目地址，只需重新修改`gunicorn.service`的内容，然后执行如下命令重启服务即可

```shell
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

### 5 Nginx服务配置

接下来配置Nginx代理服务，使之代理我们的gunicorn服务。

进入文件夹`/etc/nginx/sites-available/`，新建一个配置文件，文件名可以以你的项目命名，以下我们称之为`project`

在配置文件中写如下内容：

```shell
# /etc/nginx/sites-available/project
server {
    listen 80; # 该端口号为后续外部ip访问的端口号
    server_name 【你的服务器ip或域名】;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ { # static可改为你在django中设置的静态URL前缀
        root /【你的项目地址】;
    }

    location / {
        include proxy_params;
        		   # 即之前在gunicorn中配置的sock
        proxy_pass http://unix:【项目文件夹地址】/【项目名称】.sock; 
    }
}
```

配置好后保存文件，运行如下命令行，将配置文件链接到启用处：

```shell
sudo ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled
```

回到上一级文件夹，打开`nginx.conf`文件，确认该配置已经被写入`sites-enabled`字段：

```shell
# /etc/nginx/nginx.conf
user  ubuntu;          # 运行Nginx进程的用户
worker_processes  2;   # 启动两个Nginx worker 进程

events {
    # 每个工作进程 可以同时接收 1024 个连接
    worker_connections  1024; 
}

# 配置 Nginx worker 进程 最大允许 2000个网络连接
worker_rlimit_nofile 2000;

# ... 其他配置

http {

    # 从虚拟host文件导入配置(gunicorn所使用的配置)
    include /etc/nginx/sites-enabled/*;

	#其他配置
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    autoindex on;
    keepalive_timeout 30;
    gzip  on;

    # 使用server{ …… }写额外的服务器的配置（会覆盖虚拟host配置）
    # 如果目前的conf文件中有server{ …… }字段，将其注释或删除
}
```

保存配置后，使用`sudo nginx -t`测试配置文件有效性

显示成功后，即可开始运行nginx：

```shell
sudo systemctl restart nginx

# 如果遇到端口问题，使用ufw命令开放相应端口
sudo ufw allow 'Nginx Full'
```

使用命令`sudo systemctl status gunicorn`查看服务是否成功启动。如果成功，那么恭喜，现在通过nginx端口应该可以成功访问你的服务，部署也就完成了。

### 6 HTTPs加密配置

```
pass
```



### 常见问题及debug手段

大多数问题可以通过查看gunicorn和nginx的日志解决，日志的位置分别位于：

- gunicorn：运行如下命令

```shell
sudo journalctl -u gunicorn
```

也可以手动指定日志文件的存储位置，这需要在`gunicorn.service`配置中在`ExecStart=`参数中加入`--access-logfile`和`--error-logfile`参数并指定相应的日志文件存储位置。

- nginx：

  日志文件分别位于：

  ```
  /var/log/nginx/error.log
  /var/log/nginx/access.log
  ```

  

## 原理解释

```
pass
```



