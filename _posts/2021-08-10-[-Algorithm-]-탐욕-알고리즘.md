---
categories:
  - Algorithm
tags:
  - algorithm
use_math: true
---

# 탐욕 알고리즘
___

## 1. 탐욕 알고리즘(greedy algorithm)

답을 하나씩 고르는데, 미리 정한 기준에 따라서 매번 ‘가장 좋아 보이는 답을 선택’

동적 계획과 마찬가지로 최적화 문제를 푸는데 주로 사용(상대적으로 설계하기 더 쉬움)한다.동적 계획은 재귀관계식을 세워 입력 사례를 더 작은 입력 사례로 분할하는 반면, 탐욕 알고리즘은 입력 사례를 분할하지 않고 순서대로 답을 하나 씩 모아 최종 답을 구축하는데, 가장 좋아 보이는 답을 선택하여 모은다. 즉, 어떤 선택이든지 선택할 당시(locally)는 최적이다.

<br>
**과정**
- 선택 과정: 집합에 추가할 다음 원소를 고른다. 그 당시 최적을 선택하는 탐욕 기준에 따라 선택.

- 적절성 검사:새로운 집합이 해답이 되기 적절한지 검사.

- 해답 점검: 새로운 집합이 문제의 해답인지 결정.


<br>
### 1) 최소 비용 신장 트리


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/612bc695-7683-4fd5-a4be-f3d8f5a7d1fd" alt width=400>
<em></em>
</center>


- (a)연결된, 가중치가 있는, 비방향 그래프 G

- (b) (V_4, V_5)를 이 부분 그래프에서 제거하더라도, 그래프는 연결된 채로 남아있다.

- (c) G의 신장 트리

- (d) G의 최소 비용 신장 트리

비방향 그래프에서 경로(path)는 이음선으로 연결된 마디의 나열(sequence)이다.이음선의 방향이 없기 때문에 마디 u에서 v로 가는 경로가 있다는 사실은 v에서 u로 가는 경로가 있다는 사실과 같다. 따라서,비방향그래프에서는 “두 마디 사이에 경로가 있다.”

비방향 그래프에서 모든 마디의 쌍에 대해서 경로가 있으면 “연결되어(connected)있다.”라고 부른다.비방향 그래프에서 마디에서 자신으로 다시 돌아가는 경로에 최소한 서로 다른 3개의 마디가 있으면 단순 순환경로(simple circle)라 하고,단순 순환 경로가 없는 비방향 그래프를 비순환적(acyclic)이라 한다.(c,d 비순환적)


<br>


트리(tree)는 비순환적이며 연결되어 있는 비방향 그래프이다.

뿌리 있는 트리(rooted tree)는 마디 하나를 뿌리로 지정한 트리로,가중치의 합이 최소인 부분그래프는 꼭 트리가 된다.
부분그래프가 트리가 아니면 순환경로가 존재하게 되고 순환경로 상의 이음선 하나를 제거하여 가중치의 합이 더 작은 연결된 그래프를 만들 수 있기 때문이다.


<br>


A의 부분 그래프 b는 최소 가중치를 가질 수 없다. 순환 경로$[V_3, V_4, V_5, V_3]$ 상의 어떤 이음선을 제거하더라도 그 부분 그래프는 연결된 채로 남아있다.예를 들어 V_4와 V_5를 연결하는 이음선을 제거할 수 있으며, 그 결과로 더 작은 가중치를 가진 연결된 그래프를 얻는다.


G에 대한 신장트리(spanning tree)는 G의 마디를 모두 포함하면서 트리인 연결된 그래프이다.(c와 d는 G에 대한 신장트리) 가중치가 최소가 되는 연결된 부분그래프는 신장트리가 틀림없지만 신장트리라고 해서 모두 가중치가 최소라는 보장은 없다.(트리c는 가중치가 최소가 아닌데, 왜냐하면 신장트리 d의 가중치가 더 작기 때문)
이러한 트리를 최소 비용 신장 트리(minimum spanning tree)라고 한다.

- D는 G에 대한 최소비용 신장트리이다.

- 하나의 그래프에 하나 이상의 최소 비용 신장 트리가 존재할 수 있다.

<br>


이 알고리즘은 “부분적으로 최적인 고려 사항에 따라 이음선을 선택한다”라고 할 수 있다.

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/bb776169-2ea1-4e42-adb9-ac98d0470fe1" alt width=800>
<em></em>
</center>


<br>
### 2) 프림 알고리즘

프림 알고리즘(Prim’s Algorithm)은 이음선의 F의 부분 집합인 공집합과 임의의 마디를 포함하도록 초기화된 마디의 부분 집합에서 시작한다.

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4977506b-868a-4224-a5ad-be7643adf451" alt width=600>
<em></em>
</center>


<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/117ee546-d2da-4bf0-b8b5-9f473ce0cc5f" alt width=700>
<em></em>
</center>

<br>

이를 코드로 작성 시,

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e7611279-b7ef-4154-a25d-6693826f28d5" alt width=700>
<em></em>
</center>

<br>

- 시작할 때 Y= {v_i}이므로, nearest[i]는 1로 초기화 하고 distance[i]는 v_1 과 v_i  를 연결하는 이음선의 가중치로 초기화

