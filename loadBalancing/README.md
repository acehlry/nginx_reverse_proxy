# 로드 밸런싱

로드 밸런싱은 리소스 활용도 최적화, 처리량 최대화, 대기 시간 감소, 내결함성 구성 보장을 위해 일반적으로 사용된다.

## 1. Proxy HTTP Traffic to a Group of Servers

NGINX는 서버 그룹에 로드 밸런싱을 수행한다

<br />

`upstream` 지시어를 사용하여 로드밸런싱을 수행

`upstream` 지시어는 `http` context 블럭 안에 배치한다

`upstream` context 안에 `server` 지시어를 사용하며, NGINX에서 사용하는 virtual server와 혼동하면 안된다.

```conf
http {
    upstream backend {
        # weidht - 가중치 설정
        # 기본값은 1이며, 숫자가 클수록 요청을 많이 받음
        server backend1.example.com weight=5;
        server backend2.example.com;

        # backup - 백업서버 설정
        # 평소에는 사용하지 않고, 서버가 다운되었을 때,
        # 트래픽을 받음
        server 192.0.0.1 backup;
    }
}
```

들어오는 요청을 서버 그룹에 전달하기 위해 proxy_pass 지시문에 지정된다.

fastcgi와 같은 대체 프로토콜을 사용하는 경우, proxy 대신 다른 지시어를 사용한다.

- fastcgi_pass
- memchached_pass
- scgi_pass
- uwsgi_pass

```conf
http {
    # 로드밸런싱할 서버 그룹
    upstream backend {
        server backend1.example.com weight=5;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }

    # NGINX에서 실행되는 가상 서버가 모든 요청을
    # upstream에 정의된 백엔드 그룹에 전달
    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```

## 2. HTTP 로드 밸런싱 종류

NGINX는 4가지의 로드밸런싱을 지원

- Round Robin
- Least Connections
- IP Hash
- Generic Hash

NGINX PLUS는 위의 방법 이외에 두 가지를 더 지원

- Least Time
- Random

### NGINX 로드밸런싱 사용방법

Round Robin 이외의 로드 밸런싱을 사용할 경우,

`upstream` 블록 위에 **`hash`**, **`ip_hash`**, **`least_conn`**, **`least_time`**, **`random`** 지시어를 배치

1. Round Robin (Default)

가중치를 고려하여, 모든 서버에 균등하게 요청을 분배

```conf
upstream backend {
   # 지시어를 지정하지 않는다
   server backend1.example.com;
   server backend2.example.com;
}
```

2. Least Connections

활성화된 연결이 가장 적은 서버에 요청이 전송된다.

(가중치 또한 고려한다)

```conf
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

3. IP Hash (`세션 유지 가능*`)

요청이 전송되는 서버는 클라이언트의 IP에 따라 결정

IPv4 주소의 처음 3개의 옥텟 (예: `192.168.0`.x), 혹은 IPv6 주소가 해시 값을 계산하는데 사용

```conf
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

**일시적으로 서버를 제외해야할 경우 (점검등의 이유로)**

`down` 파라미터를 사용한다

이렇게 지정할 경우, IP 주소 매핑은 유지되며, 3번으로 가야했던 요청을 다음 서버로 이동하게 된다

```conf
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```

_※. ip_hash 주의사항_

프록시나 로드 밸런서 뒤에 있는 경우에는 실제 IP 인식이 어려워 다음의 지시어 설절을 써야한다.

```conf
# ip_hash가 실제 클라이언트의 IP를 기준으로 해시를 계산

http {
    real_ip_header X-Forwarded-For;
    set_real_ip_from 192.168.0.0/16; # 신뢰할 수 있는 프록시 IP 대역
}
```

4. Generic Hash (사용자 정의 키에 따라 결정)

Ip hash의 경우 ip를 기반으로 서버를 선택, generic hash는 사용자가 정의한 키(User-defined-key)를 기반으로 서버를 선택한다

```conf
upstream backend {
    # $request_uri --> 요청 URL을 키로 사용
    # ex) /index.html, /api/data 등

    # consistent --> 일관성 해시를 사용
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
}
```

캐시 서버나 상태를 축적해야하는 어플리케이션의 로드 밸런싱에 유용

## 3. HTTP 로드 밸런싱 옵션

1. weight (가중치)

Round Robin과 Least Connections 방법은 가중치를 사용하여 구성할 수 있다

다음과 같은 구성일 경우, 1번 서버에 5의 가중치 2번 서버에 1의 가중치, 백업 서버는 두 서버가 정상일 경우 사용하지 않는다

1번 서버에는 6개의 요청중 5개가, 2번 서버에는 6개의 요청중 1개가 전송되게 된다.

```conf
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```

2. slow start

최근 복구된 서버에 요청이 몰려 타임아웃이 발생하는 것을 방지

(NGINX PLUS 에서만 사용 가능)

3. session persistence (세션 지속)

NGINX Plus에서 사용가능, `NGINX의 경우 ip_hash와 hash를 사용하도록 한다`

4. limit the number of connections (연결 수 제한)

(NGINX PLUS 에서만 사용 가능)

참고 : [https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer
]

5. zone

각 worker 프로세느는 서버 그룹 설정을 따로 따로 가지며,

서버 연결 수나 실패 횟수 같은 카운터도 독립적

따라서, `서버 상태를 정확하게 판별하지 못하고`

`서버 그룹 설정을 동적으로 변경할 수 없음`

<br />

**zone 지시어를 사용할 경우**

모든 worker 프로세스가 공유 메모리 영역을 통해 같은 서버 그룹 설정과 카운터를 사용

그 결과, `서버 상태를 정확하게 판별`

`서버 그룹을 동적으로 변경할 수 있음`

active health check나 _Least Connections_ 방식의 로드 밸런싱이 정상적으로 작동

---

## 4. TCP and UDP 로드 밸런싱

TCP 로드 밸런싱은 주로, 안정적인 연결 유지를

UDP 로드 밸런싱은 빠른 반응과 낮은 지연을 목적으로 사용한다

단, 오픈소스 버전에서는 `--with-stream` 옵션을 사용해 빌드해야 사용이 가능하다.
