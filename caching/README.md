# NGINX Caching

응답을 캐싱해두고, 같은 요청이 들어올 경우 저장된 응답을 다시 전달함으로써 응당 속도는 빨라지고, 서버 부하는 줄어듬

오픈소스 NGINX에서 사용 가능한 캐싱 기능은 아래와 같다

- 프록시 캐싱: 백엔드 응답을 디스크에 저장하여 재사용
- FastCGI 캐싱: PHP와 같은 FastCGI 서버 응답을 캐싱
- 캐시 만료/무효화 설정: proxy_cache_valid, proxy_cache_use stale 등으로 세밀하게 제어
- 캐시 경로 및 메모리 영역 설정: proxy_cache_path로 캐시 저장소 구성

## 1. 응답 캐싱 활성화

캐싱을 활성화 하려면, `proxy_cache_path`를 http context의 최상위에 지정한다.

```conf
http {
    # proxy_cache_path 파일경로 key_zone=공유 메모리 이름과 크기
    proxy_cache_path /data/nginx/cache key_zone=mycache: 10m;
}
```

proxy_cache_path의 첫번째 필수 매개변수는 파일을 저장할 디렉토리 경로이며, 두번째 필수 매개변수 `key_zone`는 캐시영역의 이름과 공유 메모리 크기를 지정한다

`key_zone`은 worker 프로세스 간에 캐시 정보를 공유하기 위해 필수

위의 설정이 된 후에, 실제로 서버 응답을 캐싱할 서버 블록에 지정한다

```conf
http {
    # ...
    proxy_cache_path /data/nginx/cache keys_zone=mycache:10m;
    server {
        # 캐시 지정
        proxy_cache mycache;
        location / {
            proxy_pass http://localhost:8000;
        }
    }
}
```

이 때, `keys_zone`에 정의된 크기는 응답 캐시의 총량을 제한하지 않으먀, 제한하려면 `max_size`를 포함시키면 된다.

## 2. 캐싱 제한

기본적으로 응답 캐싱은 무기한으로 가지고 있으며, 최대 설정된 크기를 넘을 경우에만 제거가 된다.

응답 캐시의 유효를 설정하려면, `proxy_cache_valid` 지시어를 사용한다

```conf
# 200, 302 코드가 포함된 응답은 10분간 유효
proxy_cache_valid 200 302 10m;
# 404 코드가 포함된 응답은 1분간 유효
proxy_cache_valid 404 1m;
# 모든 상태 코드가 포함된 응답은 5분간 유효
proxy_cache_valid any 5m;
```

## 3. 캐시 삭제 (NGINX 플러스 기능)

오래된 캐시 파일을 캐시에서 제거할 수도 있다.

이전 버전과 새 버전이 동시에 제공되는 경우를 방지하기 위해서 필요하다.

캐시 삭제는 사용자 정의 HTTP 헤더, HTTP PURGE 메소드가 포함된 요청을 수신하면 제거된다.

<br />

**예시**

아래는 HTTP PURGE 메서드를 사용하는 요청을 식별하여 일치하는 URL을 삭제하는 구성 설정 예시이다.

1. purge 설정

```conf
http {
    # $request_method에 따라서 $purge_method와 같은 새 변수 생성
    map $request_method $purge_method {
        PURGE 1;
        default 0;
    }
}
```

2. 캐싱이 구성된 위치 컨텍스트에 캐시 제거 요청에 대한 조건 지정

```conf
server {
    listen 80;
    server_name www.example.com;

    location / {
        proxy_pass https://localhost:8002;
        proxy_cache mycache;

        proxy_cache_purge $perge_method;
    }
}
```

3. purge 명령어 보내기

```sh
$ curl -X PURGE -D - "https://www.example.com/*"
```

**캐시 삭제 최종 설정**

```conf
http {
    # 캐시 활성화 및 캐시 삭제 활성화
    # level은 디렉터리 구조(캐시 파일을 분산 저장)
    proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:10m purger=on;

    map $request_method $purge_method {
        PURGE 1;
        default 0;
    }

    server {
        listen      80;
        server_name www.example.com;

        location / {
            proxy_pass        https://localhost:8002;
            proxy_cache       mycache;
            proxy_cache_purge $purge_method;
        }
    }

    # 삭제 허용 ip 제한
    geo $purge_allowed {
       default         0;
       10.0.0.1        1;
       192.168.0.0/24  1;
    }

    map $request_method $purge_method {
       PURGE   $purge_allowed;
       default 0;
    }
}
```