- 마디들을 Y에 추가하면, 이 두배열은 Y 바깥에 있는 각 마디에서 가장 가까운 Y안에 있는 새로운 마디를 참조하도록 갱신

-  어느 마디를 Y에 추가 시킬 지 결정하기 위해 매번 반복할 때마다 distance$[i]$ 값이 최소가 되는 인덱스를 계산. 이 인덱스를 unear라 부른다. unear가 인덱스인 마디는 distance$[unear]$를 -1로 놓음으로서 Y에 추가된다.



### 3) 크루스칼 알고리즘


```
F = ∅  // 이음선의 집합을 공집합으로 초기화
V의 서로소 부분집합 구축  (각 마디마다 하나씩, 그 마디만 포함) 
E에 속한 이음선을 가중치가 작은 것부터 차례로 정렬

while(답을 구하지 못함){

다음 이음선을 선택  // 선택절차

 if(이음선이 서로소 부분집합의 두 마디를 연결){    // 적절성 점검
 부분집합을 합침;
 그 이음선을 F에 추가;
 }

 if(모든 부분집합이 하나로 합쳐짐)    // 해답점검
 답을 구함;
}

```

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/02fe49ad-2211-4b8d-9696-87bbe8ad5d14" alt width=700>
<em></em>
</center>


<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/016e9adb-cfd3-47b2-ab9f-2c22c55e7089" alt width=700>
<em></em>
</center>


이를 코드로 정리 하면,

- initial(n) : n개의 서로소부분 집합을 초기화하는데, 각 서로소부분 집합은 1부터 n사이의 인덱스 중에서 정확하게 하나만 포함

- p = find(i) : p가 인덱스 i를 포함하고 있는 집합을 가리키게 한다.

- merge(p,q) : p와 q가 가리키고 있는 두 개의 집합을 하나로 합친다.

- equal(p,q) : p와 q가 같은 집합을 가리키면 true를 넘겨줌

```
void kruskal (int n, int m, set_of_edges E, set_of_edges& F){
index i, j;
set_pointer p, q;
edge e;

E에 속한 m개의 이음선을 가중치가 작은 것부터 차례로 정렬;
F=  ∅;

intial(n);  // n개의 서로소부분집합을 초기화

while(F에서 이음선의 수는 n-1보다 작다){
e = 아직 고려하지 않은 이음선 중 가중치가 최소인 이음선;
i, j = e로 연결된 마디의 인덱스;
p = find(i);
q = find(j);
if(! equal(p,q)){
    merge(p,q);
    e를 F에 추가;
   }
 }
}
```


ex) 예제

- Problem: 최소 비용 신장 트리를 구하시오.

- Inputs: 정수 n ≥ 2, 양의 정수 m, n개의 마디와 m개의 이음선을 가진 연결된 가중치 포함 비방향그래프, 이 그래프는 가중치와 함께 이음선이 포함된 집합 E로 표현

- Outputs: 최소비용 신장트리에서 이음선의 집합F


<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/237623d7-2eed-4c2f-a3a7-6ec46e35bcf0" alt width=700>
<em></em>
</center>


<br>
### 3) 스케쥴 짜기

- 여러 개의 작업을 진행해야 할 때 -> (작업 시간 + 대기 시간) 생성

- 스케줄 짜기(sheduling): 대기 시간을 최소화

- 시스템 내부 시간(time in the system): 작업 시간 + 대기 시간

- 최적 스케줄: 시스템 내부의 총 시간을 최소가 되는 스케줄

- 마감 시간이 정해진 경우(scheduling with deadlines): 각 작업을 끝내는데 같은 시간이 걸리지만 각 작업과 관련된 보상을 최대로 얻기 위해서 마감 시간을 고려해야 하는 경우


- 시스템 내부 시간의 최소화

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/b22da810-19e5-4699-9377-81320f360be5" alt width=700>
<em></em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7e549dd7-973c-4605-bf80-020a3cb98845" alt width=600>
<em></em>
</center>


직관적으로 봤을 때, 짧은 작업 시간 순으로 작업 수행을 했을 때 최적의 작업 시간을 갖는다.


이를 코드로 표현 시,

```
작업시간 별로 작업을 비내림차순으로 정렬

while(답을 구하지 못함){  // 선택절차와

    다음 작업을 스케줄에 넣는다;  // 적절성 점검

    if(작업이 더 이상 없다)

      답을 구했음;  // 해답점검
}

```



이렇게 알고리즘은 정렬 후, 값을 더하므로 실제 시간 복잡도는 


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8c7f3eb9-04e0-47f3-97e9-cfdb6e380eed" alt width=200>
<em></em>
</center>


<br>

더 나아가서 이 알고리즘을 복수 작업자 스케줄 짜기 문제(multiple -server scheduling problem)으로 일반화하기 쉽다.

1. 작업자가 m명 있다고 가정했을 때 작업자의 순서는 무작위다. 

2. 작업 시간별로 작업을 비 내림차순으로 정렬한다.

3. 첫째 작업자는 첫째 작업을 하고, 둘째 작업자는 둘째 작업을 하고, ... m번째 작업자는 m번째 작업을 한다.

4. m번째 작업자가 작업을 끝낸 후, 첫 번째 작업자는 m+1번째 작업을 한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7ab02e44-a4c6-471a-80c0-cef8efd0685b" alt width=700>
<em></em>
</center>



<br>
<br>


출처: 알고리즘 기초