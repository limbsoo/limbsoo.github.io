---
categories:
  - Algorithm
tags:
  - algorithm
use_math: true
---
<br>

# 알고리즘
___

컴퓨터 프로그램은 정렬하기(sorting)와 같은 특정 작업을 수행하는 개별 모듈(module)을 모아서 구성하고 프로그램 전체 설계보다 특정 과제를 수행하는 개별 모듈 설계에 중점을 주는 특정 과제를 **문제**(problem)라고 한다.

<br>

## 1. 알고리즘

문제를 푸는 단계 별 절차, 문제를 푸는 기법은 다양하지만, 기법에 따라서 알고리즘의 성능은 차이가 날 수 있다. 그러므로 어떤 문제를 어떤 기법으로 풀 것인지, 시간(time)과 공간(space)의 사용량을 기준으로 얼마나 **효율적**인지 분석하는데 관심을 가져야 한다.

※ 시간: 알고리즘을 컴퓨터로 실행 할 때 걸리는 시간의 길이
※ 공간: 필요한 메모리의 크기

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/884f28d9-fe08-4930-b5af-831f08cb1b33" alt width="100%">
<em>알고리즘 문제</em>
</center>


<br>

- 파라미터(parameter): 문제에서 값이 지정되어 있지 않은 변수
- 입력 사례(instance): 파라미터에 지정할 값
- 리스트(list): 어떤 원소를 특정 순서로 나열해 놓은 것.
- 해답(solution) : 파라미터를 입력 사례로 지정하여 질문한 문제의 해답

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/51d485fc-b1fb-4903-b357-b0a44975d0f0" alt width="100%">
<em></em>
</center>
<br>

※프로시저: 특정한 로직을 처리하기만 하고 결과 값을 반환하지 않는 서브 프로그램. Function과 같은 기능이나 Function은 리턴값을 강조하는 한편, 프로시저는 중간처리과정 중요시한다.


## 2. 효율적 알고리즘
### 1) 이분 검색, 순차 검색

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/c9966099-008d-4800-b0c4-5b1984eb410f" alt width=400>
<em>이분 검색, 순차 검색</em>
</center>
<br>

- 이분 검색 : $x$를 찾기 위해 $x$를 배열 정중앙에 위치한 원소와 비교, 같다면 찾았기에 알고리즘을 끝내고, 그 원소보다 작다면 전반부, 크다면 후반부의 정중앙에 위치한 원소와 비교해 이분 검색 절차를 되풀이 하는 방법

- 순차 검색 : 순서대로 원소를 비교함으로써, 리스트에서 찾고자 하는 값을 맨 앞에서부터 끝까지 차례대로 찾아 나가는 검색 방법

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/919873f6-e065-45eb-84a4-d426ae00de88" alt width=500>
<em>X가 배열의 모든 원소보다 클때 순차검색과 이분검색에서의 비교횟수</em>
</center>

<br>

### 2) 피보나치 수열

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/708e6c5d-ee99-4e41-b29d-cb98541a1ca0" alt width="100%">
<em>피보나치 수열</em>
</center>

<br>

피보나치 수열의 $n$번째 수는 다음과 같이 재귀로(recursively: 자신을 되부르는 꼴로) 정의. 재귀로 정의하기 때문에 정의를 그대로 재귀 알고리즘으로 바꾸는 것 가능
<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ae5171ce-1b8d-4d96-a63f-fd1f1b6f13a0" alt width=500>
<em>알고리즘에 따른 차이</em>
</center>
<br>

## 2. 알고리즘 분석
<br>


분석 이론을 적용하면서 단위 연산, 주변 명령문,제어 명령문을 실행하는데 실제 걸리는 시간들을 고려

<br>

### 1) 알고리즘의 단위 연산(basic instruction)

입력 크기를 구한 후 , 어떤 명령문이나 명령문 덩어리(군)를 선정하여 알고리즘이 실행한 총 작업의 양을 이에 실행한 횟수에 대략적으로 비례하도록 함. 그 때의 명령문이나 명령문 군을 칭함.

<br>

### 2) 알고리즘의 시간 복잡도 분석(time complexity analysis)

입력 크기를 기준으로 단위 연산을 몇 번 수행하는지 구함. 제어 구조를 구성하는 명령문은 보통 단위 연산으로 취급X


- **일정 시간 복잡도**(every-case time complexity) : 실행 횟수가 일정한 경우 $T(n)$을 입력 크기 $n$에 대해서 알고리즘이 단위 연산을 실행하는  횟수로 정의하는데, 여기서 $T(n)$을 칭함
	- 일정 시간 복잡도 분석 : $T(n)$을 구하는 과정

- **최악 시간 복잡도**(worst –case time complexity) : 단위 연산을 수행하는 최대 횟수를 측정, $W(n)$을 입력 크기 $n$에 대해서 알고리즘이 실행할 단위 연산의 최대 횟수로 정의하는데 그때의 $W(n)$ 값
	- 최악 시간 복잡도 분석: $W(n)$을 구하는 과정, $T(n)$값이 존재하면 $W(n) = T(n)$

- **평균 시간 복잡도**(average-case time complexity) : 어떤 알고리즘에 대해 $A(n)$을 입력 크기 $n$에 대해 그 알고리즘이 수행할 단위 연산의 평균 횟수(기대치)로 정의할 때, 그때의  $A(n)$ 값
	- 평균 시간 복잡도 분석: $A(n)$을 구하는 과정 :  $W(n)$과 같이 $T(n)$이 존재하면 $A(n)= T(n)$

- **최선 시간 복잡도**(best-case time complexity analysis) : 어떤 알고리즘에 대해, $B(n)$을 입력 크기 $n$에 대해 그 알고리즘이 실행할 단위 연산의 최소 횟수로 정의할 때 그때의  $B(n)$ 값
	- 최선 시간 복잡도 분석:  $B(n)$을 구하는 과정, $W(n)$과 $A(n)$의 경우와 같이 만약$T(n)$이 존재하면 $B(n)=T(n)$.

<br>

메모리(=공간 복잡도(memory complexity)의 분석(알고리즘이 메모리 사용 면으로 얼마나 효율적인지 분석)에도 적용할 수 있으며 **복잡도 함수**(complexity function)는 양의 정수에서 양의 실수(0포함)로 매핑하는 함수로, 특정 알고리즘의 시간 복잡도나 메모리 복잡도를 가리키지 않을 때는 복잡도 함수를 보통 $f(n)$과 $g(n)$같은 표준 함수법으로 표현한다.

<br>

### 3) 명령문(instructions)

- **주변 명령문**(overhead instructions) : 루프 이전에 실행되는 초기화문 같은 명령문, 실행 횟수는 일반적으로 입력 크기에 상관없이 항상 일정

- **제어 명령문**(control instructions) : 루프를 제어하기 위한 인덱스 증가 명령문 같은 명령문, 실행 횟수는 입력 크기에 비례하여 증가

<br>

## 3. 정확성 분석

알고리즘이 기대한 대로 실행되는지 증명하는 것

<br>











출처: 알고리즘 기초