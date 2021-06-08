# 1828126011安装

使用清华大学开源软件镜像`https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-13.6.7-ce.0.el7.x86_64.rpm`

安装过程如下：

```
# 安装依赖
yum -y install policycoreutils openssh-server openssh-clients postfix
# 设置postfix开机自启，并启动，postfix支持gitlab发信功能
systemctl enable postfix && systemctl start postfix
# 线上安装
rpm -ivh https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-13.6.7-ce.0.el7.x86_64.rpm
```

安装完成后显示，需要按照提示，修改配置文件url地址为本地服务器地址并重新加载配置：

```
vim /etc/gitlab/gitlab.rb
# 编辑，注意要与nginx代理相同
external_url 'http://gitlab'
# 编辑更换端口号，在有冲突时:：:：
unicorn['port'] = 9003
grafana['http_port'] = 3001
nginx['enable'] = true
nginx['listen_port'] = 9002
# 编辑邮箱
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qiye.aliyun.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "app_sinolube@ritsc.com"
gitlab_rails['smtp_password'] = "SinoLube@2020"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'app_sinolube@ritsc.com'
# 从新加载配置并重启
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```

其他gitlab操作

```
gitlab-ctl stop // 停止gitlab
gitlab-ctl restart // 重启
gitlab-ctl status // 查看状态
gitlab-ctl tail {down的服务} // 查看错误原因
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION // 查看版本号
rpm -e gitlab-ce // 卸载gitlab
```

测试邮件发送

```
gitlab-rails console
Notify.test_email('yaohui_huang@aliyun.com', 'Hello World', 'This is a test message').deliver_no
w
```



设置关闭注册

# 配置nginx

编辑创建对于gitlab的配置

```
upstream gitlab {
    server 127.0.0.1:9002;
}
server {
    listen       80;
    server_name  gitlab.4008109886.cn;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_connect_timeout   18000;
        proxy_send_timeout      18000;
        proxy_read_timeout      18000;
        proxy_pass http://gitlab;
        proxy_redirect off;
    }
}
```

重新加载nginx配置 

```
nginx -t
nginx -s reload
```



# 重置密码

登录gitlab的服务器，使用命令: `su - git ` 进入控制台

```bash
gitlab-rails console
```

通过查找管理员账号并重新设置密码

```
u = User.where(id: 1).first
u.password='new_password'
u.save!
```

