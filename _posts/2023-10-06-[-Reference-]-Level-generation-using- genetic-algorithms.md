---
categories:
  - Reference
tags:
  - reference
  - paper
  - geneticAlgorithm
---
# 개요
___




Fitness를 사용하는 대신, 선택 기준을 사용. 대칭적이고 좋은 디자인을 가진 레벨이 이미 다수 존재할 때, 이를 선택 기준으로 삼아 기존 맵들과는 다른 새로운 맵을 생성한다.




그리드의 대칭을 유지하기 위해 5 종류의 크로스오버 연산을 사용하여 그리드를 분할하고 해당 부분에서 연산을 수행. 

- 그리드의 중간을 세로 혹은 가로로 자름. 길이가 홀수일 경우, 확률을 부여해  선택. 
	(ex) grid.col = 7 , 4 / 3  

- 그리드의 중심을 기준으로 사각형 4개로 나눔.이 또한 길이가 홀수일 경우, 확률을 부여해 선택. 
	(ex)  4x4, 4x3, 3x4, 3x3 : 1/4의 확률 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4779af78-4723-4ec2-b707-cac3e5815b71" alt width=300>
<em></em>
</center>


- 그리드의 중심을 기준으로 십자 모양의 열과 행을 하나의 section으로 설정하고 나머지 부분을 하나의 section으로 분할.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/bebfc52f-3581-409d-b765-5e5cef7eca28" alt width=300>
<em></em>
</center>



- 그리드의 중심을 기준으로 X자 모양의 열과 행을 하나의 section으로 설정하고 나머지 부분을 하나의 section으로 분할. 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a6f1d3ab-1108-4755-bbe3-609c984ccd8b" alt width=300>
<em></em>
</center>


Crossover 연산과 함께 mutate또한 사용 (더 독특한 level 생성)


유전자 연산


Crossover 연산
- 위치 변경
	ex) fig.1의 4x4, 4x3, 3x4, 3x3 section의 순서를 변경하여 3x3, 3x4, 4x3, 4x4의 section으로 나뉜 그리드를 생성.

- 회전 사용 - 일부 section 혹은, 전체 section을 회전시키는 local, global 회전을 사용하거나, 일부 section을 시계 방향으로, 일부 section은 반시계 방향으로 회전해 같은 크기의 그리드에 배치하여 적합한 레벨의 그리드 생성

Mutate
- 이러한 연산이 진행되는 장애물과 같은 블록을 빈 cell 등의 다른 장애물 유형으로 변경, 이 때 동일 장애물 유형은 사용하지 않음


1번 레벨의 section을 뒤집어 새 레벨의 맨 아래 배치. (stone이 ice로 변경)
2번 레벨의 section을 새 레벨의 맨 위에 배치. (ice가 빈 cell이 되고 stone은 더 강한 stone으로 변경)


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/2382af23-6421-4bb6-9b68-d6eeaf3270a2" alt width=600>
<em></em>
</center>


그러나, 이러한 알고리즘에서 생성된 레벨이 실제 플레이 시, 클리어 될 수 있는지는 불투명하며, 테스트 하지 않음. 이 연구에서 사용된 유전 알고리즘의 목적은 보기 좋은 고유 레벨만 생성.











출처 : LEVEL GENERATION USING GENETIC  ALGORITHMS  AND DIFFICULTY TESTING USING REINFORCEMENT LEARNING  IN MATCH-3  GAME


