<!-- TITLE: Https启用记录 -->
<!-- SUBTITLE: A quick summary of Https启用记录 -->

# 安装 nginx
`sudo apt install nginx`
也可以使用其他的web服务器替代 nginx

# 安装 acme.sh
```
curl  https://get.acme.sh | sh
```

#使用 acme.sh 生成证书
```
curl  https://get.acme.sh | sh
```
默认的安装目录是`~/.acme.sh`，其下面有一个`acme.sh`脚本。

### 使用 http 验证的方式生成证书
http 验证的原理是在特定目录生成特定文件，如外部互联网能访问这个文件，则验证通过。
```
acme.sh  --issue  -d mydomain.com -d www.mydomain.com  --webroot  /home/wwwroot/mydomain.com/
```

这需要预先配置好 `nginx` ，使服务器外部可访问 `mydomain.com` 和/或 `www.mydomain.com`

### 使用 dns 验证的方式生成 ssl 证书
dns 验证的原理是在在域名的解析商设置一条 txt 记录，可手工添加。先运行下面的命令来产生 txt 记录信息，然后再手工增加。

```
acme.sh  --issue  --dns   -d mydomain.com
```

目前， cloudflare, dnspod, cloudxns, godaddy 以及 ovh 等数十种解析商都提供了相应的 api ，因此，acme.sh 可以自动增加这条 txt 信息.

先参考 [How to use DNS API](https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md) 来设置。

需要注意的是，如果 export 了 key 和 secret 之后，依然提示未设定 key 和 secret ，可以直接手工将两者的值写入 `~/.acme.sh/account.conf`中，例如（我是 cloudxns）：

>CX_Key='API KEY'
>CX_Secret='SECRET KEY'

然后运行下面的命令得到证书
```
acme.sh   --issue   --dns dns_cx   -d mydomain.com  -d www.mydomain.com
```

#安装 ssl 证书
```
cd ~/.acme.sh
sudo ./acme.sh  --install-cert  -d  mydomain.com   \
        --key-file   /etc/nginx/ssl/mydomain.key \
        --fullchain-file /etc/nginx/ssl/mydomain.cer \
        --reloadcmd  "service nginx force-reload"
```

#配置 nginx
### 生成更安全的 Diffie-Hellman Group
```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

### 定义 ssl 参数
```
sudo nano -w /etc/nginx/snippets/ssl-params.conf
```

内容参考自 Mozilla SSL Generator：
```
# from https://mozilla.github.io/server-side-tls/ssl-config-generator/
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# modern configuration. tweak to your needs.
ssl_protocols TLSv1.2;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
ssl_prefer_server_ciphers on;

# HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
add_header Strict-Transport-Security max-age=15768000;

# OCSP Stapling ---
# fetch OCSP records from URL in ssl_certificate and cache them
ssl_stapling on;
ssl_stapling_verify on;

## verify chain of trust of OCSP response using Root CA and Intermediate certs
# ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

resolver 8.8.8.8 8.8.4.4;

ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

### 修改 nginx 的网站配置
```
server {
        listen 80;
        listen [::]:80;

        server_name mydomain.com;

        # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
        ssl_certificate /etc/nginx/ssl/mydomain.com.cer;
        ssl_certificate_key /etc/nginx/ssl/mydomain.com.key;

        include snippets/ssl-params.conf;

        server_name mydomain.com;

        root /home/wwwroot/mydomain.com/;
		
		......
```

#通过 crontab 自动续签 ssl 证书

`sudo crontab -e` 之后，配置内容如下：

```
# m h  dom mon dow   command
40 0 * * * /full path to/acme.sh --cron --home "/full path to/.acme.sh" > /dev/null
```
