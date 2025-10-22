# 기본 기능

NGINX는 하나의 마스터 프로세스, 하나 이상의 워커 프로세스를 사용

<br />

캐싱이 활성화 된 경우, cache loader와 cache manager 프로세스도 실행

<br />

## 마스터 프로세스

- 설정 파일을 읽고 평가
- 워커 프로세스를 관리

## 워커 프로세스

- 실제 요청을 처리
- 운영체제에 따라 매커니즘을 활용하여, 워커 프로세스에 효율적으로 분산
- nginx.conf의 worker_processes 지시어를 통해 정의
- **고정된 숫자 or CPU 코어 수에 따라 자동으로 조정되도록 구성**

---

# NGINX 설정 파일

일반적으로 다음 중 하나의 경로를 사용

- /usr/local/nginx/conf
- /etc/nginx
- /usr/local/etc/nginx

<br />

아래의 명령어를 사용하여 구성파일의 경로를 출력할 수 있다

```sh
# linux or macOS
nginx -V 2>&1 | awk -F: '/configure arguments/ {print $2}' | xargs -n1

# powershell
nginx -V 2>&1 | Select-String "configure arguments" | ForEach-Object {
    ($_ -split ":")[1] -split " "
}
```

```sh
.\nginx -V 2>&1 | Select-String "configure arguments" | ForEach-Object {
    ($_ -split ":")[1] -split " "
}

--with-cc=cl
--builddir=objs.msvc8
--with-debug
--prefix=
--conf-path=conf/nginx.conf
--pid-path=logs/nginx.pid
--http-log-path=logs/access.log
--error-log-path=logs/error.log
--sbin-path=nginx.exe
--http-client-body-temp-path=temp/client_body_temp
--http-proxy-temp-path=temp/proxy_temp
--http-fastcgi-temp-path=temp/fastcgi_temp
--http-scgi-temp-path=temp/scgi_temp
--http-uwsgi-temp-path=temp/uwsgi_temp
--with-cc-opt=-DFD_SETSIZE=1024
--with-pcre=objs.msvc8/lib/pcre2-10.39
--with-zlib=objs.msvc8/lib/zlib-1.3.1
--with-http_v2_module
--with-http_realip_module
--with-http_addition_module
--with-http_sub_module
--with-http_dav_module
--with-http_stub_status_module
--with-http_flv_module
--with-http_mp4_module
--with-http_gunzip_module
--with-http_gzip_static_module
--with-http_auth_request_module
--with-http_random_index_module
--with-http_secure_link_module
--with-http_slice_module
--with-mail
--with-stream
--with-stream_realip_module
--with-stream_ssl_preread_module
--with-openssl=objs.msvc8/lib/openssl-3.0.15
--with-openssl-opt='no-asm
no-tests
-D_WIN32_WINNT=0x0501'
--with-http_ssl_module
--with-mail_ssl_module
--with-stream_ssl_module
```

---

# NGINX 기능별 파일을 구분하여 사용

<br />

구성을 쉽게 유지, 관리하려면 `/etc/nginx/conf.d` 디렉터리에 저장된 기능별 파일 세트로 분할하여 `nginx.conf` 파일의 include 문을 사용하여 내용을 참조할 수 있다.

```conf
include conf.d/http;
include conf.d/stream;
include conf.d/exchange-enhanced;
```

---

# Contexts

<br />

컨텍스트라고 불리는 최상위 구문들은 다양한 트래픽 유형에 적용되는 지시문을 그룹화한다.

1. events

- 일반적인 연결 처리 (General connection processing)

2. http

- HTTP traffic

3. mail

- Mail traffic

4. stream

- TCP and UDP traffic

<br />

위 컨텍스트 밖의 명령문은 main context에 있다고 한다.
