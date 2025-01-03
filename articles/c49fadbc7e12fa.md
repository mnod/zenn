---
title: "KeyCloak をリバースプロキシの背後に配置してみる"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["keycloak", "nginx"]
published: true
---

# 概要

KeyCloak を Reverse Proxy(nginx) の背後に配置してみる。
フロント側、バックエンド側とも https 通信で暗号化する。

# 参考

https://www.keycloak.org/server/reverseproxy

# KeyCloak 設定

compose.yaml の keycloak の箇所は以下の感じ。

```
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    volumes:
      - ./keycloak:/work/keycloak
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
      KC_DB: postgres
      KC_DB_PASSWORD: password
      KC_DB_USERNAME: keycloak
      KC_DB_URL_PORT: 5432
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: keycloak
      KC_HTTPS_CERTIFICATE_FILE: /work/keycloak/example.crt
      KC_HTTPS_CERTIFICATE_KEY_FILE: /work/keycloak/example.key
      KC_PROXY_HEADERS: xforwarded
    command:
      - "start-dev"
    ports:
      - 8443:8443
    depends_on:
      - postgres
```

# nginx 設定

リバースプロキシ(nginx) の設定は以下の感じ。
mod_auth_openidc を使って RP を構築するとき、ここで指定する証明書を信頼させる必要がある。

```
server {
        listen 80 ssl;
        server_name ＜バーチャルホスト名＞;
        ssl_certificate ＜証明書＞;
        ssl_certificate_key ＜秘密鍵＞;

        location / {
                proxy_pass https://＜KyeCloakホスト＞:8443;
                proxy_set_header  Host              $http_host;
                proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
                proxy_set_header  X-Forwarded-Proto $scheme;
                proxy_set_header  X-Forwarded-port  $server_port;
        }
}
```

# その他

バックエンド側 KeyCloak を http で通信するときの設定がうまくできなかった。
