
> Source : https://blog.cloudflare.com/tackling-email-spoofing/

## 개요

오늘 우리는 이메일 스푸핑 및 피싱을 해결하고 이메일 전달성을 개선하기 위한 새로운 도구를 출시한다.  
새로운 Email Security DNS Wizard 를 사용하여 다른 사람이 당신의 도메인을 대신하여 악성 이메일을 보내는 것을 방지하는 DNS 레코드를 생성할 수 있다.  
이 새로운 기능은 또한 안전하지 않은 도메인의 DNS 구성에 대해 경고하며, 수정 방법에 대한 권장 사항 또한 보여준다.  
이 새로운 기능에 대해 알아보기 전에 한 발짜국 뒤로 물러나서 이메일 스푸핑 및 피싱에 대해 살펴보자

## What is email spoofing and phishing?

스푸핑은 불법적인 이득을 얻기위해 다른 사람으로 가장하는 과정이다.  
한 가지 예로는, 누군가가 mycoolwebpage.xyz와 같은 웹사이트를 호스팅하는 도메인과 매우 흡사한 mycoolwebpaqe.xyz 도메인을 사용한다고 가정하자.(page -> paqe)  
사용자들은 허위 웹사이트에 접속한 사실도 모른채 민감한 정보를 제공하기 쉽다. 브라우저에서 주소 표시줄만 보고 차이점을 찾기란 매우 어렵다.  

