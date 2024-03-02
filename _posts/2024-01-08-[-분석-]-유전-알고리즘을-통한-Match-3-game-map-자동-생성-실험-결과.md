---
categories:
  - 기타
tags:
  - match-3Game
  - regression
---
# 분석
___


특수 블록이 배치되지 않은 서로 다른 크기의 맵에서 목표달성에 필요한 스왑 횟수 측정 후,

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/11a3f4fc-7cd3-404f-8134-ebae1d3a1611" alt width=400>
<em>자동플레이를 통한 특수 블록 미배치 맵의 스왑매치 경우의 수에 따른 평균 스왑 횟수 측정</em>
</center>

<br>


특수 블록의 배치에 따른 스왑 횟수 증가량을 측정하였다.




<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d8af78f4-6446-4bde-9c16-5d74a0b84d30" alt width=600>
<em>특수 블록의 종류별 맵 크기와 스왑매치 경우의 수 감소에 따른 평균 스왑 횟수 측정</em>
</center>


<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/cf0ae02d-3d65-494a-a5ba-9362109c9545" alt width=500>
<em>특수 블록의 종류별 맵 크기에 따른 스왑 횟수 증가</em>
</center>


<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/6a46c117-6c21-428a-9061-1d25cfca13f0" alt width=500>
<em>특수 블록의 종류별 맵 크기에 따른 스왑 횟수 증가량</em>
</center>


<br>
<br>



특수 블록의 종류별 맵 크기에 따른 스왑 횟수 증가를 통해 배치되는 특수 블록의 종류에 따른 예측 정확도를 파악하기 위해 MSE를 계산했다

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d991de33-6fc2-4474-ad8d-430c94ef7b8e" alt width=500>
<em>특수 블록 배치 종류에 따른 맵 크기 별 예측값 비교</em>
</center>


<br>

낮은 스왑 증가 값을 가질수록 좋은 예측 정확도를 보임을 확인할 수 있었다. 그리고 여러 특수 블록이 배치된 맵의 경우, 하나의 특수 블록 배치 시보다 높은 MSE 값을 보이나, 여전히 낮은 예측 정확도를 유지하는 것을 확인할 수 있었다.



<br>

<br>



Fi-2pop을 활용한 유전 알고리즘은 infeasible map, 즉 실제 플레이가 불가능한 맵을 진화 연산에 사용하여 feasible map 간 자식을 통한 해집단 이동을 발생시키는 알고리즘이다. 이를 infeasible map을 진화 연산에 사용하지 않는 기존 유전알고리즘과 비교했다


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0723492f-01a9-47f8-9116-799ccac1fd07" alt width=500>
<em>Fi-2pop과 기존 유전알고리즘 간 적정 스왑매치 경우의 수 탐색 시 사용 세대 수 비교</em>
</center>

<br>



원하는 스왑매치 경우의 수를 가진 맵을 찾는데 필요한 세대 수를 각각의 알고리즘에서 측정하였다. 경우의 수당 30개의 표본에서 55의 허용오차를 두고 측정하였을 때 기존 유전알고리즘은 300, 400의 스왑매치 경우의 수 탐색에서 Fi-2pop 알고리즘보다 적은 세대 탐색 수를 보이며 우세했으나, 500 이후부터 급격히 증가하였다. 이러한 결과를 통해 일정 이상의 스왑 매치 경우의 수 탐색 시 Fi-2pop 알고리즘이 기존 유전알고리즘보다 높은 성능을 보임을 확인했다.
