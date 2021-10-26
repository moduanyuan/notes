# Get free SSL/TLS certificates, To enable HTTPS on your website

申请 Let's Encrypt 发布的HTTPS证书.(以下 `<>` 部分需要替换为实际内容)

1. 安装 `acme.sh`.

    `curl https://get.acme.sh | sh`

2. 开启一个新的web服务(此处使用了 `docker` 工具 + `nginx` 镜像).

    `docker run --name nginx-case -v <your html directory>:/usr/share/nginx/html -p 80:80 -d nginx:1.17`

3. 由于默认的nginx配置信息是将网站根目录指定到了 `<your html directory>`, `acme.sh` 则会在网站根目录下创建一个新目录及文件, 用于 Let's Encrypt 验证. 因此需要改变网站根目录权限, 让其可以写入文件.

    `sudo chmod 777 <your html directory>`

4. 使用 `acme.sh` 进行HTTPS证书的申请, 这一步完成便已经获取到了证书.(可以使用多个 `-d` 参数来添加多个子域名)

    `acme.sh --issue -d <your domain> -d <your subdomain> --webroot <your html directory>`

5. 编写nginx服务器的配置信息, 以开启HTTPS功能(文件位于 `/etc/nginx/conf.d/`)

    示例(`default.conf`):

    ```conf
    server {
        # 该服务器配置用于将HTTP跳转到HTTPS
        listen 80;
        server_name <your domains>;
        server_tokens off;

        # 服务重定向
        location / {
            return 301 https://$server_name$request_uri;
        }
    }

    server {
        # 此处开了HTTPS功能以及HTTP2.0
        listen 443                  ssl http2 default_server;
        server_name                 <your domains>;
        server_tokens               off;

        # 设置你的证书路径
        ssl_certificate             /etc/nginx/cert/fullchain.cer;
        ssl_certificate_key         /etc/nginx/cert/<your domain>.key;
        ssl_session_timeout         60m;

        location / {
            root /usr/share/nginx/html;
            index homepage.html;
        }
    }
    ```

6. 使用 `docker` 启动web服务.

    ```bash
    docker run --name <your container name> --restart=always -p 80:80 -p 443:443 \
        -v <your html directory>:/usr/share/nginx/html \
        -v <your nginx.conf file>:/etc/nginx/nginx.conf \
        -v <your nginx-conf.d directory>:/etc/nginx/conf.d \
        -v <your certificates directory>:/etc/nginx/cert \
        -v <your nginx log directory>:/var/log/nginx \
        -d nginx:1.17
    ```

7. 对证书存放路径进行权限修改, 为 `acme.sh` 提供证书迁移功能.

    `sudo chmod 777 <your certificates directory>`

8. 使用 `acme.sh` 将HTTPS证书复制到 nginx 服务配置的证书路径下. `--reloadcmd` 选项则是设置证书复制后, 所要执行的动作. 因为是使用 `docker` 进行部署的, 所以只要重启 `docker` 容器即可.

    ```bash
    acme.sh --installcert    -d <your domain> \
            --key-file       <your certificates directory>/<your domain>.key \
            --fullchain-file <your certificates directory>/fullchain.cer \
            --reloadcmd      "docker restart <your container>"
    ```