![1](https://blog.cloudflare.com/content/images/2021/09/Screen-Shot-2021-09-25-at-10.05.21-AM.png)

이메일 스푸핑에 대해 설명하기 위해서 Cloudflare 제품 업데이트 이메일을 살펴보겠다.  
헤더와 이메일 본문이 포함된 이메일의 전체 소스를 볼 수 있다.  

```aidl
Date: Thu, 23 Sep 2021 10:30:02 -0500 (CDT)
From: Cloudflare <product-updates@cloudflare.com>
Reply-To: product-updates@cloudflare.com
To: <my_personal_email_address>
```
위에서 볼 수 있다시피, 이메일을 받은 시간, 보낸 사람, 답장해야 할 사람, 내 개인 이메일주소가 헤더에 표시된다.  
From 헤더의 값은 보낸 사람을 보여주기 위해서 사용된다.  

![2](https://blog.cloudflare.com/content/images/2021/09/image3-30.png)

위와 같은 이메일을 받으면 사용자들은 Cloudflare에서 이메일이 보내진 것으로 가정한다.  
그러나 메일 서버에서 From 헤더를 수정해서 얼마든지 이메일을 보낼 수 있다.  
따라서, 이메일 서버가 뒷 부분에서 다룰 보안 작업을 적용하지 않은 경우, 사용자는 이 이메일이 Cloudflare에서 온 것으로 착각할 것이다.  

![3](https://blog.cloudflare.com/content/images/2021/09/image13-4.png)

이는 두 번째 공격 유형인 이메일 피싱으로 이어진다. 
악의적인 행위자가 회사 고객에게 회사 서비스 이메일 중 하나에서 보낸 것처럼 속이는데 성공했다고 가정해보자.  
이러한 이메일은 스타일과 형식이 합법적인 이멜이과 똑같아 보이기에 구별하기가 쉽지 않다.  
이메일 본문에는 하이퍼링크를 포함하여 일부 계정 정보를 긴급하게 업데이트 하라고 요청할 수도 있다  
또는 사용자가 링크를 클릭하여 악성 코드를 실행하거나 민감한 정보를 요청하는 스푸핑된 도메인으로 유도할 수도 있다.  

FBI의 2020년 인터넷 범죄 보고서에 따르면 인터넷 피싱은 2020년 가장 흔한 사이버 범죄였으며 240,000명 이상의 피해자가 5천만 달러 이상의 손실을 입었다고 집계된다.  
그리고 피해자의 수는 2019년 이후 2배 이상 증가했으며 2018년보다 거의 10배 이상 증가했다.  

![4](https://blog.cloudflare.com/content/images/2021/09/image5-23.png)

대부분의 피싱 공격이 수행되는 방식을 이해하기 위해 2020년 Verizon 데이터 침해 조사 보고서를 살펴보자  
피싱은 social attacks 중 80%를 차지하며, 소셜 활동의 96%는 이메일을 통해 전송되고 3%는 웹사이트를 통해, 1%는 전화 또는 문자를 통해 전송된다.  

![5](https://lh6.googleusercontent.com/zJaupW_6sFUgHngoCJ6naiTt_XpKHahn5P63J9jlxpynfThhNpDb9wAREZbAkL9SSIuUfu4l_K0rnYUP-iOQZvmZinl5Kt9BNKrreXFyQ07q0YZApEAdw927zjk7C5ohTdjBe9H2=s0)

이것은 이메일 피싱이 인터넷 사용자에게 큰 골칫거리가 되는 문제임을 분명히 보여준다.  
따라서 이메일 피싱에 도메인을 오용하지 못하도록 우리가 무엇을 할 수 있는지 알아보자  

## How can DNS help prevent this?

다행스럽게도 DNS(Domain Name System)에는 이미 3가지 스푸핑 방지 메커니즘이 내장되어 있다. 
* SPF(Sender Policy Framework)
* DKIM(DomainKeys Identified Mail)
* DMARC(Domain-based Message Authentication Reporting and Conformance)

그러나 이러한 것을 경험이 적은 사람들이 올바르게 구성하는 것은 쉽지 않다.  
구성을 너무 엄격하게 설정하면 합법적인 이메일이 삭제되거나 스팸으로 표시될 수 있다.  
구성을 너무 느슨하게 설정하면 도메인이 이메일 스푸핑에 오용될 수 있다.

## SPF (Sender Policy Framework)

SPF는 해당 도메인으로 이메일을 보낼 수 있는 IP 주소를 지정하는데 사용된다.  
SPF 정책은 도메인 TXT 레코드에 게시되므로 모든 사람들이 DNS를 통해 해당 도메인의 SPF 정보를 얻을 수 있다.  
coludflare.com 의 TXT 레코드를 살펴보자

```aidl
cloudflare.com 	TXT	"v=spf1 ip4:199.15.212.0/22 ip4:173.245.48.0/20 include:_spf.google.com include:spf1.mcsv.net include:spf.mandrillapp.com include:mail.zendesk.com include:stspg-customer.com include:_spf.salesforce.com -all"
```

SPF TXT 레코드는 항상 v=spf1로 시작한다. 일반적으로 ip4 또는 ip6 메커니즘을 사용하며 IP 주소 목록을 포함한다  
include 메커니즘은 다른 도메인의 SPF 레코드를 참조하는데 사용된다.  
이는 일반적으로 당사를 대신하여 이메일을 보내야하는 다른 제공업체에 의존하는 경우 수행된다.
위에 cloudflare.com의 SPF 를 예시로 보면 Zendesk, Mandrill 를 마케팅 및 거래 이메일로 사용하고 있는 것을 확인할 수 있다.

마지막으로, catch-all 메커니즘이 있다.  
catch-all 메커니즘은 +(Allow), ~(Softfail), -(Fail)을 우선적으로 등장한다.  
+(Allow) 방식을 용하면 기본적으로 모든 IP주소와 도메인이 지정한 도메인을 대신하여 이메일을 보낼 수 있도록 설정하므로 SPF 레코드를 쓸모 없게 만든다.  
~(Softmail) 방식은 서버에 따라 이메일을 스팸 또는 안전하지 않은 것으로 표시한다.  
-(Fail)은 서버에 지정되지 않은 소스에서 보낸 이메일을 수락하지 않도록한다.  

![6](https://blog.cloudflare.com/content/images/2021/09/image12-7.png)

위 그림은 앞서 설명한 메커니즘을 설명해준다.
구체적인 예시로 설명하면
1. hannes@mycoolwebpage.xyz 도메인으로 203.0.113.10 IP를 가진 서버로부터 이메일을 받았다고 가정하자
2. 이메일을 받고나서 receiver는 mycoolwebpage.xyz의 SPF 레코드를 찾는다
3. receiver는 SPF 레코드에 명시된 IP 리스트에 203.0.113.10 IP가 있는지 확인한다. 없다면 catch-all 메커니즘에 따라 행동한다.

## DKIM (DomainKeys Identified Mail)

SPF를 사용하여 혀용된 IP 주소만 특정 도메인으로 이메일을 보낼 수 있도록 했다.  
하지만 만약 IP가 변경되고 SPF 레코드가 업데이트 되지 않거나, 동일한 Google Email Server(클라우드)를 사용해서 동일한 IP에서 메일을 보내도록 하면 어떻게 될까?  
이러한 경우를 위해 DKIM이 필요하다.  
DKIM은 이메일의 일부(보통 본문 및 특정 헤더)를 암호화 하여 메일의 서명을 제공한다.  
이에 대해 자세히 알아보기 전에 cloudflare.com에 사용된 DKIM 레코드를 살펴보자

```aidl
google._domainkey.cloudflare.com.   TXT   "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDMxbNxA2V84XMpZgzMgHHey3TQFvHkwlPF2a11Ex6PGD71Sp8elVMMCdZhPYqDlzbehg9aWVwPz0+n3oRD73o+JXoSswgUXPV82O8s8dGny/BAJklo0+y+Bh2Op4YPGhClT6mRO2i5Qiqo4vPCuc6GB34Fyx7yhreDNKY9BNMUtQIDAQAB"
```

DKIM의 구조는 <selector>._domainkey.<domain> 이다. selector는 이메일 제공자를 의미한다.  
DKIM TXT 레코드는 항상 v=DKIM1으로 시작된다. 그 후에 public key type(k 태그), public key(p 태그)를 확인 할 수 있다  
아래는 DKIM이 어떤식으로 동작하는지 순서를 보여준다.
1. email server는 이메일 본문으로부터 hash 값을 생성한다
2. email server는 hash 값을 DKIM private key로 암호화한다.
3. 암호화 된 해시 값이 이메일에 포함 된 상태로 이메일이 전송된다.
4. email receiving server는 해당 도메인에 대한 DKIM TXT를 검색한다
5. DKIM public key를 사용해서 복호화한다.
6. email receiving server는 이메일 본문으로부터 hash 값을 생성한다.
7. 앞서 복호화 한 hash값이랑 직접 구한 hash값이 같은지 비교한다

## DMRAC(Domain-based Message Authentication Reporting and Conformance)
DMARC는 Reporting과 Conformance에 대한 메커니즘을 제공해준다.  
Reporting은 이메일 발신자가 얼마나 부적절한 이메일을 받았는지 알 수 있게 한다.  
Conformance는 이메일 수신자가 이메일을 처리하는 기준을 명확하게 제공해준다.  
이메일 수신자는 DMARC 레코드가 없어도 SPF 또는 DKIM 검사에 실패한 이메일에 자체 기준을 적용할 수 있다.  
그러나, DMARC 레코드에 구성된 정책(기준)은 이메일 발신자가 명시한 지침이므로 더 자신있게 처리할 수 있다.  

아래는 cloudflare.com의 DMARC 레코드이다
```aidl
_dmarc.cloudflare.com.   TXT   "v=DMARC1; p=reject; pct=100; rua=mailto:rua@cloudflare.com"
```

DMARC TXT 레코드의 구조는 _dmarc.<domain> 이다.  
그 후에 항상 v=DMARC1 이 따라온다. 그 이후에는 3개의 태그 정보가 있다.  
p태그는 만약 이메일 수신자가 SPF 또는 DKIM 체크에서 실패한 경우 어떻게 처리해야 할지를 명시한다.  
none,reject,quarantine 중에 값을 지정할 수 있으며 none 정책은 오로지 모니터링 용도로만 사용되며 체크 실패한 이메일을 정상으로 판단해버린다  
quarantine은 이메일 수신자가 SPF 또는 DKIM 체크에서 실패한 경우 스팸 폴더로 이메일을 보내버린다  
reject는 SPF 또는 DKIM 체크에서 실패한 경우 이메일을 버려버린다

퍼센테이지 태그(pct)는 설명 생략

마지막 태그인 rua는 리포팅 URI를 지정한다.  
SPF 또는 DKIM 체크에서 실패한 이메일들을 취합해서 그 레포트를 보내는 이메일 주소를 의미한다.  

![7](https://blog.cloudflare.com/content/images/2021/09/image10-5.png)

위 그림은 DMARC가 어떻게 동작하는지를 나타낸다.
1. From 헤더 값에 hannes@mycoolwebpage.xyz을 가지고 있는 이메일이 전송되었다
2. 이메일 수신자는 mycoolwebpage.xyz 도메인에 대한 SPF, DKIM, DMARC를 검색한다.  
3. 이메일 수신자는 SPF 와 DKIM 체크를 수행한다. 만약 둘 다 성공한 경우 해당 메일은 정상으로 분류된다. 만약 둘 중에 하나라도 실패한 경우 DMARC 정책이 정상 / 비정상 유무를 결정한다.
4. 마지막으로 이메일 수신자는 그 결과를 집계해서 레포트를 반환한다. 

## A few numbers on the current adoption

이제 우리는 SPF, DKIM, DMARC에 대해 배웠다.  
이 기술이 얼마나 지혜롭게 널리 사용되고 있는지 확인해보자

2020년 말까지 50% 미만의 도메인만 DMARC 레코드를 보유하고 있는 것을 확인 하였다  
DMARC 레코드가 없으면 SPF 및 DKIM 검사가 명확하게 시행되지 않는다는 것을 다시 떠올려보자  
또한 DMARC 레코드를 가지고 있는 도메인 중에서 65% 이상이 단순 모니터링 정책 옵션(p=none)을 사용하고 있음을 확인하였다  

![8](https://blog.cloudflare.com/content/images/2021/09/image2-37.png)

2021년 8월 1일 또 다른 보고서는 은행 부문에 속하는 도메인에 대한 통계를 제공한다.  
미국의 2,881개의 은행 중 44%만이 DMARC 레코드를 제공한다. DMARC 레코드가 있더라도 5개 중 2개는 p=none으로 설정했다.  
덴마크는 94%의 은행 도메인이 DMARC를 사용하고 있는 반면에 일본은 오직 13%만이 DMARC를 사용하고 있다.  
SPF이 적용 된 비율은 상당히 높은데 이는 SPF 표준이 2006년에 처음 도입되었고 2015년에 DMARC가 표준이 되었다는 사실과 관련이 있을 수 있다. 

## 생략

이 밑 부분은 SPF, DKIM, DMARC를 UI에서 쉽게 설정하는 자사의 새로운 서비스 소개임  
개인적으로 궁금하다면 Source 링크타고 확인할 것
