
> Source : https://blog.cloudera.com/an-introduction-to-ranger-rms/

## 개요

CDP(Cloudera Data Platform)는 Apache Ranger를 통해서 table,columns,files,directories에 대한 접근을 컨트롤 하는 기능을 제공하였다.  
동일한 데이터를 다른 workload를 통해 접근하는 경우는 굉장히 흔한데, 예를 들면 Apache Hive query를 통해 요청이 올 때는 table level에서 권한이 요구되며  
Apache Spark 잡을 통해 파일에 대한 요청이 올 때는 다른 권한이 요구된다.  
불행하게도 이러한 상황에서 Hive와 HDFS에 대한 Ranger policy를 별도로 둘 다 생성하고 유지해야한다. (두 개의 데이터가 실제로는 동일함에도 불구하고)

때문에 Hive table에 대한 policy 변화가 발생했을 때, admin 유저는 해당 table에 대응하는 HDFS policy도 수동으로 같이 변경해줘야 할 필요가 있다.  
이러한 작업을 하지 않으면 데이터 접근이 없는 유저에게 데이터가 노출이 되는 등의 보안 이슈가 있기 때문이다.  
이러한 구조를 이상적으로 그려보면 table policy를 admin 유저가 변경했을 때 해당 table와 대응되는 HDFS policy가 자동으로 sync가 맞춰지는 구조를 떠올릴 수 있다.

바로 이러한 이상적인 구조가 Ranger RMS(Resource Mapping Service)를 통해서 제공된다.  
RMS는 CDP Private Cloude Base 7.1.4에 프리뷰 방식으로 포함되었다.

## What is Ranger RMS?

Ranger RMS를 간단하게 요약하면 Hive에서 HDFS로의 접근 권한을 자동으로 변환해주는 기능을 제공한다.  
즉, 어떤 유저가 A라는 테이블에 대한 접근 권한을 가지고 있으면, A 테이블을 구성하는 실제 HDFS 파일 접근 권한을 자동으로 받는다.  
그러므로, Ranger RMS는 Hive 테이블에 정의된 policy를 통해서 HDFS 파일과 디렉토리에 대한 접근 허용을 가능하게 해준다.  
Ranger RMS를 통해 materialized 된 모든 액세스 권한은 Ranger audit 로그에 모두 표시된다.

## How does it help?

이러한 기능은 Spark와 같이 Hive workload가 아닌 작업에서 external table을 사용하는 경우 매우 유용할 수 있다.  
유저가 Cloudera 제품에 대한 서로 다른 버전을 가지고 있다고 가정했을 때 이러한 기능이 얼마나 도움이 되는지 보자  
* CDH
    * CDH stack에서는 Apache Sentry가 Hive/Impala 테이블에 대한 권한을 관리한다.
    * Sentry는 HDFS ACL Sync라는 기능을 제공하는데 이는 Ranger RMS와 흡사하다.
    * Sentry는 Hive 테이블의 HDFS 파일에 대한 사용자 접근을 HDFS ACL를 사용해서 처리한다.
    * Ranger RMS의 Hive에서 HDFS로의 접근 권한 자동 변형기능과는 전혀 다르게 HDFS ACL Sync가 동작하지만, 기본 컨셉과 최종 결과는 테이블 수준에서 동일하다.
* HDP
    * HDP stack에서는 Hive 테이블의 HDFS 파일에 대한 접근이 필요한 경우 storage access policy를 직접 생성해줘야 한다
    * HDFS policy를 생성하는 작업은 Ranger를 통해서 하거나 POSIX permission 또는 HDFS ACLs를 통해서 할 수 있다
    * 그리고 두 가지 서로 다른 Hive 및 HDFS에 대한 policy에 대한 관리도 모두 수동으로 동기화되어야 한다
* CDP(RMS가 추가되기 이전 버전)
    * HDP와 마찬가지로 Hive 테이블의 HDFS에 대한 policy를 생성하는 작업은 Ranger,POSIX permission,HDFS ACLs를 통해서 처리된다.
    * 그리고 마찬가지로 서로 다른 두 Hive 및 HDFS에 대한 policy는 수동으로 생성되고 싱크가 맞춰줘야 한다

CDP Private Cloud Base 7.1.4 버전에서 Ranger RMS가 소개되면서 이제 CDH의 Sentry HDFS ACL sync와 동일한 기능을 제공할 수 있게 되었다.  
CDH에서 CDP로 이관하는 유저들은 이러한 기능이 CDP에서도 제공하니까 걱정하지 않다도 되고  
HDP에서 CDP로 이관하는 유저들은 이제 Hive table의 HDFS 파일에 대한 별도 policy들을 관리할 필요가 업게 된다  
즉, 기존에 HDFS파일에 대한 Ranger Policy, Hadoop ACL, POSIX permission 등을 구성하는 작업이 더 이상 필요없게 되었다는 의미이다.  
이러한 기능은 admin 유저가 수동으로 관리할 필요가 없으므로 실수의 가능성을 줄인다

## What does it do?

앞서 설명했다 시피, Ranger RMS는 내부적으로 Hive policy를 HDFS access rule로 변환한다.  
그리고 변환된 그러한 rule을 HDFS NameNode가 시행하도록 허용한다.  
이러한 자동 변환 기능은 직접적이고 간단하게 보이는데 내부적으로는 특정 사용자에게 HDFS 파일 엑세스 권한을 부여하기 전에 여러 요소들을 고려한다.  
Ranger RMS는 Ranger 내부에 HDFS policy를 직접 생성하지도 않고 사용자에 대한 HDFS ACL을 변경하지도 않는다.(Apache Sentry가 hdfs dfs -ls 명령을 수행하는 것처럼...?)  
또한 Ranger RMS는 mapping 정보를 만들어서 HDFS NameNode의 Ranger Plugin이 런타임 도중에 Hadoop SQL 권한으로 특정 접근에 대한 허용 여부를 판단하도록 해준다.  

