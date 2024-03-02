---
categories:
  - Reference
tags:
  - reference
  - paper
  - graphics
---
# k(+)-buffer
___

## 1. 개요

### 1) 제안

Fragment 깊이 정렬은 복잡한 렌더링 효과를 시뮬레이션 하는 다수의 이미지 기반 기술의 기본이나, 깊이 복잡도가 높은 장면을 래스터화 할 때는 시간과 공간의 관점에서 어려운 작업이다. 낮은 그래픽 메모리 요구 사항이 가장 중요할 때, k-버퍼는 생성된 모든 조각의 하위 집합에 대해 올바른 깊이 순서를 유리하게 보장할 수 있는 가장 선호 되는 framework로 간주되며, k+-buffer는 단일 렌더링 패스에서 k-buffer의 동작을 정확하게 시뮬레이션 하는 빠른 framework이다.

### 2) 성능

- 두 가지 메모리 제한 데이터 구조: 최대 배열 및 최대 힙은 GPU에서 개발되어 픽셀 동기화 및 fragment 컬링을 탐색하여 픽셀당 k-nearest fragment 유지

- 메모리 친화적 전략
	- 동적으로 개별 픽셀에 매모리를 할당하여 depth complexity가 낮으면 메모리를 아껴, 개별 픽셀의 낭비되는 메모리 할당 감소
	- 간단한 깊이 histogram을 통해 다양한 애플리케이션 목표 및 하드웨어 제한에 따라 할당된 k-버퍼 크기를 최소화
	- 고정 메모리 깊이 정렬 메커니즘으로 로컬 GPU 캐시를 관리

- 효율
	- 메모리 사용량, 성능 비용 및 이미지 품질 측면에서 이전의 모든 k-버퍼 변형에 비해 작업의 이점을 보여준다.


<br>

※Fragment: 픽셀을 하나하나 쪼갠 것
※depth complexity: 깊이 복잡도, 여러 개가 보이면 복잡하다고 할 수 있다.
※Histogram: 표로 되어 있는 도수 분포를 정보 그림으로 나타낸 것.



## 2. k+-buffer

- geometry sorting prior to rasterization, unbounded memory requirements, RMW memory-hazards, depth precision conversion artifacts and specialized hardware extensions (i.e. pixel synchronization, 64-bit atomic operations)등으로부터 자유롭다.

- 즉석에서 생성된 조각을 저장하고 정렬하는 대부분의 k-버퍼 대안과 달리, 처음에 k-nearest fragments가 정렬되지 않은 순서로 저장한 다음 깊이에 따라 재정렬하는 사후 정렬 단계를 따름.

- semaphore-based spin-lock mechanism은 공유 메모리에서 픽셀당 조각 작업의 원자성을 보장

- 먼 조각의 경합(busy-waiting)을 완화하기 위해 현재 유지되는 모든 조각에서 더 멀리 떨어져 있는 조각을 효율적으로 버리는 컬링 검사를 동시에 수행

- 가장 가까운 픽셀 별 프래그먼트를 정확하게 저장하기 위해 GPU에 두 개의 배열 기반 데이터 구조를 구축(최대 요소가 항상 첫 번째 항목에 저장되는 배열인 max-array, 각 내부 노드의 값이 해당 노드의 자식 값보다 크거나 같은 complete binary tree인 maxheap)

<br>
<br>

※바쁜 대기(영어: busy waiting 또는 spinning)란 어떠한 특정 공유 자원에 대하여 두 개 이상의 프로세스나 스레드가 그 이용 권한을 획득하고자 하는 동기화 상황에서 그 권한 획득을 위한 과정에서 일어나는 현상이다. 대부분의 경우에 스핀락(Spin-lock)과 이것을 동일하게 생각하지만, 엄밀히 말하자면 스핀락이 바쁜 대기 개념을 이용한 것이다.

※컴퓨터 과학 에서 읽기-수정-쓰기 은 원자 연산 의 클래스입니다 (예 : test-and-set , fetch-and-add 및 compare-and- swap )은 완전히 새로운 값이나 이전 값의 일부 기능을 사용하여 메모리 위치를 읽고 동시에 새 값을 기록합니다. 이러한 작업은 다중 스레드 응용 프로그램에서 경쟁 조건 을 방지합니다. 일반적으로 뮤텍스 또는 세마포어 를 구현하는 데 사용됩니다

※원자적 연산은 더 이상 쪼갤 수 없는 연산이라는 뜻이다. 따라서 여러 스레드(고루틴), CPU코어에서 같은 변수(메모리)를 수정할 때 서로 영향을 받지 않고 안전하게 연산할 수 있다. 보통 원자적 연산은 CPU의 명령어를 직접 사용하여 구현되어 있다.



### 1) Spin-Lock (SL)


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/eb892022-ad94-429e-b422-a387cff6b4e1" alt width=500>
<em></em>
</center>


<br>

- 내부 픽셀 형식이 R 32UI인 32비트 부호 없는 정수 텍스처가 픽셀 당 세마포어를 나타내기 위해 할당, 먼저 full-screen quad 렌더링(클리어 패스)을 실행하여 텍스처를 0으로 초기화

