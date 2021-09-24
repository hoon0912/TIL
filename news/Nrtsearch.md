> Source : https://engineeringblog.yelp.com/2021/09/nrtsearch-yelps-fast-scalable-and-cost-effective-search-engine.html

## 개요

Yelp 서비스에서 검색과 랭킹은 굉장히 중요한 기능이다.  
때문에 우리는 [Yelp's Elasticsearch-based ranking 플랫폼](https://www.youtube.com/watch?v=rv-SCLb2oOQ) 을 만들어서 real-time indexing, learning-to-rank 을 수행하였다.  
하지만, 우리는 Elasticsearch를 대체하고 루신 기반 검색 엔진(Nrtsearch)를 최근에 개발했다  

## 왜 ES를 교체하는가?

우리가 기존에 만들었던 ES 기반의 랭킹 플랫폼은 여러 케이스에서 잘 동작하고 있었지만, 더 많은 use-case에 대해서는 한계점을 가지고 있었다.  
ES가 그러한 상황에서 확장성을 제공하지 못한다고 판단하여 우리가 직접 개발하였다  
1. 문서 기반의 복제
* ES에서 하나의 문서는 replica 마다 개별적으로 인덱싱된다. 우리는 scape up이 아니라 scale out 방식으로 확장이 필요한데 replica를 scale out을 하기 위해서는 인덱싱을 위한 더 많은 CPU가 필요하게 되었다
2. 샤딩이 잘 안됨
* ES에 의해 여러 인덱스들의 샤드는 분산 저장된다. 그러나 우리는 특정 노드에서는 트래픽이 몰리고 특정 노드에서는 트래픽이 없는 hot/code 노드들을 마주할 수 있었다. 따라서 이러한 상황을 피하는 샤드 분배 기능이 필요하였다
3. 자동 확장이 어려움
* 검색 트래픽이 발생하고 있는 노드에서 새로운 노드로 샤드를 이관이 필요하다. 이러한 문제는 실시간 환경에서 확장하는 것을 어렵게 만들었다
따라서 우리는 항상 최대치의 용량으로 프로비저닝을 하였고 각 인덱스의 shard, replica의 배수 별로 확장을 고려해야 해서 작업을 복잡하게 만들었다
  
베를린 Buzzwords에서 Yelp의 랭킹 플랫폼 진화에 대해 발표했을 때, 아마존이 전자 상거래 검색에 루씬을 사용한 방법에 대해 설명했다.  
우리의 관심을 끈 두가지 루씬의 기능이 강연에서 언급되었다.  
1. NRT(준실시간) 세그먼트 복제
* 루씬은 인덱스된 데이터를 segment에 저장한다. 이는 immutable하게 이루어지며 덕분에 독립적으로 검색이 가능하도록 만들어준다.  
primary 노드는 모든 인덱싱 작업을 하고 replica 노드는 문서를 재색인하지 않고 단순히 segment를 pull해오면 된다
  
2. 동시 검색
* 루씬은 멀티코어 CPU의 이점을 활용하여 인덱스의 여러 세그먼트를 병렬 검색할 수 있다.
따라서 여러 shard에 대한 병렬화 대신에 루씬 세그먼트에 대해 병렬화 할 수 있다
  
ES는 장래에 이러한 기능을 지원할 계획이 없기 때문에 루씬에 대해 더 조사하기로 결정했다

## 디자인 목표

준실시간 세그먼트 복제, 동시 검색이 우리가 검색 엔진에 필요한 메인 기능이기도 했지만  
기존 플랫폼에 존재하던 기능도 유지할 필요가 있다. 게다가, auto scaling 을 지원하기 위한 몇 가지 기능이 필요하였다.  
아래 리스트는 우리가 새로운 플랫폼에 대해 설정한 설계 목표였다

1. 루씬 기반으로 구축이 필요하다. 준실시간 세그먼크 복제, 동시 검색을 제공하는 것 외에도 최소한의 변경으로 기존에 작성한 우리 자바 코드들(ML 기반 랭킹, 분석 등등)을 재사용할 수 있기 때문
2. 인덱싱 및 세그먼크 병햡으로 인해 검색 요청을 처리하는 노드의 오버헤드를 최소화 해야 함
3. 분석이 아니라 검색에 최적화 된 시스템이어야 함
4. primary 노드의 인덱스 복사본을 외부 저장장치(예: 아마존의 S3)에 저장하여 로컬 인스턴스 저장장치에 백업하지 않고도 노드를 중지 및 재시작할 수 있어야 함
5. node startup 단계에서 ES의 리벨런싱 작업 같은게 없도록 시작 시간이 빨라야 한다. 그래서 서비스의 load가 증가 되었을 때 빠르게 새로운 노드를 초기화 할 수 있도록 한다
6. 특정 버전에 의존성을 가지지 않는 확장 기능을 쉽게 배포할 수 있는 안정적인 API가 필요함 (예: custom 분석 및 ML 랭킹은 자바 모듈 내에서 구현될 것으로 예상)
7. 필요한 애플리케이션을 위한 시간 일관성?

우리는 NrtSearch를 구축하면서 몇 가지 다른 도전을 예상하였다.  
이러한 문제는 쿠버네티스를 사용하여 Nrtsearch의 외부에서 문제를 해결하도록 설계하였다  
예를 들면, crash 된 primary 노드를 교체하는 작업은 Nrtsearch에서 처리하지 않고 쿠버네티스에 의해 처리되도록 설계

1. Nrtsearch는 server-side 로드 밸런싱 기능을 구현하지 않을 것임. 각각의 replica들은 단순히 request가 들어오면 그에 따른 response만 리턴하도록 설계 되었음  
따라서 replica 들로 트래픽을 공평하게 분배하는 별도의 수단이 필요하였고 우리는 우리의 service mesh? 에 의존해서 로드 밸런싱을 계획 중임
   최종적으로 그리는 그림은 Nrtsearch 자바 클라이언트 라이브러리에 라운드 로빈 로드 밸런싱 기능을 추가하는 것임
   
2. 우리는 Nrtsearch에 트랜잭션 로그를 추가하지 않기로 결정하였음. 클라이언트가 최근 변경 사항을 디스크에서 유지하기 위해서는 주기적으로 commit을 호출하도록 요구할 것임
즉 primary 노드가 crash 되었을 때, 마지막 commit이후에 인덱스 된 문서들은 유실된다는 의미임  
   우리의 Flink 기반 인덱싱 시스템은 특정 기간 및 특정 문서 수마다 commit을 호출해서 체크포인트를 자동적으로 생성하고 있음
   만약 primary node가 crash 되거나 재시작된다면, 이전 체크포인트로 revert 되어서 해당 체크포인트 이후 인덱싱 요청을 다시 보낸다
   
## 구현

우리는 Mike McCandless의 오픈소스인 [Lucene Server](https://github.com/mikemccand/luceneserver) 를 기반으로 우리의 검색 엔진을 구현할 것을 결정했다.
해당 오픈소스는 루씩을 기반으로 구현되어있고, 준실시간 segement 복제, replica 검색, 문서 인덱싱 및 검색을 위한 API 등을 가지고 있기 때문에 선택했다.  
이 프로젝트는 루씬 6.x 기반으로 구현되어 있어서 우리는 가장 먼저 루씬 8 버전으로 업그레이드 하였다.  
그리고, REST/JSON 기반으로 작성 된 API를 gRPC/protobuf 로 변경해서 serialization/deserialization 성능을 높이고자 하였다.  
또한, 루씬이 API 구현자에게 결정을 맡기는 부분인 primary 노드와 replica 노드간의 segment 복제를 gRPC를 사용하였다  
gRPC는 마이크로 서비스간의 통신에 탁원하지만 여전히 일부 REST 클라이언트와 Swagger 기반 서비스가 있다.  
또한, gRPC는 사람이 서비스에 직접 쿼리하기 어려운 구조이기 때문에 이러한 문제를 처리하기 위해 grpc-gateway를 추가하여 REST API를 추가하였다.  
또한, 여러 segments들을 대상으로 병렬적으로 동시에 검색을 하기 위해서 신규 doc value를 추가하였다.  

우리는 초기에 잡아놓은 디자인을 조금 수정한 것이 있는데, primary 노드가 segment를 disk에 쓰는 것에서 Amazon S3와 같은 외부 저장장치에 쓰기로 결정한 사안을  
영구 storage volumn (ex Amazon EBS) 로 변경하기로 결정했다.  
그 결과, 만약 primary 노드가 재시작되거나 다른 인스턴스로 역할이 변경되는 경우, 우리는 disk에 index 데이터를 다운로드 할 필요 없이 단순히 EBS volumn에 attach 하기만 하면 된다.  
따라서 primary 노드의 시작 시간 및 indexing downtime을 줄이는 효과를 얻을 수 있었다.  
primary 노드에서 S3에 인덱스를 백업하기 위해서 backup endpoint를 가끔 호출하기는 한다.  
replica가 시작되면 S3에서 해당 인덱스의 가장 최근 백업본을 다운로드 한 다음 primary 노드부터 업데이트를 동기화한다.  
primary 노드가 문서를 인덱싱할 때, 모든 replica에 대해서 업데이트를 publish해준다. 따라서 최신 segment를 가져와 최신 상태를 유지할 수 있게 한다.  

그 외의 추가 구현 내용은 Source를 참조 할 것. 생략!

## 기존에 존재하던 Workflow 를 이전하는 작업

우리는 여러 검색 use-case에 해당하는 작업들을 몇 년에 걸쳐 Nrtsearch 로 이전해왔다.  
처음에는 상대적으로 낮은 트래픽에 적은 index를 요구하는 것부터 해서 점차적으로 높은 트래픽에 많은 index를 요구하는 use-case 까지 이르렀다.  
대부분의 검색 use-case들은 Apollo(ES Client를 위한 proxy)를 통해서 처리가 되는데, 우리는 단지 request가 Nrtsearch로 가도록 Apollo를 수정하기만 하면 됬다.  

우리는 예상치 못한 문제로 인한 중단을 방지하기 위해 단계적으로 롤아웃을 수행했다.  
ES와 Nrtsearch에 모두 트래픽을 보냈지만 ES의 결과만 반환하는 1% dark-launch로 시작했다.  
시스템 안전성을 모니터링 하면서 우리는 dark-launch의 비율을 100% 까지 끌어올렸다. 그리고, 새로운 시스템의 성능이 서비스 가능한 상태라는 것을 확인할 수 있었다.  
우리는 또한 ES와 Nrtsearch로부터의 결과를 비교하는 작업도 수행하였는데, 그 결과가 다르지 않다는 것을 확인한 이후 live-launch 를 시작하였다. 
그리고 다시 1%에서 100% 까지 단계적으로 진행하면서 모니터링 작업을 진행하였다.  

우리는 response의 차이를 확인하기 위한 evals를 사용하였다. evals는 두 개의 다른 endpoint로 요청을 보내서 응답의 차이점을 찾아 어떤 구성 요소가 최종 점수의 차이를 발생시켰는지 식별할 수 있는데 도움을 주었다.  
그 다음에는 얘네들이 자체적으로 만든 플로그인을 migration 하는 작업이 이루어졌는데 Nrtsearch 용 플러그인 인터페이스를 만들어서 잘 이관했다는 내용이 이어진다.  

## 승리!

우리는 30,50,99 퍼센타일에서 Nrtsearch로 이관한 이후 timing 수치가 30~50% 개선된 것을 확인할 수 있었다.  
Nrtsearch로 마이그레이션해서 인프라 비용도 절감했다는 점을 고려하면 이런 수치는 훨씬 더 인상적이다.  
이러한 결과는 우리 서비스의 변동하는 트래픽 패턴에 따른 autoscalling 기능과 spot instance를 shared pool에서 실행하는 것 때문이다.  
autoscaling 때문에 우리는 max 트래픽에 맞춰서 혹은 region failover 상황에 맞추어 인프라를 유지할 필요가 없게 되었다^^

이에 대한 차트는 Source에서 확인

