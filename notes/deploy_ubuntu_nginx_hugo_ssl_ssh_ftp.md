# Ubuntu 部署 nginx + hugo + SSL + SSH + ftp +  git

## 1. 阿里云服务器必要配置

### 服务器配置： 

> 在阿里云控制台中配置

阿里云：云服务器 ECS；系统：ubuntu_22_04

#### 服务器端口配置：

> 在阿里云的安全组中配置

增加端口 443 用于 SSL；增加端口 80 用于Nginx；端口 22 用于通信（阿里云默认已开启）。<20,21 端口根据需要开放，本次配置的通信都在 22 端口，因此未开放 20 和 21>

### 域名解析：

域名：coding-x.tech （已通过 ICP 备案）；

域名解析：www(A),@(A)；以及用于 （https） SSL 免费证书 ACME 域名配置，记录类型为 CNAME，具体配置过程和方法及信息查看： [FreeSSL.cn](https://freessl.cn/)。


## 2. 建立本地和服务器之间的通信（SSH + FTP）

> 本地系统为 Ubuntu 或者 Windows

### SSH 安装

SSH 的目的是通过本地通信，操作服务器端系统。

在阿里云控制台中通过远程连接可以实现本地到服务器端的通信。同时，可以配置和修改登录密码。登录服务器后，安装 openssh:

```console

sudo apt-get update (获取需要更新应用的信息)；或 sudo apt update

sudo apt-get upgrade （更新应用，optional）

sudo apt install openssh-server

sudo systemctl status ssh (查看 ssh 状态)

```

Windows 可以用 SSH 工具[PuTTY](https://www.putty.org/)实现本地和服务器端的通信。

Ubuntu 通过：ssh root@公网IP 登录服务器。

### FTP 安装

FTP 安装的目的是实现本地编辑的网站或数据上传至服务器端。

安装 vsftpd:

```console

sudo apt update

sudo apt install vsftpd 

sudo systemctl enable vsftpd

sudo systemctl start vsftpd

sudo systemctl status vsftpd

sudo systemctl restart vsftpd （重启）

```

创建用户（user），本次配置均在 root 主用户下操作。创建的新用户具有有限权限和仅可访问一个指定的文件夹路径控制。

```console

sudo useradd -m ftp_client

sudo passwd ftp_client

```

本地 ftp 上传工具使用 [FileZilla](https://filezilla-project.org/)。

更多的 vsftp 配置可参考 [Install VSFTPD on Ubuntu 20.04](https://www.linode.com/docs/guides/vsftpd-on-ubuntu-2004-installation-and-configuration/)

## 3. Nginx + hugo

### 安装 nginx 和防火墙（ufw）

```console
sudo apt install nginx ufw

sudo ufw allow 22,443,80 (防火墙通过端口)

sudo ufw reload

sudo ufw allow 'Nginx Full'

sudo ufw reload

sudo ufw status

sudo systemctl start nginx

```

编辑 nginx 配置文件，nginx 默认安装位置为 `/etc/nginx`，因此最好定位到该位置`cd /etc/nginx`后再操作 nginx。

nginx 配置方式有两种，一种是直接配置 `nginx.conf`文件；二是在`sites-available`文件夹中新建一个文件，例如`coding-x.tech` （用了域名作为文件名），作为源文件，并链接复制该文件到 `sites-enable`文件夹下（`sudo ln -s /etc/nginx/sites-available/coding-x.tech /etc/nginx/sites-enabled/`），因此后续只需要修改`sites-available`文件夹中新建的文件，`sites-enable`文件夹下链接的文件会自动同步更新。本次配置使用了第二种方法。


```console
sudo vim /etc/nginx/sites-available/coding-x.tech

```

本地 hugo 网站通过 `hugo -D` 生成一个 `public`文件夹，其中即为 hugo 静态网站的所有内容。使用 ftp 工具，例如 FileZilla 将 `pulic` 文件夹上传至服务器端，这里将其置于 `/home`路径下，并修改了文件夹名为`coding-x`，方便标识。 （如果将其置于 vsftpd 自定义用户指定的文件夹下，需要修改文件夹的权限，否则可能无法显示网站，如`sudo chmod 0755 /your/path/to/www`）

在进行 nginx 文件配置之前，可以先在浏览器中打开网页，会出现 nginx 服务器信息，说明上述配置正确。然后配置如下内容，指向自己的 hugo 网页。

`coding-x.tech`示例文件中配置内容如下：

> 注意，此时还未配置 SSL 证书。


```text
server {
    listen 80; # Configure the server for HTTP on port 80
    server_name coding-x.tech; # Your domain name
    root /home/coding-x; # The files for your server
    index index.html; # You will create this file shortly
}
```

配置 nginx 文件后，需要执行

```console
nginx -t （检查配置文件配置是否正确）

sudo nginx -s reload
```

此时应可以打开了网页。

更多关于 nginx + hugo， 可以参考[Deploying a simple static website using Nginx and Hugo](https://pvera.net/posts/create-site-nginx-hugo/)


## 4. SSL 证书获取及对应 nginx 文件配置

如果不申请和配置SSL（https）证书，打开的网站在地址栏会提示 No Secure（不安全）提示，用户传输的信息未加密，造成安全隐患。

一般选择申请免费的 SSL 证书。本次配置使用的为 [FreeSSL.cn](https://freessl.cn/acme-deploy) 。SSL 申请和配置可以按该网站流程提示操作。注意，申请时，提示需要在阿里云控制台按 SSL 给出的信息在安全组中配置端口，其记录类型为 CNAME（将域名指向另一个域名），即将未加密的 http 指向经过 SSL 证书加密的 https。通过 DCV 预检验后会提示，并给出在服务器端获取 SSL证书的命令行，通常包括两种方式，一种为acme.sh；另一种为certbot。任选一种。本次配置过程使用 cerbot。

获取和部署 SSL 前，需要安装 `acme.sh` 或`cerbot`。

```console
curl https://get.acme.sh | sh -s email=my@example.com

curl https://gitcode.net/cert/cn-acme.sh/-/raw/master/install.sh?inline=false | sh -s email=my@example.com (为国内备选地址)

sudo snap install --classic certbot (为 cerbot)
```

然后通过下述命令获取 SSL 证书（由 FreeSSl.cn 给出）。

```console
acme.sh --issue -d coding-x.tech  --dns dns_dp --server https://acme.freessl.cn/v2/DV90/directory/... (为 acme.sh 部署)

certbot certonly --manual -d coding-x.tech  --server https://acme.freessl.cn/v2/DV90/directory/... (为 certbot 部署)
```

执行上述命令后，会提示给出SSL证书所在目录，如：

对于 acme.sh:

```text
[Wed Apr  3 07:52:23 PM CST 2024] Your cert is in: /root/.acme.sh/coding-x.tech_ecc/coding-x.tech.cer
[Wed Apr  3 07:52:23 PM CST 2024] Your cert key is in: /root/.acme.sh/coding-x.tech_ecc/coding-x.tech.key
[Wed Apr  3 07:52:23 PM CST 2024] The intermediate CA cert is in: /root/.acme.sh/coding-x.tech_ecc/ca.cer
[Wed Apr  3 07:52:23 PM CST 2024] And the full chain certs is there: /root/.acme.sh/coding-x.tech_ecc/fullchain.cer
```

对于 cerbot：

```text
Certificate is saved at: /etc/letsencrypt/live/coding-x.tech/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/coding-x.tech/privkey.pem
```

获取的 SSL 证书，一般要复制到 `/etc/nginx/ssl`目录下，其中如果没有`ssl`，用`mkdir /etc/nginx/ssl`建立。复制的命令为`cp /etc/letsencrypt/live/coding-x.tech/fullchain.pem /etc/nginx/ssl`，`cp /etc/letsencrypt/live/coding-x.tech/privkey.pem /etc/nginx/ssl`。（以 cerbot 为例）

acme.sh （nginx）部署如下：

```console
acme.sh --install-cert -d coding-x.tech \
--key-file       /etc/nginx/ssl/privkey.pem   \
--fullchain-file /etc/nginx/ssl/fullchain.pem \
--reloadcmd     "service nginx force-reload"
```

acme.sh 部署后，当证书过期后，会自动更新。

对于 cerbot SSL 证书的自动更新，可以编辑 `crontab`文件`sudo crontab -e`，增加行`0 9 * * * certbot renew --post-hook "systemctl reload nginx"`。

获取和部署 SSL 后，需要修改 nginx 的配置文件，如本次示例的`coding-x.tech`，修改结果如下：

```text
server{
    server_name coding-x.tech;
    root /home/coding-x;
    index index.html;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/coding-x.tech/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/coding-x.tech/privkey.pem; # managed by Certbot
}

server{
    if ($host = coding-x.tech) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        listen 80;
        server_name coding-x.tech;
    return 404; # managed by Certbot
}
```

> 注意包括两个 server 语句块，第二个为指向经过 SSL 证书加密后的 https 地址。

上述所有配置完成后，需要重新加载 nginx，命令如下:

```console
nginx -t （检查配置文件配置是否正确）

sudo nginx -s reload
```

此时打开网站，地址栏不再提示 No Secure（不安全）。





































