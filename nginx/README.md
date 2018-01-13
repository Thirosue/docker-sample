# nginx 

## nginxで流量制限

> httpディレクティブに[limit_req_zone](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_zone "limit_req_zone")を追加

```
http {
    #流量制限を追加
    limit_req_zone $binary_remote_addr zone=limit_req_by_ip:10m rate=50r/s;
    limit_req_log_level error;
    limit_req_status 503;
```

> ロケーション毎に制限を適応する

```
    location / {
        #制限なし
    }

    location /limit/ {
        #秒間50を超えたリクエストは503を返す
        limit_req zone=limit_req_by_ip burst=50 nodelay;
    }
```

## 接続元IPアドレスでフィルタ

> `blacklist.conf`を準備

```blacklist.conf
  limit_req zone=limit_req_limited burst=1;
  
  if ($remote_addr ~ " ?172\.20\.0\.4$" ) { set $limit_req_key $binary_remote_addr; } #ブラックリストのIPは遅延実行する
  if ($http_x_forwarded_for ~ " ?172\.20\.0\.4$" ) { set $limit_req_key $binary_remote_addr; } #ブラックリストのIPは遅延実行する
```

以下を参考に設定を実施しました。<br/>
[Nginx の limit_req/limit_conn を特定の条件でのみ適用する/しない - Quita](https://qiita.com/ral/items/ffcd8670d9ef2484fc8e "")

> ロケーション毎に制限を適応する

```
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location /limit/ {
        include /etc/nginx/conf.d/blacklist.conf; #blacklist設定をディレクディブ毎に設定 -> 一部APIの許可のみ制限するなど
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
```

## 設定検証

+ コンテナ起動

```:bash
$  
$ docker-compose up -d
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
4e25f9baf911        httpd:alpine        "httpd-foreground"       3 seconds ago       Up 4 seconds        80/tcp                 ab1
04343e39793c        httpd:alpine        "httpd-foreground"       3 seconds ago       Up 5 seconds        80/tcp                 ab2
f61bb508287c        nginx_limit         "nginx -g 'daemon of…"   3 seconds ago       Up 5 seconds        0.0.0.0:8080->80/tcp   nginx_limit
```

+ すべてのコンテナに入り(`docker exec`)疎通及び接続元IP確認 

> ab1/ab2コンテナに入り(`docker exec`)、`ab`実行

```:bash
$ docker exec -ti ab1 /bin/ash
/usr/local/apache2 # ab -n 1 -c 1 http://nginx/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      1
Time taken for tests:   0.000 seconds
Complete requests:      1
Failed requests:        0
Total transferred:      845 bytes
HTML transferred:       612 bytes
Requests per second:    2785.52 [#/sec] (mean)
Time per request:       0.359 [ms] (mean)
Time per request:       0.359 [ms] (mean, across all concurrent requests)
Transfer rate:          2298.59 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     0    0   0.0      0       0
Waiting:        0    0   0.0      0       0
Total:          0    0   0.0      0       0
```

> nginx_limitコンテナに入り(`docker exec`)、`access.log`を参照し、接続元IPを確認

```:bash
$ docker exec -ti nginx_limit /bin/ash
/ # tail -f /var/log/nginx/access.log
172.20.0.4 - - [13/Jan/2018:06:59:12 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3" "-"
172.20.0.3 - - [13/Jan/2018:06:59:22 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3" "-"
```

> ab1/ab2コンテナに入り(`docker exec`)、`ab`実行

```:bash
$ docker exec -ti ab1 /bin/ash
/usr/local/apache2 # ab -n 10 -c 10 http://nginx/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      1
Time taken for tests:   0.000 seconds
Complete requests:      1
Failed requests:        0
Total transferred:      845 bytes
HTML transferred:       612 bytes
Requests per second:    2785.52 [#/sec] (mean)
Time per request:       0.359 [ms] (mean)
Time per request:       0.359 [ms] (mean, across all concurrent requests)
Transfer rate:          2298.59 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     0    0   0.0      0       0
Waiting:        0    0   0.0      0       0
Total:          0    0   0.0      0       0
```

+ 流量制限発動確認

> 対象IPからの遅延実行対象外パスへのアクセスがエラーとなっていないことを確認
> 対象IPからの遅延実行対象パスへのアクセスがエラー(503)となっていることを確認

```:bash
$ docker exec -ti ab1 /bin/ash
#対象IPからの流量制限対象外パスへのアクセス
/usr/local/apache2 # ab -n 10 -c 10 http://nginx/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      10
Time taken for tests:   0.002 seconds
Complete requests:      10
Failed requests:        0 # <------------------------------------ Fail=0
Total transferred:      8450 bytes
HTML transferred:       6120 bytes
Requests per second:    4280.82 [#/sec] (mean)
Time per request:       2.336 [ms] (mean)
Time per request:       0.234 [ms] (mean, across all concurrent requests)
Transfer rate:          3532.51 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      1       1
Processing:     0    1   0.4      1       1
Waiting:        0    1   0.4      1       1
Total:          1    1   0.2      1       2
ERROR: The median and mean for the initial connection time are more than twice the standard
       deviation apart. These results are NOT reliable.

Percentage of the requests served within a certain time (ms)
  50%      1
  66%      1
  75%      2
  80%      2
  90%      2
  95%      2
  98%      2
  99%      2
 100%      2 (longest request)

#対象IPからの流量制限対象パスへのアクセス
/usr/local/apache2 # ab -n 10 -c 10 http://nginx/limit/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /limit/
Document Length:        29 bytes

Concurrency Level:      10
Time taken for tests:   0.503 seconds
Complete requests:      10
Failed requests:        8 # <------------------------------------ 3回目以降のアクセスが想定通りFailしている
   (Connect: 0, Receive: 0, Length: 8, Exceptions: 0)
Non-2xx responses:      8
Total transferred:      6368 bytes
HTML transferred:       4354 bytes
Requests per second:    19.90 [#/sec] (mean)
Time per request:       502.620 [ms] (mean)
Time per request:       50.262 [ms] (mean, across all concurrent requests)
Transfer rate:          12.37 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    1   0.1      1       1
Processing:     1   51 158.4      1     501
Waiting:        0   51 158.4      1     501
Total:          1   52 158.4      2     502

Percentage of the requests served within a certain time (ms)
  50%      2
  66%      2
  75%      2
  80%      2
  90%    502
  95%    502
  98%    502
  99%    502
 100%    502 (longest request)
```

+ 流量制限対象外確認

> 対象外IPからの遅延実行対象外パスへのアクセスがエラーとなっていないことを確認
> 対象外IPからの遅延実行対象パスへのアクセスがエラーとなっていないことを確認

```:bash
$ $ docker exec -ti ab2 /bin/ash
#対象外IPからの流量制限外対象パスへのアクセス
/usr/local/apache2 # ab -n 10 -c 10 http://nginx/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /
Document Length:        612 bytes

Concurrency Level:      10
Time taken for tests:   0.002 seconds
Complete requests:      10
Failed requests:        0 # <------------------------------------ Fail=0
Total transferred:      8450 bytes
HTML transferred:       6120 bytes
Requests per second:    5425.94 [#/sec] (mean)
Time per request:       1.843 [ms] (mean)
Time per request:       0.184 [ms] (mean, across all concurrent requests)
Transfer rate:          4477.46 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.2      1       1
Processing:     0    1   0.2      1       1
Waiting:        0    0   0.2      1       1
Total:          1    1   0.0      1       1
ERROR: The median and mean for the initial connection time are more than twice the standard
       deviation apart. These results are NOT reliable.
ERROR: The median and mean for the waiting time are more than twice the standard
       deviation apart. These results are NOT reliable.

Percentage of the requests served within a certain time (ms)
  50%      1
  66%      1
  75%      1
  80%      1
  90%      1
  95%      1
  98%      1
  99%      1
 100%      1 (longest request)

#対象外IPからの流量制限対象パスへのアクセス
/usr/local/apache2 # ab -n 10 -c 10 http://nginx/limit/
This is ApacheBench, Version 2.3 <$Revision: 1807734 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking nginx (be patient).....done


Server Software:        nginx/1.13.1
Server Hostname:        nginx
Server Port:            80

Document Path:          /limit/
Document Length:        29 bytes

Concurrency Level:      10
Time taken for tests:   0.002 seconds
Complete requests:      10
Failed requests:        0 # <------------------------------------ Fail=0
Total transferred:      2600 bytes
HTML transferred:       290 bytes
Requests per second:    5330.49 [#/sec] (mean)
Time per request:       1.876 [ms] (mean)
Time per request:       0.188 [ms] (mean, across all concurrent requests)
Transfer rate:          1353.44 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.1      0       1
Processing:     0    1   0.2      1       1
Waiting:        0    1   0.2      1       1
Total:          1    1   0.1      1       1
ERROR: The median and mean for the initial connection time are more than twice the standard
       deviation apart. These results are NOT reliable.

Percentage of the requests served within a certain time (ms)
  50%      1
  66%      1
  75%      1
  80%      1
  90%      1
  95%      1
  98%      1
  99%      1
 100%      1 (longest request)
```
