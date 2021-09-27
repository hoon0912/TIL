
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


