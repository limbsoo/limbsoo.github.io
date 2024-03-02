---
categories:
  - Algorithm
tags:
  - algorithm
use_math: true
---
# 동적 계획
___

## 1. 동적 계획

### 1) 하향식 분석

: top - down으로, 출력 형태를 만들어 놓고 회수하는 형태이며 결과가 이전의 결과에 영향을 받는다.

※ 재귀 복습을 단계적으로 풀이 해 놓은 것을 하향식 분석이라고 할 수 있다.

하향식 분석이란, 가장 위쪽에 위치한 상자의 메소드 호출부터 시작해 계단식으로 자세히 조사하는 분석 기법을 말한다. top부터 분석하면 같은 메소드의 호출이 여러 번 나올 수 있기 때문에 "하향식 분석이 반드시 효율적이다" 라고는 할 수 없다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/fa9f454a-737f-49fc-97e4-1439eb21870a" alt width=500>
<em></em>
</center>


### 2) 상향식 분석

: bottom - up으로, 가장 아래쪽부터 위로 쌓아 올리며 분석하는 방법이다. 하향식 분석과는 분석 방향이 반대이다.

- 루프(대표적으로 while)
- 꼬리 재귀(tail recursion)
- 현재 상태 이후가 현재 상태에서 무언가를 더하여 나타나는 경우(ex) $f(n+1) = f(n) + a$)

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/af186365-f929-4b38-8a30-db658eb7cd16" alt width=600>
<em></em>
</center>


- 하향식 접근법은 큰 작업을 작은 하위 작업으로 분해하는 반면 상향식 접근법은 먼저 작업의 다른 기본 부분을 직접 해결 한 다음 해당 부분을 전체 프로그램에 결합하는 방식을 선택합니다.

- 각 하위 모듈은 하향식 방식으로 개별적으로 처리됩니다. 반대로 상향식 접근법은 캡슐화 할 데이터를 검사하여 정보 은닉 개념을 구현합니다.

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ceb62e2e-bb96-4a16-85bc-60903643955b" alt width=600>
<em></em>
</center>



동적 계획(dynamic programming)은 분할 정복과 문제해결 방향이 거꾸로로, 문제의 입력 사례를 분할하여 문제를 푼다는 점은 분할 정복과 비슷하나, 분할한 입력 사례를 재귀 호출하여 답을 얻는 대신, 가장 작은 입력 사례의 답을 먼저 구하여 저장해 놓고, 필요하면 꺼내 쓴다.

<br>


“동적 계획”이라는 용어는 제어 이론(control theory)에서 왔는데, 여기서 계획(programming)이란 해답을 구하기 위해 배열(테이블)을 구축함을 뜻하며 피보나치의 수를 구하는 효율적인 알고리즘은 동적 계획을 이용한 전형적인 사례이다. 인덱스가 0부터 n까지 인 배열에서 앞 부분의 n+1 항을 순서대로 구축하여 n번째 피보나치 항을 구하며, 동적 계획 알고리즘은 배열을 이용하여 상향식으로 해답을 구한다. 따라서 동적 계획은 상향식(bottom-up) 접근 방법이다.

<br>


※동적 계획 알고리즘의 개발절차

1.문제의 입력 사례에 대해서 해답을 계산하는 재귀 관계식(recursive property)을 세운다.

2.작은 입력 사례부터 먼저 해결하는 상향식 방법으로 전체 입력 사례에 대한 해답을 구한다.

<br>


## 2. 이항계수(binomial coefficient) 구하기


값이 큰 n과 k에 대한 이항계수를 구하는 경우 그리 크지 않은 n에 대해서도 n!은 꽤 크기 때문에 다음 재귀 관계식을 사용하면 n!이나 k!을 계산하지 않고 이항계수를 구할 수 있다.
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4402ee57-c4c8-4f29-ab28-24183d3de3b4" alt width="80%">
<em></em>
</center>

<br>
<br>



ex) 이항계수 구하기

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e581a838-29e6-4364-a79f-abc2b74d5802" alt width="80%">
<em></em>
</center>

<br>

이를 동적 계획으로 설계해보자면,

1.재귀 관계식을 세운다. B의 형태로 다시 작성하면,

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/51d498f0-f661-4814-9ee3-981d594d4b36" alt width="80%">
<em></em>
</center>
<br>

2.첫째 행부터 B의 행에 값들을 차례로 계산하여 저장하는 상향식 (bottom–up) 방식으로 문제를 푼다.절차 2의 배열은 파스칼 삼각형, 각 행의 값은 절차1에서 정립한 재귀 관계식을 적용하여 바로 이전에 계산한 값을 가지고 계산한다.

k = 2이기 때문에 앞의 두 열만 계산, 일반화하면 각 행에서 k번째 열까지 값만 계산하면 된다. n = k = 0 일 때 이항계수는 1이므로 B(0)(0) = 1이다.

<br>



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/bd61d86f-bfd0-4e44-a711-727f2dbd025c" alt width=400>
<em></em>
</center>

<br>
<br>

<br>

※ 용어 정리

- ⌈x⌉ : 실수x 보다 크거나 같은 가장 작은 정수

- ⌈x⌉ : x의 올림수(ceiling)라고 하며 모든 정수 n에 대해서 ⌈n⌉=n.

- ⌊x⌋ : 실수x 보다 작거나 같은 가장  큰 정수

- ⌊x⌋ : x의 내림수(floor)라고 하며 모든 정수 n에 대해서 ⌊n⌋=n.

-  ≈  : 결과의 근사치만 구할 수 있는 경우,  “근사하게 같다” 라는 의미

-  ≠ : “같지 않다＂를 의미

-  ∑ : 시그마(sigma), 수열 혹은 임의의 정수에 대한 대상의 합을 축약해서 나타낼 때 사용.

- ∞ : 무한대(infinity),어떤 실수보다도 큰 수로, 모든 실수에 대해 x< ∞이 성립



<br>
<br>


출처: 알고리즘 기초