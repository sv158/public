先申请证书
```
acme.sh --dns dns_dp --issue -d example.com -d *.example.com
```
安装docker
```
sudo apt install docker.io
```
再安装v2ray
```
sudo docker pull v2ray/official
```

再配置v2ray

```
mkdir /etc/v2ray
vim /etc/v2ray/config.json
```
内容如下
```
{
    "log": {
      "access": "/var/log/v2ray/access.log",
      "error": "/var/log/v2ray/error.log",
      "loglevel": "warning"
    },
    "inbounds": [
      {
        "port": 36000,
        "protocol": "vmess",
        "settings": {
          "clients": [
            {
              "id": "{{uuid}}",
              "alterId": 32
            }
          ]
        },
        "tag": "in-0",
        "streamSettings": {
          "network": "ws",
          "wsSettings": {
            "path": "/{{path}}"
          }
        }
      }
    ],
    "outbounds": [
      {
        "tag": "direct",
        "protocol": "freedom",
        "settings": {}
      },
      {
        "tag": "blocked",
        "protocol": "blackhole",
        "settings": {}
      }
    ],
    "routing": {
      "domainStrategy": "AsIs",
      "rules": [
        {
          "type": "field",
          "ip": [
            "geoip:private"
          ],
          "outboundTag": "blocked"
        }
      ]
    }
}
```

配置完启动容器
```
docker run -d --name v2ray -v /etc/v2ray:/etc/v2ray -v /var/log/v2ray:/var/log/v2ray -p 36000:36000 v2ray/official v2ray -config=/etc/v2ray/config.json
```
如果有必要，可以在`-d`后面加上`--restart=always`参数

然后配置nginx
```
server {
    listen 443 ssl;
    
    ssl on;                                                         
    ssl_certificate       {{fullchain.cer}};  
    ssl_certificate_key   {{example.com.key}};
    ssl_protocols         TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_prefer_server_ciphers on;              
    #ssl_ciphers           HIGH:!aNULL:!MD5;

    root /var/www/v2ray;
    server_name us1.cs3.wiki;

    access_log  /var/log/nginx/v2ray_access.log;
    error_log /var/log/nginx/v2ray_error.log;

    location / {
        try_files $uri $uri/ =404;
        index index.html;
    }

    location /upload {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:36000/upload;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 300s;
        proxy_intercept_errors on;

         sendfile                on;
	 tcp_nopush              on;
	 tcp_nodelay             on;
	 keepalive_requests      25600;
	 keepalive_timeout	300 300;
	 proxy_buffering         off;
	 proxy_buffer_size       8k;
    }

    error_page 400 404 /404.html;
}
```

最后配置客户端

```
{
  "inbounds": [
    {
      "tag": "proxy",
      "port": {{port}},
      "listen": "127.0.0.1",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "settings": {
        "auth": "noauth",
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "us1.cs3.wiki",
            "port": 443,
            "users": [
              {
                "id": "{{uuid}}",
                "alterId": 32
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "allowInsecure": true
        },
        "wsSettings": {
          "connectionReuse": true,
          "path": "/{{path}}"
        }
      },
      "mux": {
        "enabled": true
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings":{}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      }
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
          "api"
        ],
        "outboundTag": "api"
      }
    ]
  }
}
```

如果证书renew失败，则尝试手动
```
acme.sh --renew -d "example.com" -d "*.example.com" --dns
```
生成证书的位置可能会变，因此链接可能会失效，所以最保险的办法还是把检查过期和安装证书的步骤合并然后定期执行（比较麻烦，故放弃）。
