
> Source : https://ai.googleblog.com/2021/09/announcing-wit-wikipedia-based-image.html?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%3A+blogspot%2FgJZg+%28Google+AI+Blog%29


## 개요

이미지와 텍스트간의 관계를 모델링하기 위해서 Multimodal visio-linguistic 모델은 풍부한 데이터셋이 필요하다.  
이러한 데이터셋을 얻기 위해서 전통적으로 수행되는 방식으로는 이미지 캡쳐, 웹 크롤링, 제목에서 [alt-text](https://www.w3schools.com/tags/att_img_alt.asp) 추출 등이 있었다.  
수동적으로 이미지에 주석을 남기는 행위는 더 높은 품질의 데이터를 생성해주는 경향이 있지만 생성할 수 있는 데이터의 양을 제한한다.  
반면에, 자동화된 추출 방식은 더 큰 데이터셋을 생성할 수 있지만 데이터 품질을 보장하기 위한 heuristics과 careful filtering이 요구된다.  
또한, 기존 데이터셋은 영어가 아닌 언어의 적용 범위가 부족하다는 단점도 있었다.  

이러한 상황은 자연스럽게 우리에게 이러한 질문을 하도록 만든다.  
> 이러한 한계를 극복하고 다양한 콘텐츠를 포함하는 고품질의 대용량 다국어 데이터셋을 만들 수 있을까?


오늘 우리는 WIT(Wikipedia-Based Image Text) 데이터셋을 소개한다.  
위키피디아 기사 및 이미지 링크에서 이미지와 관련된 텍스트를 추출하여 생성된 대규모 multimodal 데이터셋이다.  
이 데이터셋은 고품질 image-text만 유지하기위해 엄격한 필터링이 수반되었다.  
더 자세한 정보는 SIGIR'21에서 발표 된 자료 에서 확인할 수 있다.
* https://dl.acm.org/doi/10.1145/3404835.3463257

  
그 결과, 108개 언어에서 1,150만 개의 고유한 이미지가 포함된 3,750만 개의 image-text 데이터세트가 만들어졌다.  
WIT 데이터셋은 Creative Commons license하에 다운로드 및 사용이 가능하다.  
또한, Kaggle에서 WIT 테이터셋으로 대회를 개최할 예정이다.  

WIT 데이터셋의 장점은 다음과 같다
1. Size : WIT는 공개적으로 사용 가능한 가장 큰 mutimodal image-text 데이터셋이다.
2. Mulitilingual : 108개 언어로 다른 데이터셋보다 10배 이상의 언어를 가지고 있다
3. Contextual information : 이미지당 하나의 캡션만 있는 일반적인 multimodal 데이터셋과 다르게, WIT는 페이지 및 섹션 수준의 텍스트 정보가 포함된다.
4. Real world entities : 방대한 지식 정보를 가지고 있는 위키피디아를 기반으로 하므로 WIT의 entity는 현실을 잘 표현한다.
5. Challenging test set : EMNLP에서 승인된 작업에서 모든 최신 모델은 기존 평가 세트보다 WIT를 사용한 경우 더 높은 성능을 보여주었다.
  
## 데이터셋 생성

WIT의 주요 목표는 품질이나 적용 범위를 희생하지 않고 대규모 데이터셋을 만드는 것이다.  
그래서 우리는 가장 큰 온라인 백과사전인 위키피디아를 활용하여 시작했다.  
예를 들면, [Half Dome (Yosemite National Park, CA)](https://en.wikipedia.org/wiki/Half_Dome) 페이지를 기준으로 살펴보자  
해당 문서는 이미지에 대한 다양한 text captions, contextual information을 가지고 있다. 또한 그 외에 title,description, 그 외 metadata 들이있다.  

우리는 이미지가 있는 위키피디아 페이지를 선택한 다음 다양한 image-text 정보 및 주변 context를 추출하였다.  
또한 데이터 품질을 보장하기 위해 엄격한 필터링 프로세스를 수행하였다.(자세한 필터 과정은 source 참조)  
우리는 image-caption 데이터셋에서 샘플링을 통해 그 품질을 직접 평가하였다. 그리고, 샘플의 98%가 좋은 데이터셋이라고 평가되었다.  

## 매우 다국적인 특징

108개 언어로 된 데이터가 있는 WIT는 최초의 대규모 다국어 multimodal 데이터셋이다.  

## 최초의 상황별 image-text 데이터셋

대부분의 multimodal 데이터셋은 주어진 이미지에 대한 단일 text caption만 주어진다(또는 유사한 caption의 여러 버전)  
WIT는 context 정보를 제공하는 최초의 데이터셋이다. 연구자가 이미지를 선택할 때 context의 영향도를 확실할 수 있는 모델을 만들 수 있게한다.  

특히, 연구에 유용할 수 있는 WIT의 주요 textual 필드는 다음과 같다
* Text captions: WIT는 세 가지 종류의 이미지 caption을 제공한다. 각각 "참조 설명", "속성 설명", "대체 텍스트 설명"이 이에 해당된다.
* Contextual information :  WIT는 페이지 title, description, URL, local context 등의 정보를 가지고 있다.

## 매우 고품질의 학습용 데이터셋 그리고 평가 벤치마크를 위한 도전

위키피디아의 다양한 개념에 대한 광범위한 적용은 WIT 데이터셋이 도전적인 벤치마크 역할이 된다는 것을 의미한다.  
기존 데이터셋을 이용한 텍스트 검색 모델에서는 80대의 recall 결과가 나오는 반면 WIT 데이터셋을 사용한 경우  
자원이 풍부한 언어의 경우 40대, 풍부하지 않은 경우 30대의 reacall 결과가 나왔다.  
우리는 연구자들이 더 강력한 모델을 구축하는데 WIT 데이터셋이 도움이 되기를 바란다  

## 결론
우리는 WIT 데이터셋이 연구자들이 더 나은 multimodal 다국어 모델을 구축하는데 도움이 될 것이며 궁극적으로 시각 언어 데이터보다 실제 작업에서 개선된 기계 학습 모델로 이어질 것이라고 믿는다



