sudo mkdir -p /var/www/cdn
sudo curl -sSL -o /var/www/cdn/flag-icons.min.css https://fastly.jsdelivr.net/gh/lipis/flag-icons@7.0.0/css/flag-icons.min.css
sudo curl -sSL -o /var/www/cdn/font-logos.css https://fastly.jsdelivr.net/npm/font-logos@1/assets/font-logos.css

upstream dashboard {
    server 127.0.0.1:8008;
    keepalive 512;
}
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name 32.188.73.145;

    underscores_in_headers on;
    server_tokens off;

    # ---- 压缩：Nginx -> 浏览器 这一段的体积优化 ----
    gzip on;
    gzip_vary on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    # ---- gRPC：Agent 连接（Agent 地址改到 80/443 后才会走这里） ----
    location ^~ /proto.NezhaService/ {
        grpc_set_header Host $host;
        grpc_set_header nz-realip $remote_addr;
        grpc_read_timeout 600s;
        grpc_send_timeout 600s;
        grpc_socket_keepalive on;
        client_max_body_size 10m;
        grpc_buffer_size 4m;
        grpc_pass grpc://dashboard;
    }

    # ---- 静态资源：长效缓存，第二次访问直接命中本地 ----
    location ^~ /assets/ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Accept-Encoding "";
        proxy_pass http://127.0.0.1:8008;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # ---- WebSocket：前台实时数据 ----
    location ~* ^/api/v1/ws/(server|terminal|file)(.*)$ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header nz-realip $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_pass http://127.0.0.1:8008;
    }

    # ---- 本地化 CSS（需先下载到 /var/www/cdn/，零回源） ----
    location = /cdn/flag-icons.min.css {
        alias /var/www/cdn/flag-icons.min.css;
        default_type text/css;
        expires 30d;
        access_log off;
    }
    location = /cdn/font-logos.css {
        alias /var/www/cdn/font-logos.css;
        default_type text/css;
        expires 30d;
        access_log off;
    }

    # ---- Web 主入口 ----
    location / {
        proxy_http_version 1.1;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Host $host;
        proxy_set_header nz-realip $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
        proxy_max_temp_file_size 0;
        proxy_pass http://127.0.0.1:8008;

        sub_filter_once off;
        sub_filter_types text/html;
        sub_filter 'https://fastly.jsdelivr.net/gh/lipis/flag-icons@7.0.0/css/flag-icons.min.css' '/cdn/flag-icons.min.css';
        sub_filter 'https://fastly.jsdelivr.net/npm/font-logos@1/assets/font-logos.css' '/cdn/font-logos.css';
    }
}
