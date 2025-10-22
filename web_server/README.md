# NGINX WEB SERVER

NGINX를 웹서버로 구성하는 방법

## virtual servers

설정 파일에는 하나 이상의 가상 서버 블록을 지정해야 한다.

NGINX는 요청을 처리할 경우 요청을 처리할 가상 서버를 선택한다.

기본 서버는 default_server; 파라미터를 주지 않는한 첫번째의 server를 사용한다.

(http 컨텍스트 안에 여러 가상 서버를 추가할 수 있다.)

```conf
http {
    server {
        # 서버 설정
    }
}
```

서버 블록에는 일반적으로 요청을 처리할 IP와 PORT가 지정 되어있다.

IPv4, IPv6 모두 사용 가능하며, IPv6는 대괄호로 묶는다.

```conf
http {
    server {
        # 포트를 생략할 경우 표준 포트가 사용된다
        listen 127.0.0.1:8080;
        # 서버 설정 추가
    }
}
```

### 매칭 우선순위 (server 블록 찾기)

요청이 들어올 경우 IP주소와 포트가 일치하는 서버 블록드 중에서 Host 헤더 값을 `server_name`과 비교해서 가장 잘 맞는 블록을 선택한다.

1. 정확한 이름

2. 와일드 카드 이름 (와일드 카드 위치 앞, 뒤 순으로)

- \*.example.org
- mail.\*

3. 정규 표현식

순으로 우선순위를 결정한다.

```conf
# 요청 Host가 www.example.org일 경우
server {
    listen 80;
    server_name example.org www.example.org;
}

# 요청 Host가 blog.example.org일 경우
server {
    listen 80;
    server_name *.example.org;
}

# 요청 Host가 www123.example.org일 경우
server {
    listen 80;
    server_name ~^www\d+\.example\.org$;
}
```

## location

`location` 컨텍스트는 URI 패턴에 따라 처리 방식을 결정하는 설정 단위로, 각 `location`은 자신만의 처리 시나리오를 갖는다.

<br />

- 요청을 프록시 서버로 전달 (proxy_pass)
- 정적 파일을 반환 (root, index)
- URI를 수정하거나 리다이렉션 ( rewrite, return )
- 특정 오류 코드 반환 및 커스텀 에러 페이지 설정 (error_page)

`location`은 `server` 컨텐스트 안에 위치한다.

### 매칭 방식

1. Prefix(접두어)

URL가 Prefix로 시작하면 매칭된다

```conf
# /images/ 로 시작할 경우 매칭
location /images/ {
    root /var/www/static;
}

# /static/ 로 시작할 경우 매칭
location ^~ /static/ {
    root /var/www/static_files;
}
```

2. Regex(정규 표현식)

```conf
# 대소문자를 구분하여, .php로 요청이 올경우 매칭
location ~ \.php$ {
    proxy_pass http://php_backend;
}

# 대소문자를 구분하지 않으며, jpg, jpeg, pgn, gif로 요청이 올경우 매칭
location ~* \.(jpg|jpeg|png|gif)$ {
    root /var/www/images;
}
```

### 매칭 우선순위

NGINX에서는 아래의 순서로 location 블록을 선택

1. `=` 정확한 매칭

2. `^~` 접두어

3. 가장 긴 접두어 매칭

- ex) /images/ vs /images/icons/ 시 후자를 우선 선택

4. 정규 표현식 매칭

- 첫번째로 매칭된 블록을 사용

5. 그 외 접두어 매칭이 안되면, 가장 긴 접두어 블록을 사용

### 처리 방식

1. 정적 파일

root는 실제 파일 경로를 지정하고, URI를 뒤에 붙여서 전체 경로를 만든다.

```conf
# /images/ 요청시 /data/images/에서 찾음
# root 디렉터리에 전체 URI를 그대로 붙여서 계산
location /images/ {
    root /data;
}

# /images2/ 요청시 /data/에서 찾음
# alias는 location 경로를 제거한 뒤 나머지를 붙여서 계산
location /images2/ {
    alias /data;
    autoindex on;
}
```

`autoindex`를 붙여 요청시 디렉토리 목록을 볼 수 있다

2. 프록시

요청을 다른 서버로 전달하고, 응답을 클라이언트에 반환

```conf
location / {
    proxy_pass http://www.example.com;
}
```

### location 매칭 예시

```conf
server {
    listen 80;
    server_name example.com;

    # 1. 정확한 매칭 (=)
    location = / {
        return 200 "Welcome to the homepage!";
    }

    # 2. 접두어 매칭 (prefix)
    location /images/ {
        root /var/www/static;
    }

    # 3. ^~ 접두어 우선 매칭
    location ^~ /static/ {
        root /var/www/static_files;
    }

    # 4. 정규 표현식 매칭 (대소문자 구분)
    location ~ \.php$ {
        proxy_pass http://php_backend;
    }

    # 5. 정규 표현식 매칭 (대소문자 무시)
    location ~* \.(jpg|jpeg|png|gif)$ {
        root /var/www/images;
    }

    # 6. 기본 처리
    location / {
        proxy_pass http://default_backend;
    }
}
```

## NGINX 변수 사용

1. `$` 기호로 시작하는 런타임 계산 값이며,

