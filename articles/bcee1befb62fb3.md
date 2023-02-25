---
title: "Debian11 で HAProxy"
emoji: "🎎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["debian","haproxy"]
published: true
---

# 環境について

Debian11 に HAProxy によるロードバランサーを構築してみます。


# L4ロードバランシング

まずは、単純にラウンドロビンする設定をテストします。

## パッケージ導入

apt でインストールします。
```
# apt install --no-install-recommends haproxy
```

設定ファイルを用意します。
```:/etc/haproxy/haproxy.cfg
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private
        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
frontend web_proxy
        bind *:80
        default_backend    web_servers
        option             forwardfor
backend web_servers
        balance            roundrobin
        server             debian11-2 10.2.1.242:80 check
        server             debian11-3 10.2.1.243:80 check
```

コンフィグテストにパスしたら、haproxy をリスタートします。
```
# haproxy -f /etc/haproxy/haproxy.cfg -c
Configuration file is valid

# systemctl restart haproxy
```

外部からアクセスした際のログが下記のようになります。
```
Feb 19 04:02:26 debian11 haproxy[4418]: 10.2.0.1:54786 [19/Feb/2023:04:02:26.864] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:02:28 debian11 haproxy[4418]: 10.2.0.1:54792 [19/Feb/2023:04:02:28.311] web_proxy web_servers/debian11-3 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:02:29 debian11 haproxy[4418]: 10.2.0.1:54794 [19/Feb/2023:04:02:29.217] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:02:30 debian11 haproxy[4418]: 10.2.0.1:54796 [19/Feb/2023:04:02:30.447] web_proxy web_servers/debian11-3 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
```

## 疑似障害

リアルサーバの一台で nginx を停止してみます。
nginx ダウンを検知してから、タイムアウトまでの間に外部からアクセスすると、503エラーになります。
タイムアウト後は、nginx を停止したリアルサーバが切り離されます。
```
Feb 19 04:03:01 debian11 haproxy[4418]: 10.2.0.1:56930 [19/Feb/2023:04:03:01.793] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:03:03 debian11 haproxy[4418]: [WARNING] 049/040303 (4418) : Server web_servers/debian11-3 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Feb 19 04:03:03 debian11 haproxy[4418]: Server web_servers/debian11-3 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Feb 19 04:03:03 debian11 haproxy[4418]: Server web_servers/debian11-3 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.

Feb 19 04:03:05 debian11 haproxy[4418]: 10.2.0.1:56932 [19/Feb/2023:04:03:02.911] web_proxy web_servers/debian11-3 0/0/-1/-1/3009 503 221 - - SC-- 1/1/0/0/3 0/0 "GET / HTTP/1.1"

Feb 19 04:03:42 debian11 haproxy[4418]: 10.2.0.1:49438 [19/Feb/2023:04:03:42.071] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:03:43 debian11 haproxy[4418]: 10.2.0.1:49446 [19/Feb/2023:04:03:43.550] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:03:44 debian11 haproxy[4418]: 10.2.0.1:55286 [19/Feb/2023:04:03:44.624] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
```


## 切り戻し

先ほど停止した nginx を起動します。
自動的にロードバランサーに切り戻されます。
```
Feb 19 04:05:07 debian11 haproxy[4418]: [WARNING] 049/040507 (4418) : Server web_servers/debian11-3 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Feb 19 04:05:07 debian11 haproxy[4418]: Server web_servers/debian11-3 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Feb 19 04:05:07 debian11 haproxy[4418]: Server web_servers/debian11-3 is UP, reason: Layer4 check passed, check duration: 0ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.

Feb 19 04:05:14 debian11 haproxy[4418]: 10.2.0.1:45236 [19/Feb/2023:04:05:14.432] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:05:15 debian11 haproxy[4418]: 10.2.0.1:45242 [19/Feb/2023:04:05:15.392] web_proxy web_servers/debian11-3 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:05:16 debian11 haproxy[4418]: 10.2.0.1:45248 [19/Feb/2023:04:05:16.236] web_proxy web_servers/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
```



# L7ロードバランシング

設定ファイルを用意します。
パスが /app1 のときは web_server01 へ、/app2 のときは web_server02 へ、デフォルトでは web_server01 へ向くように設定をします。
```:/etc/haproxy/haproxy.cfg
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private
        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
frontend web_proxy
        bind *:80
        default_backend    web_server01
        option             forwardfor
        acl is-app1 path_beg /app1
        acl is-app2 path_beg /app2
        use_backend web_server01 if is-app1
        use_backend web_server02 if is-app2
backend web_server01
        balance            roundrobin
        http-request replace-path /app1(/)?(.*) /\2
        server             debian11-2 10.2.1.242:80 check
backend web_server02
        balance            roundrobin
        http-request replace-path /app2(/)?(.*) /\2
        server             debian11-2 10.2.1.243:80 check
```


