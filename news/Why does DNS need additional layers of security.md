
> Source : https://www.cloudflare.com/ko-kr/learning/dns/dns-over-tls/

## 개요

DNS는 인터넷의 전화번호부와 같다.  
DNS resolver는 사람이 읽기 편한 도메인 이름을 기계가 읽는 IP 주소로 변환해준다.  
기본 값으로, DNS 쿼리는 UDP를 통해 plaintext 포맷으로 전송되고 응답을 받게 되는데, 이는 ISP 또는 누구나 네트워크 모니터링을 통해 엿볼 수 있다는 것을 의미한다.  
웹사이트 자체는 HTTPS 를 사용하더라도 해당 사이트에 접근하기 위한 DNS 쿼리는 노출이 될 수 있다는 말이다.

이러한 프라이버시의 부재는 보안에 큰 취약점을 가져다준다.  
예를 들어 정부나 해커들은 사용자가 온라인에서 어떠한 행동을 하는지 쉽게 추적하고 감시할 수 있게된다.

![1](https://images.ctfassets.net/slt3lc6tev37/5Cgzkxb8COyIZ9evqqGFyF/384df7ee28643474080bbcd564fc3cfa/dns-traffic-unsecured.svg)

암호화되지 않은 DNS 쿼리를 보낸다는 것은 메일로 엽서를 보낸 것과 같다고 생각할 수 있다.  
메일을 처리하는 사람은 누구나 엽서 뒷면에 적힌 내용을 볼 수 있으므로 민감한 내용이 포함된 엽서를 우편으로 보내는 것은 현명하지 않다.  

DLS over TLS 및  DNS over HTTPS는 악의적인 사람(ex 광고주, ISP 등)이 데이터를 해석할 수 없도록 DNS 트래픽을 암호화하는 두 가지 표준이다.  
이는 엽서를 봉투에 봉인해서 보내는 것과 같아서 누군가 내용을 엿보는 것을 걱정하지 않고 엽서를 보낼 수 있다.

![2](https://images.ctfassets.net/slt3lc6tev37/7qcyOJwWyOt4EVJykiIRTn/30e34453409eb42fa1ec36680609ad8d/dns-traffic-over-tls-https.svg)

## What is DNS over TLS?

DNS over TLS(DoT)는 DNS 쿼리를 보호하기 위해 암호화하는 표준 방식이다.  
이는 HTTPS 가 통신을 인증하기 위해서 사용하는 TLS 프로토콜을 동일하게 사용한다.  
UDP를 통해 보내지는 통신 위에 한 단계 암호화 단계(TLS)를 추가한 것이며 이를 통해 DNS 쿼리와 응답을 안전하게 보호할 수 있다

## What is DNS over HTTPS?

DNS over HTTPS(DoH)는 DoT의 대안이다.  
DoH를 사용하면 UDP 프로토콜 대신 HTTP 또는 HTTP/2 프로토콜을 통해 전송되며 DoT와 마찬가지로 DNS 트래픽을 위조하거나 변경할 수 없도록 한다.  
DoH 트래픽은 네트워크 관리자의 관점에서 볼 때 다른 HTTPS 트래픽처럼 보인다.

2020년 2월, firefox 브라우저는 미국에서 DoH를 default로 사용하도록 적용했다.  
firefox에서의 DNS 트래픽은 DoH를 통해 암호화된다.  
다른 브라우저도 DoH를 지원하기는 하지만 아직까지는 default로 제공하고 있지는 않다.

## Wait, doesn't HTTPS use TLS for encryption too? How are DNS over TLS and DNS over HTTPS different?

각각의 표준 구격은 그에 맞는 RFC 문서가 존재한다.  
이 것도 차이점이라고 볼 수는 있겠지만 DoT와 DoH의 가장 큰 차이점은 그들이 사용하는 port가 다르다는 것이다.  
DoT는 오직 853 port를 사용하고, DoH는 443 port를 사용한다.  

DoT는 전용 port를 사용하기 때문에 DNS 트래픽 자체는 암호화되었지만, DNS 트래픽이 발생하고 있다는 것을 해당 port를 보고 파악할 수 있다.  
반면 DoH는 다른 HTTPS 통신들과 공유하는 port를 사용하기 때문에 일반적인 HTTPS 트래픽처럼 보이게 된다.

## What is a port?

port는 네트워크에서 다른 시스템의 기기와 연결할 수 있는 가장의 장소이다.  
네트워크로 연결 된 모든 컴퓨터에는 여러 포트가 존재하며, 특정 포트는 특정 유형의 통신을 위해 예약되어 있다.

## Which is better, DoT or DoH?

이 주제는 논쟁의 여지가 있다.  
네트워크 보안 관점에서 DoT가 틀림없이 더 좋다. 네트워크 관리자는 DNS쿼리를 모니터링하고 차단할 수 있으며 이는 악성 트래픽을 식별하고 차단하는데 중요하다  
하지만 DoH 쿼리는 일반 HTTPS 트래픽 속에 숨겨져 있으므로 다른 모든 HTTPS 트래픽도 차단하지 않고는 쉽게 차단할 수 없다

하지만, 프라비ㅓ시 관점에서는 DoH가 틀림없이 더 좋다. DoH를 사용하면 DNS쿼리는 다른 HTTPS 트래픽 속에 숨겨지기 때문이다.  
이러한 점은 네트워크 관리자의 가독성을 낮추게 만들지만 유저의 프라이버시는 높아지게 된다.

## What is the difference between DNS over TLS/HTTPS and DNSSEC?

DNSSEC는 DNS resolver와 통신하기 위해서 필요한 DNS root server와 다른 nameserver들이 실제로 맞는지 검증하는 확장 보안도구이다.  
이는 DNS cache poisoning을 방지하기 위해서 디자인되었다.