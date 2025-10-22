# Nginx Conf Module

NGINX 설정을 모듈화하는 디렉토리

## include 지시어

`include`를 사용하여 현재 설정의 일부처럼 포함

<br />

와일드 카드(`*`)를 사용하여, .conf 파일 전부를 불러올 수 있음

```conf
include /etc/nginx/conf.d/*.conf;
```

## 사용예시

아래와 같은 구조로 구성 되었을 경우

```
├── nginx.conf
└── conf.d/
    ├── http.conf
    ├── stream.conf
    ├── exchange-enhanced.conf

```

### nginx.conf 파일

```conf
user  nobody;
worker_processes  auto;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # 여기서 feature별 설정 파일을 불러옴
    include conf.d/http.conf;
    include conf.d/exchange-enhanced.conf;

    sendfile        on;
    keepalive_timeout  65;
}

# TCP/UDP용 스트림 설정
stream {
    include conf.d/stream.conf;
}

```

### conf.d 디렉터리 파일

`conf.d/http.conf`

```conf
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

`conf.d/steram.conf`

```conf
# TCP 포트 포워딩
server {
    listen 3306;
    proxy_pass 192.168.0.1:3306;
}
```

`conf.d/exchange-enhanced.conf`

```conf
# Exchange 서버 관련 리버스 프록시 설정
server {
    listen 443 ssl;
    server_name mail.example.com;

    ssl_certificate     /etc/ssl/certs/mail.crt;
    ssl_certificate_key /etc/ssl/private/mail.key;

    location /owa/ {
        proxy_pass https://exchange_backend/owa/;
    }
}
```

## 장점

1. 여러 서비스에 프록시를 할 경우, 유지보수가 용이
2. 여러 사람이 설정을 수정할 때, 각 작업자가 맡은 파일만 수정
3. 배포 환경(운영, 테스트)에 따라서 다르게 설정 가능
   - include 경로를 운영, 테스트 등의 환경별로 지정
4. 도커나 CI/CD를 구성할 경우, 필요 기능만 include 하여 로드