2. 요청의 상태나 속성에 따라 자동으로 값이 설정된다

3. 설정 파일에서 디렉티브 인자로 사용한다

### 자주 사용되는 기본 변수

- $remote_addr: 클라이언트 IP 주소
- $uri: 요청된 URI
- $host: 요청의 Host 헤더 값
- $requst_method: 요청 메서드
- $http_user_agent: 클라이언트의 User-Agent
- $args: 쿼리 문자열 (예시: ?id=admin)
- $status: 응답 상태코드

### 사용자 정의 변수 설정하기

```conf
# set
# 고정된 값이나 간단한 표현식으로 변수에 직접 할당
# 주로 location 블록 안에서 사용
set $my_var "hello";


# map
# 조건에 따라 변수 값을 설정
map $request_method $method_type {
    GET     "read";  # request_method가 GET이면 method_type = "read";
    POST    "write"; # request_method가 POST이면 method_type = "write";
    default "other"; # 그 외, method_type = "other";
}

# geo
# IP 주소에 따라서 변수 값 설정
# 주로 접근 제어, 지역 기반 처리에서 사용
geo $is_local {
    default         0; # 기본적으로 0 할당
    192.168.0.0/16  1; # 192.168.x.x이면 1 할당
    10.0.0.1        1; # 10.0.0.1이면 1 할당
}
```

사용 예시

```conf
server {
    location / {
        # POST 요청일 경우 403 리턴
        if ($request_method = POST) {
            return 403;
        }

        # 나머지의 경우 프록시로 전달
        proxy_pass http://backend;
    }
}
```

## 특정 상태 코드 반환

return 지시문을 사용하여 응답 코드, 리다이렉션 URL나 텍스트를 반환할 수 있다.

`return`은 `location`과 `server`에 모두 포함될 수 있다

```conf
location /wrong/url {
    return 404;
}

location /permanently/moved/url {
    return 301 http://www.example.com/moved/here;
}
```

_에러 페이지 설정_

아래와 같이 설정하면, /admin 요청시 403을 반환하여 /custom_403.html 파일을 보여준다

```conf
location /admin {
    return 403;
}

error_page 403 /custom_403.html;

location = /custom_403.html {
    root /var/www/errors;
}
```

## 요청 URI 다시 쓰기

클라이언트가 요청한 URI를 다른 URI로 변경하여 처리한다.

- 정규식: 요청 URI가 이 패턴과 일치해야한다.
- 대체 URI: 일치한 경우 이 URI로 변경한다.
- flag(옵션): last, break, redirect, permanent 등
  - last: URI를 바꾸고 새 URI에 맞는 location 블록을 다시 검색(URI 이동에 사용)
  - break: URI를 바꾸고 현재 location 블록에서만 처리, location 재검색 안함(간단한 URI 변경에 사용)
  - redirect: 302 응답을 전송함, 임시 이동일 경우 사용
  - permanent: 301 응답을 전송함, 브라우저는 새 주소를 기억함(SEO에도 영향)

```conf
location /users/ {
    # rewrite <정규식> <대체할 URI> [flag];
    rewrite ^/users/(.*)$ /show?user=$1 break;
    proxy_pass http://backend;
}
```

/users/john 요청 --> /show?user=john으로 변경,

- $1 = john으로 변경되어, /show?user=john으로 된다

## 응답 본문 수정하기

프록시 서버에서 받은 응답의 본문 내용을 문자열 치환해서 클라이언트에 전달

- sub_filter: 응답 본문에서 문자열 치환
- sub_filter_once: on일경우 한번만 치환, off일 경우 여러번 치환

```conf
sub_filter <찾을 문자열> <바꿀 문자열>;
sub_filter_once on|off;
```

아래의 예시는 HTML 응답에서 링크와 이미지 경로를 클라이언트의 HOST 기반으로 변경한다.

```conf
location / {
    sub_filter 'href="http://127.0.0.1:8080/' 'href="https://$host/';
    sub_filter 'img src="http://127.0.0.1:8080/' 'img src="https://$host/';
    sub_filter_once on;
}

```

## 에러 핸들링

아래 설정을 보면,

클라이언트가 요청한 파일이 없을 경우, /404.html 페이지를 보여주며, 즉시 404를 반환하지 않는다(return 404;가 담당)

```conf
error_page 404 /404.html;
```

_에러 코드 변경 + 리다이렉션 (SEO나 URL 구조 변경 시 유용)_

/old/path.html 요청시 파일이 없으면 404 발생

error_page가 404를 301로 변경하고 새 주소로 리다이렉션

브라우저는 /new/path.html로 이동, 영구 이동(301)으로 기억

```conf
location /old/path.html {
    error_page 404 =301 http://example.com/new/path.html;
}
```

_백엔드로 요청 넘기기 (내부 리다이렉션)_

```conf
location /images/ {
    root /data/www;
    open_file_cache_errors off;
    error_page 404 = /fetch$uri;
}

location /fetch/ {
    proxy_pass http://backend/;
}
```

/images/some/file 요청 시 파일이 없으면 → 404 발생

error_page가 /fetch/images/some/file로 내부 URI 변경

NGINX는 새 URI에 맞는 location을 다시 찾고 → /fetch/로 매칭

결과적으로 요청은 http://backend/images/some/file로 프록시됨
