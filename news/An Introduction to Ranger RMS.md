
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
    * HDP stack에서는 
