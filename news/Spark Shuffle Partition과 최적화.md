
> Source : https://tech.kakao.com/2021/10/08/spark-shuffle-partition/

## 개요

카카오 기술 블로그에서 Spark partition에 대한 개념과 특성에 따른 최적화 방법을 실험을 통해 보여주고 있다  

## Spark Partition이란?

Partition은 스파크의 분산 단위이자 연산 단위이다  
Spark에서 하나의 최소 연산을 Task라고 표현하는데, 이 하나의 Task에서 하나의 Partition이 처리된다  
또한, 하나의 Task는 하나의 Core가 연산 처리한다.  

즉, 1 Core = 1 Task = 1 Partition

동일한 데이터셋을 input으로 사용한다면 결국 partition의 크기가  
각 task에게 할당되는 input의 크기가 된다.  
만약 partition의 개수가 적으면 각 partition의 크기는 커지게 되고  
partition의 개수가 크면 각 partition의 크기는 작아지게 된다  

즉, partition 개수가 적으면 그만큼 단일 partition의 크기가 커지게 되어 각 task에서 처리해야 하는 양이 늘어나므로 병렬성이 떨어져 처리 속도가 느려질 수 있다  
그리고 partition의 크기가 커지다보니 각 executor 에 할당된 task의 개수만큼 partition 크기가 곱해져서 execution memory 영역에 올라올 것이며 이는 곧 memory spill 이 발생할 확률이 높아진다  
반대로, partition 개수가 많으면 그만큼 단일 partition의 크기가 작아지게 되고 병렬성이 올라가서 처리 속도가 빨라질 수 있다  
하지만, 클러스터 전체 core수보다 더 많은 partition 개수가 생기게 되면 결국 클러스터가 처리할 수 있는 병렬성을 초과하게 되며 task 생성 비용이 더 커지는 오바헤드가 발생할 수도 있다  

따라서 partition의 개수는 executor의 memory와 직접적인 연관이 있으므로  
memory가 부족할 때 발생하는 현상(ex GC, shuffle spill)이 발생하면 partition 개수를 조절하는 것을 먼저 고려하기를 권장하고 있다  

## 최적화 실험

4번의 실험을 거쳐서 partition 수, 쿼리 최적화를 통해 얼마나 성능이 개선되는가를 보여주고 있다  
partition 수를 늘려서 기존에 단일 partition에 너무 많은 데이터가 할당되어 shuffle spill이 많이 발생하는 것을 줄일 수 있음을 증명하였다  
쿼리 최적화를 통해서 join -> groupBy 순서를 groupBy -> join의 순서로 최적화함으로 써 성능이 향상됨을 증명하였다  

즉, 2개의 최적화 요소를 이 실험을 통해 증명하고 있다

## 이해안가는 포인트

partition 수와 memory의 관계. 그리고 shuffle spill과의 관계는 이해하기가 수월했다  
하지만, join -> groupBy 보다 groupBy -> join이 더 성능이 좋은지는 이해가 안간다  
간단하게 시나리오를 그려봐서 테스트해보았을 때, groupBy를 먼저 하면은 각 key에 대한 locality가 보장된 상태가 되므로  
그 이후에 join을 하면 네트워크 I/O가 줄어드는 것이 확인되었다  
(그냥 처음부터 join을 해버리면 동일한 key가 여러 노드에 복사되어 전송된다)  
하지만, 메트릭 지표를 보면 shuffle read 양은 쿼리 최적화 전,후 모두 동일한 것으로 보아 네트워크 전송량 자체는 변하지 않았음을 의미하는 것 같다  
왜 groupBy를 먼저 실행하면 shuffle spill이 안 발생하게 되었는가? 이 부분이 아직 이해가 안가서 계속 고민 중이다