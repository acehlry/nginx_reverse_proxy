# Vitual Servers

각 트래픽 처리 컨텍스트에는 요청 처리를 제어하는 가상 서버를 정의하기 위한 하나 이상의 서버 블록이 포함된다.

`server`컨텐스트 내에 포함할 수 있는 지시문은 트래픽 유형에 따라 다르다.

<br />

**HTTP Traffic**

각 서버의 지시문은 특정 도메인이나 IP 주소의 리소스에 대한 요청 처리를 제어

<br />

서버 컨텍스트 내의 하나 이상의 위치 컨텍스트는 특정 URI 세트를 처리하는 방법을 정의

<br />

**Mail or TCP/UDP Traffic**

Mail이나 TCP/UDP 트래픽의 경우, 서버 지시문은 각각 특정 TCP 포트 혹은 UNIX 소켓에 도착하는 트래픽 처리를 제어

---

## Example Configuration

<br />

```conf
user nobody; # a directive in the 'main' context

events {
    # configuration of connection processing
}

http {
    # Configuration specific to HTTP and affecting all virtual servers

    server {
        # configuration of HTTP virtual server 1
        location /one {
            # configuration for processing URIs starting with '/one'
        }
        location /two {
            # configuration for processing URIs starting with '/two'
        }
    }

    server {
        # configuration of HTTP virtual server 2
    }
}

stream {
    # Configuration specific to TCP/UDP and affecting all virtual servers
    server {
        # configuration of TCP virtual server 1
    }
}
```

## 상속(Inheritance)

일반적으로, 자식 컨텍스트는 부모 컨텍스트에 포함된 지시문의 설정을 상속한다.

ex) proxy_set_header
