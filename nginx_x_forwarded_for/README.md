# nginx 

## nginx proxyで接続元IPアドレス(X-Forwarded-For)を設定する

nginxでロードバランサで接続元IP([X-Forwarded-For](https://ja.wikipedia.org/wiki/X-Forwarded-For "X-Forwarded-For"))を設定します。

```:nginx.conf
    location / {
        proxy_set_header X-Forwarded-for $remote_addr; <------------を追加
        proxy_pass   http://web;
    }
```

location以上に設定できる。

http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=2.232664543.1708492467.1516082737-1679737107.1515976897#proxy_set_header