コンフィグテストに問題なければ、 haproxy をリスタートします。
```
# haproxy -f /etc/haproxy/haproxy.cfg -c
Configuration file is valid

# systemctl restart haproxy
```


`curl 10.2.0.240` でアクセスすると web_server01 へ向きます。
```
Feb 19 04:10:01 debian11 haproxy[4436]: 10.2.0.1:53554 [19/Feb/2023:04:10:01.913] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:10:03 debian11 haproxy[4436]: 10.2.0.1:53568 [19/Feb/2023:04:10:03.247] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
```

`curl 10.2.0.240/app1` でアクセスすると web_server01 へ向きます。
```
Feb 19 04:10:34 debian11 haproxy[4436]: 10.2.0.1:36948 [19/Feb/2023:04:10:34.131] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app1 HTTP/1.1"
Feb 19 04:10:35 debian11 haproxy[4436]: 10.2.0.1:36958 [19/Feb/2023:04:10:35.345] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app1 HTTP/1.1"
```

`curl 10.2.0.240/app2` でアクセスすると web_server02 へ向きます。
```
Feb 19 04:10:59 debian11 haproxy[4436]: 10.2.0.1:36752 [19/Feb/2023:04:10:59.371] web_proxy web_server02/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app2 HTTP/1.1"
Feb 19 04:11:01 debian11 haproxy[4436]: 10.2.0.1:36766 [19/Feb/2023:04:11:01.499] web_proxy web_server02/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app2 HTTP/1.1"
```


## 疑似障害

web_server02 で nginx を停止してみます。
```
Feb 19 04:11:57 debian11 haproxy[4436]: [WARNING] 049/041157 (4436) : Server web_server02/debian11-2 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Feb 19 04:11:57 debian11 haproxy[4436]: Server web_server02/debian11-2 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Feb 19 04:11:57 debian11 haproxy[4436]: [NOTICE] 049/041157 (4436) : haproxy version is 2.2.9-2+deb11u4
Feb 19 04:11:57 debian11 haproxy[4436]: Server web_server02/debian11-2 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Feb 19 04:11:57 debian11 haproxy[4436]: [NOTICE] 049/041157 (4436) : path to executable is /usr/sbin/haproxy
Feb 19 04:11:57 debian11 haproxy[4436]: backend web_server02 has no server available!
Feb 19 04:11:57 debian11 haproxy[4436]: [ALERT] 049/041157 (4436) : backend 'web_server02' has no server available!
Feb 19 04:11:57 debian11 haproxy[4436]: backend web_server02 has no server available!
```


`curl 10.2.0.240` でアクセスすると web_server01 へ向きます。
```
Feb 19 04:12:39 debian11 haproxy[4436]: 10.2.0.1:55330 [19/Feb/2023:04:12:39.762] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
Feb 19 04:12:40 debian11 haproxy[4436]: 10.2.0.1:55340 [19/Feb/2023:04:12:40.723] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET / HTTP/1.1"
```


`curl 10.2.0.240/app1` でアクセスすると web_server01 へ向きます。
```
Feb 19 04:13:10 debian11 haproxy[4436]: 10.2.0.1:37136 [19/Feb/2023:04:13:10.183] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app1 HTTP/1.1"
Feb 19 04:13:11 debian11 haproxy[4436]: 10.2.0.1:37138 [19/Feb/2023:04:13:11.481] web_proxy web_server01/debian11-2 0/0/0/1/1 200 214 - - ---- 1/1/0/0/0 0/0 "GET /app1 HTTP/1.1"
```

`curl 10.2.0.240/app2` でアクセスすると 503エラーとなります。

```
$ curl 10.2.0.240/app2
<html><body><h1>503 Service Unavailable</h1>
No server is available to handle this request.
</body></html>
```
```
Feb 19 04:13:36 debian11 haproxy[4436]: 10.2.0.1:54090 [19/Feb/2023:04:13:36.655] web_proxy web_server02/<NOSRV> 0/-1/-1/-1/0 503 221 - - SC-- 1/1/0/0/0 0/0 "GET /app2 HTTP/1.1"
```


OS起動時に HAProxy が起動するのが困る場合には、下記で自動起動を停止しておきます。
```
# systemctl disable haproxy
Synchronizing state of haproxy.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable haproxy
Removed /etc/systemd/system/multi-user.target.wants/haproxy.service.
```