아래 그림은 NameNode, Ranger Admin, Hive Metastore, Ranger RMS간의 상호작용을 설명한다.  
![1](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2021/10/RMS-Architecture-1.png)

Ranger RMS가 실행될 때마다, Ranger RMS는 HMS(Hive Metastore)에 연결한다.  
그리고 Hive resource랑 그에 대한 HDFS 파일 위치 맵핑 정보를 파일로 만들며 sync를 유지한다.  
이러한 맵핑 파일은 RMS 내부에 cache 형태로 저장이 되며, Ranger 서비스에서 백엔드 데이터베이스로 사용하는 테이블에도 저장된다. (위 그림에서 1,2,3번 참조)  
Ranger RMS가 실행되고난 이후 ranger-rms.polling.notifications.frequence.ms 옵션에 설정한 시간마다 주기적으로 HMS에 맵핑 정보를 query해서 데이터를 갱신한다. 기본 값은 30초이다  
만약 특정 Hive table에 대한 맵핑 정보 쿼리 시 일치하는 값이 HMS에 없거나(Hive table이 삭제된 경우인 듯)  
특정 Hive table policy가 Ranger DB에서 삭제된 경우 (admin 유저가 기존에 있던 policy를 삭제한 경우인 듯) HMS에 저장 된 모든 맵핑 정보를 full sync 한다 (아마도 30초마다 실시하는 sync 작업은 policy 설정을 한 Hive table만 대상으로 query를 날리는 듯함. 다만, full sync가 필요한 순간들이 있다는 얘기를 하는 듯)  
예를 들면, 특정 configuration이 변경되면 백엔드 데이터베이스에서 Ranger RMS 맵핑 데이터를 지워야 할 수 있다.  
이러한 작업은 sync checkpoint를 지우고 다음에 RMS가 다시 재시작할 때 전체 재동기화를 발생시킨다.  
동기화 프로세스는 HMS에 쿼리하여 HDFS의 파일 경로에 매핑할 데이터베이스 이름 및 테이블 이름과 같은 Hive Object 메타데이터를 수집한다.  

HDFS 관점에서 바라보면 NameNode에 실행 중인 Ranger HDFS 플러그인에 추가적으로 HivePolicyEnforcer 모듈이 추가되었다.  
NameNode가 Ranger Admin으로부터 HDFS policy를 다운로드 받는 것 뿐만 아니라, 이제는 백엔드 데이터베이스에 저장 된 맵핑 정보를 토대로 Hive policy도 다운로드가 가능해졌다.  
그리고 HDFS 엑세스는 HDFS, Hive policy를 모두 고려해서 결정한다.

어떤 HDFS 파일에 대한 접근 요구사항을 판단할 때 우선 Ranger의 HDFS policy를 먼저 고려한다. (위 그림에서 4,5번에 해당하는 듯)  
그리고 해당 HDFS 파일이 RMS에서 제공하는 리소스 매핑 파일에 항목이 존재하는지 확인한다.  (위 그림에서 6번에 해당하는 듯)  
그런 다음 매핑 파일로부터 Hive 리소스를 계산한 다음 Ranger에 있는 Hive 저책을 적용한다 (위 그림에서 8,9번에 해당하는 듯)  
마지막으로 복잡한 설정 조건에 따라 최종적으로 HDFS 파일에 대한 접근 요구사항 가능 여부를 판단한다.

## Policy Evaluation Flow with Ranger RMS

아래 그림은 Ranger RMS가 포함되었을 때 policy 평가 과정을 표현한 그림이다.  
Ranger RMS를 사용하도록 설정한 경우에도 수동으로 생성된 HDFS 정책이 우선 순위를 가진다 (아래 그림에서 Managed Table에 대한 부분을 이야기하는 듯)  
![2](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2021/10/HDFS-Access-thru-RMS.png)

## What about Managed Tables, can it help?

CDP에서 Hive는 두 종류의 테이블을 가진다. 각각 Managed Table, External Table  
이에 대한 자세한 설명은 [여기](https://docs.cloudera.com/cdp-private-cloud-base/7.1.7/using-hiveql/topics/hive_hive_3_tables.html) 에서 확인 가능하다.  
Ranger RMS의 주요 사용 사례는 External table에 대한 policy 관리를 단순화하는 것이지만 Managed table에서도 유용한 경우가 종종 있다.  

Hive Warehouse Connector의 [Spark Direct Reader](https://docs.cloudera.com/cdp-private-cloud-base/7.1.7/integrating-hive-and-bi/topics/hive_spark_direct_reader.html) 모드는 파일 시스템에서 Hive 트랜잭션 테이블을 직접 읽을 수 있다.  
이러한 경우가 위에서 언급한 Managed table 대상으로 Ranger RMS가 유용한 케이스이다.  
이러한 spark 애플리케이션을 실행하는 사용자는 Hive 웨어하우스 location에 대한 읽기 및 실행 권한이 있어야 한다  
하지만 보안상의 이유로 RMS를 통해 Managed table에 대한 Hive 웨어하우스 location을 여는 것은 좋지 않다.  
그러므로 Managed Table에 대한 Ranger RMS 사용 여부는 사전에 미리 조사를 해보고 신중하게 결정하여라.  

이 밑으로는 Managed table에 대한 추가 설명이 있는데, 현재 보안이 중요한 하둡 클러스터만 관심 영역이므로 생략하겠음




