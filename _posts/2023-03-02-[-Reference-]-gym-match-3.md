---
categories:
  - Reference
tags:
  - reference
  - python
  - reinforcementLearning
  - match-3Game
---
# 개요
___

gym match3는 강화 학습을 위한 환경으로, Match-3 보드 복제를 통해 새로운 레벨 생성


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/69492785-d8a3-4107-a50f-05f27813799b" alt width=700>
<em></em>
</center>


## 1. Console command

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/6f02356c-48c6-4d93-8a70-92a6a4bbf916" alt width=400>
<em></em>
</center>

1. Import Match3Env 및 선언
2. 활성화 함수 호출
3. observation, reward, done, info = env.step(env.action_space.sample())



## 2. Initialize


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/5fae62fa-b65a-43bf-886a-e78c1180b57d" alt width=600>
<em></em>
</center>


### 1) level.py

레벨 설정

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8a34710c-2741-4bc1-abe1-f587f434825a" alt width=600>
<em></em>
</center>

Levels에 저장된 level(0,-1로 이루어진 일정 패턴을 가진 array)들의 Height, width, shapes의 최댓값을 dimenstion으로 설정(level들 간 크기 차이가 있어도 학습 가능)

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/88d82052-52fc-4336-b30f-9fbbd102a1d9" alt width=300>
<em>Level(height, width, shapes)</em>
</center>

### 2) game.py

게임 설정

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7b576bbd-286c-4026-8d1d-ca06186ba59b" alt width=600>
<em></em>
</center>




매치 가능 블록 탐색 방향, 이동 방향 설정 : 2차원 배열이므로 위, 아래, 왼쪽, 오른쪽

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/59697a33-f3ff-46ff-afe2-5dea9ed79289" alt width=600>
<em></em>
</center>


**Reset**()을 통한 보드 설정,Levels 중 1개를 랜덤하게 선택 후 이를 template으로 지정.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4b1f6612-1dcf-48e2-a5d8-7b8cebcc057c" alt width=600>
<em></em>
</center>




<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ae172db1-178c-4714-9386-0da04678789b" alt width=700>
<em></em>
</center>


- Template과 같은 크기의 empty board를 생성하여 0부터 Level.n_shapes까지의 랜덤하게 숫자 채움

- 저장한 최대 dimension보다 작을 경우 패딩 하여 크기를 늘려 줌.

- Empty board와 Expand template 결합하여 보드 생성


**get_available_actions**()을 통한 액션 설정

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4b1f6612-1dcf-48e2-a5d8-7b8cebcc057c" alt width=600>
<em></em>
</center>

- 현재 Board의 각 cell들을 point로 설정하고 이 point가 이동할 수 있는 가짓수를 중복하지 않게 순서 없이 저장한다.


## 3. step

**step**(), 활성화 함수를 통해 플레이를 진행

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/95462411-ebdf-4228-aa48-06a42993ae39" alt width=600>
<em></em>
</center>


- 첫 번째 action 선택 후 이를 swap

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7c49c8bf-758f-443f-ab15-368979486c4a" alt width=400>
<em></em>
</center>