- OpenGLs imageAtomicExchange(texture lock, ivec2 P, uint V) function을 사용, V값을 인수로 원자적으로 교체하여 좌표 P에서 텍셀의 원래 값을 반환

- Pixel Synchronization (PS)는 조각 충돌을 방지하고 RMW 메모리 작업이 제출 순서대로 수행되도록 보장

- 최소한의 구현 수정으로 제안된 파이프라인을 리모델링하지 않고 PS/FSI를 사용하여 향상 가능.

- 픽셀 별 세마포어 사용을 피하면 메모리 요구량 감소


<br>
<br>


※세마포어 S는 정수값을 가지는 변수이며, 다음과 같이 P와 V라는 명령에 의해서만 접근할 수 있다. (P와 V는 각각 try와 increment)

※P는 임계 구역에 들어가기 전에 수행되고, V는 임계 구역에서 나올 때 수행된다. 이때 변수 값을 수정하는 연산은 모두 원자성을 만족해야 한다. 다시 말해, 한 프로세스(또는 스레드)에서 세마포어 값을 변경하는 동안 다른 프로세스가 동시에 이 값을 변경해서는 안 된다.

※스핀락(spinlock)은 임계 구역(critical section)에 진입이 불가능할 때 진입이 가능할 때까지 루프를 돌면서 재시도하는 방식으로 구현된 락을 가리킨다. 스핀락이라는 이름은 락을 획득할 때까지 해당 스레드가 빙빙 돌고 있다(spinning)는 것을 의미한다. 스핀락은 바쁜 대기의 한 종류이다.


<br>


### 2) Fragment Storing


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/fd294adb-9691-46ca-b2fd-51580d4f1587" alt width="100%">
<em>k = 8인 두 데이터 구조가 비순차적 조각 삽입의 수로부터 구성되고 업데이트 되는 방법을 보여줍니다. 버퍼가 완전히 가득 차면 두 개의 들어오는 조각이 성공적으로 삭제, max-array와 maxheap 노드 포인터 간의 내부 데이터 표현 비교 표시</em>
</center>


<br>


- geometry rendering (store pass)은 RG 32F(R은 색상, G는 깊이) 및 k 길이의 내부 형식을 사용하여 초기화 되어, 64-bit floating point 3D array buffer에서 픽셀당 가장 가까운 조각 데이터를 포착하기 위해 수행

- 가장 가까운 k에 속하지 않는 n개의 생성된 프래그먼트의 spinning을 완화하기 위해 고속 컬링 메커니즘이 수행, 이는 fragment를 받을 때 현재 유지되고 있는 fragment와 비교하여 depth value 값이 같거나 더 큰 값을 효율적으로 버리는 것으로 k-nearest fragment가 항상 살아남도록 보장

- 들어오는 모든 프래그먼트에 대해 전체 픽셀 행을 순회하지 않고 프래그먼트 컬링을 달성하기 위해 GPU에서 첫 번째 배열 위치에 최대 요소를 저장하는 두 가지 배열 기반 데이터 구조를 개발 (max-array (K+B-Array) and maxheap (K+B-Heap)).

- Max-array는 깊이 값이 가장 큰 프래그먼트를 항상 첫 번째 위치에 저장하고 나머지는 무작위로 배치하는 배열로 간주

- 들어오는 조각이 세마포어를 얻으면 첫 번째 빈 항목(O(1))에 정보를 저장. 이 경우 픽셀당 카운터가 인덱스로 사용되며 성공적인 삽입 후 증가

- per-pixel counter는 클리어 렌더링 패스 동안 0으로 초기화되고, 이 배열이 가득 차면, 가장 큰 깊이 값이 가장 큰 조각을 대신합니다.

- 완전히 채워진 배열에 삽입한 후 최대 배열을 일관성 있게 유지하기 위해 가장 큰 깊이 값(O(k))을 가진 조각을 찾아 새로 추가된 조각으로 바꿉니다(후자가 가장 큰 조각인 경우 제외).

- 이 프로세스는 fragment atomicity이 보장되기 때문에 값비싼 원자 연산을 사용하지 않고 구현

- fragment가 큰 경우에 매우 효율적이나, 힙이 완전히 가득 찬 후에는 비용이 많이 드는 작업을 수행해야 하므로, 힙 구조에 적용하지 않는 것이 좋다



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/496c6334-accd-4b67-a464-8250a354b120" alt width=500>
<em>프래그먼트 컬링 조건</em>
</center>


<br>



- 최종 이미지를 생성하기 전에 각 픽셀의 조각을 재정렬하기 위해 정렬 프로세스가 사용,정렬되지 않은 조각은 깊이 정렬을 수행하기 전에 처음에 픽셀당 k 크기의 로컬 배열에 복사, 정렬



<br>


### ※ l+-depth-sorting

 K+B-Array를 확장하여 l +- 깊이 정렬 제안

- 세마포어 필요 X

- capture를 반복하여 깊이 별 정렬을 진행한다. 캡처된 가장 먼 조각은 이전에 처리된 모든 조각을 버리기 위해 다음 반복에서 효율적으로 사용됩니다.

- 그러나 이는 z-fighting 문제를 처리할 수 없고, 전역 GPU에서 전체 조각 목록을 여러 번 읽어야 한다.





