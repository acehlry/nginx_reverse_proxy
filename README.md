# NGINX REVERSE PROXY

참고자료

https://nginx.org/

## nginx 명령어

```sh
start nginx.exe # (exe가 존재하는 경로) window에서 실행

nginx -s signal
nginx -s stop   # fast shutdown
nginx -s quit   # graceful shutdown
nginx -s reload # reloading the configuration file
nginx -s reopen # reopening the log files
```

## 설정 파일 구조

NGINX는 모듈로 구성, 각 모듈은 설정 파일의 명령문(directive)으로 제어

1. 단순 명령문 (Simple Directive)

name과 parameter로 이루어져 있으며, 세미콜론(;)으로 끝남

```conf
worker_processes 1;
```

2. 블록 명령문 (Block Directive)

세미콜론 대신 중괄호로 감싸진 추가 설정들을 포함

<br />

블록 안에 또 다른 명령문을 포함하며, 컨텍스트(context)라고 지칭

ex) events, http, server, location

```sh
# main context 안에 events와 http가 존재
main
 ├── events
 └── http
      |   # http context 안에 server 존재
      └── server # server context 안에 location 존재
           └── location
```

---

## 정적 콘텐츠 서비스

웹서버의 기본 역할인 정적 파일(html, 이미지 등을 제공)

<br />

요청 경로에 따라 다음 두 디렉터리에서 파일을 제공

- /data/www --> HTML 등 일반 웹 파일
- /data/images --> 이미지 파일

**디렉터리를 다음과 같이 구성한다.**

```sh
/data/www    --> index.html 파일을 위치
/data/images --> 이미지 파일을 위치
```

<br />

**설정 파일을 수정한다.**

```conf
http {
    server {
        location / {
            # /로 시작하는 요청을 /data/www 디렉터리로 매핑
            # http://localhost/some/index.html → /data/www/some/index.html
            root /data/www;
        }

        location /images/ {
            # /images/ 로 시작하는 요청을 /data/images 디렉터리로 매핑
            root /data;
        }
    }
}
```

여러 location이 매칭될 경우, 가장 긴 prefix를 가진 블록이 사용된다.

/는 가장 짧은 prefix 이므로 기본 fallback 역할을 한다.

!! Window에서는 상대경로를 제대로 찾지 못할 수 있으니, 절대경로를 사용한다.

```conf
http {
    server {
        location / {
            root ./html;
        }

        location /images/ {
            # alias ./images/;
            alias D:/WORKSPACE/NGINX_REVERSE_PROXY/nginx-1.28.0/images/
        }
    }
}
```

_root와 alias_

1. root는 server 혹은 location에서 지정한 경로 + 요청 URI를 합쳐 실제 파일 경로를 찾는다.

```conf
server {
    location /images/ {
        root /var/www;
    }
}
```

위의 설정 후에 요청을 IP:PORT/images/logo.png로 했을 경우,

<br />

```
root + URI
= /var/www + /images/logo.png
= /var/www/images/logo.png 가 된다
```

2. alias는 location 경로를 실제 파일 경로로 대체한다.

```conf
server {
    location /images/ {
        alias /var/www/images/;
    }
}
```

위의 설정 후에 요청을 IP:PORT/images/logo.png로 했을 경우,

```
alias + URL에서 location prefix 제거
= /var/www/images/ + logo.png
= /var/www/images/logo.png
```

3. root vs alias

- root

  - root + URI
  - location, server 블록에서 사용
  - 일반적인 루트 디렉터리 설정

- alias

  - alias + URI에서 location prefix 제거
  - location 블록에서 사용
  - 특정 location만 다른 디렉터리로 매핑할 경우 사용

---

## 프록시 서버 설정

프록시 - 요청을 받아 다른 서버에 전달, 응답을 클라이언트에 전달

**백엔드 서버 정의**

8080 포트에서 동작하며, /api/v1을 루트로 사용

/backend/api/v1에 index.html을 만들어 테스트 진행해보자.

```conf
    server {
        listen 8080;
        root ./backend/api/v1;

        location / {

        }
    }
```

IP:8080 요청시 /backend/api/v1의 index.html을 가져온다.

<br />

**프록시 서버 설정**

```conf
server {
    location / {
        # root ./html;
        proxy_pass http://localhost:8080;
    }
}
```

<br />

**정규식 기반 이미지 처리**

`/images/` 대신 파일 확장자로 구분

```conf
# /images/ --> ~ \.(gif|jpg|png)$

location ~ \.(gif|jpg|pgn)$ {
    root /data/images;
}
```

- `~`는 정규식 매칭을 나타냄
- `.gif`, `.jpg`, `.png`로 끝나는 URI는 /data/images에서 제공

이때 NGINX는 다음 순서로 location을 선택한다.

1. prefix 기반 location 검삭 (가장 긴 prefix 기억)
2. 정규식 location 검사
3. 정규식 매칭시, 해당 블록 사용 아닐 경우 기억하고 있는 prefix 블록사용
