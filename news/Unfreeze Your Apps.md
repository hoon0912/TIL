
> Source : https://blog.jetbrains.com/idea/2021/10/unfreeze-your-apps/

## 개요

이 글에서는 프로젝트 구조 및 API에 대한 사전 지식없이 장애가 발생한 코드를 찾고 디버그하고 수정하는 방법을 알려준다

## The problem

만약 이 글을 읽으면서 따라해보고 싶으면 레포지토리를 클론해서 실험해봐라
https://github.com/flounder4130/debugger-example

만약 어떤 행동을 해서 멈춰있는 복잡한 애플리케이션이 있다고 가정해보자  
이러한 버그를 재현해서 원인을 파악해야겠지만 해당 행동과 관련된 코드가 어느 부분인지 아는 것은 어려운 상황이다  

![1](https://blog.jetbrains.com/wp-content/uploads/2021/09/sophisticated-app-1.png)

예시로 든 애플리케이션으로 설명하자면 버튼 N을 눌렀을 때 프로그램이 멈춘다.  
하지만, 버튼 N 클릭과 관련된 코드를 찾는 것은 쉽지 않다   

![2](https://blog.jetbrains.com/wp-content/uploads/2021/09/find-in-path.png)

인텔리제이의 디버거를 통해서 어떻게 할 수 있는지 살펴보자

## Method breakpoints

method breakpoints의 장점은 전체 계층적인 클래스에 모두 적용이 가능하다는 것이다.  
이러한 기능이 우리의 상황에 어떻게 도움이 될까?  

위에 링크를 달아둔 예시를 보면, 모든 액션 클래스들은 perform 메소드를 가지고 있는 Action 클래스를 상속하고 있는 것을 확인할 수 있다.  
![3](https://blog.jetbrains.com/wp-content/uploads/2021/09/method-breakpoint-0.png)

인터페이스에 해당 코드 라인에 break point를 검으로 써 해당 메소드를 override해서 사용하는 모든 메소드가 실행될 때  
break point가 걸려서 멈추게된다.  

실제로 브레이크 포인트를 걸고 버튼 N을 클릭하면 Action 인터페이스를 실행하는 ActionImpl14에서 브레이크 포인트가 걸리는 것을 확인할 수 있다  
![4](https://blog.jetbrains.com/wp-content/uploads/2021/09/method-breakpoint-suspend.png)

## Pause application

위에서 언급한 방식은 잘 동작하지만 parent interface가 무엇인지 알고있다는 가정 하에 가능한 방법이다  
만약 우리가 그러한 사항을 모른다면 어떻게 대처해야될까?  

동일한 방식을 브레이크 포인트 없이도 적용할 수 있다.  
버튼 N을 눌러서 애플리케이션이 멈추었을 때, 인텔리제이 메뉴에서 Run > Debugging Actions > Pause  
로 들어가서 현재 프로그램이 코드상의 어떤 부분에서 멈추었는지 확인할 수 있다  

![5](https://blog.jetbrains.com/wp-content/uploads/2021/09/pause-app-0.png)

Debugger 탭에서 현재 쓰레드의 상태도 확인 가능하며 애플리케이션이 특정 순간에 어떻게 돌아가고 있는지를 이해하기도 쉽다  
애플리케이션이 멈춰있기 때문에 어떤 메소드에서 멈추었는지 call site를 추적할 수 있다  

## Thread dumps

마지막으로 쓰레드 덤프를 뜰 수도 있다.  
버튼 N을 눌러서 애플리케이션이 멈추었을 때, 인텔리제이의 메뉴로 가서 Run > Debugging Actions > Get Thread Dump 를 통해 가능하다  

왼쪽 메뉴에서 스캔 가능한 쓰레드들을 선택할 수 있고,  
AWT-EventQueue에서 문제를 발생시키는 원인을 확인할 수 있다  

쓰레드 덤프의 단점으로는 특정 순간의 프로그램에 대한 스냅샷만 제공해준다는 것이다  
쓰레드 덤프를 통해서 variable을 탐색한다거나 프로그램 실행을 컨트롤할 수는 없다  

## HotSwap

버그를 발견했으니 수정해보죠  

프로그램을 멈추고 다시 컴파일하고 재실행을 할 수도 있다  
하지만 적은 부분을 수정하고 다시 배포하기 위해서 전체 프로그램을 재컴파일해야하는 것은 분명 큰 비용이다  

더 스마트한 방법을 사용해보자  
![6](https://blog.jetbrains.com/wp-content/uploads/2021/09/correct-the-code.png)

Run > Debugging Actions > Reload Changed Classes 를 통해서 새로운 코드가 VM에 적용되도록 할 수 있다  

![7](https://blog.jetbrains.com/wp-content/uploads/2021/09/hotswap-balloon.png)

하지만 HotSwap에는 여러 제한들이 있는 기능이기 DCEVM, JRebel 같은 도구를 사용해도 좋다

