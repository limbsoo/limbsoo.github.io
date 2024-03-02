---
categories:
  - DX11
tags:
  - DirectX11
  - Direct3D
  - DX11
  - graphics
  - programming
  - cpp
  - solution
---
# 문제
___


Stencil Routed Rendering을 통한 Depth Peeling을 진행 시, 아래와 같이 일부 폴리곤이 렌더링되지 않는 현상이 발생한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/70859e36-9518-4b7c-9044-f2c4bdb294d7" alt width="100%">
<em></em>
</center>




# 해결
___


이는 색상 샘플 방식 차이로 발생하는 것으로,

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e1cbf3cc-91cb-4ae3-a250-134415567be2" alt width="100%">
<em></em>
</center>



DirectX3D의 버전에 따라 샘플 방식이 달라, 발생한다. 


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/51b08485-2755-468e-b6f4-517a5f926e5e" alt width="100%">
<em></em>
</center>


<br>

Polygon이 픽셀의 중앙을 cover하는 것으로만 판단하던 이전 버전의 GPU와는 달리 현재의 GPU는 renderTarget마다 각각의 Sample position을 가짐, 따라서 renderTarget마다 Sample position이 다르면 Polygon이 픽셀의 일부 renderTarget만 Cover하는 경우가 발생

<br>

이로 인해, Polygon이 Cover하는 renderTarget과 연결된 Stencil buffer의 Stencil 값만 1씩 감소하게 되어 Stencil 값이 감소하지 않은 renderTarget의 Stencil값 과 동일해지고 같은 Stencil값을 가진 renderTarget들은 같은 Polygon의 색상을 저장하여 layer가 감소한다.



<br>



이를 해결하기 위해 기존 렌더링으로 출력한 이미지를 stencil 레이어의 첫 번째 이미지를 대체하여 방지하고, 샘플 픽셀의 depth를 주변 픽셀과 비교한다.

<br>

손실된 픽셀의 depth는 주변 픽셀에 비해 큰 값을 가지며, 이를 많이 샘플링 할수록 픽셀의 depth 평균이 주변 픽셀보다 커진다. 


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/3a6ef1cb-ff0f-4372-a72f-48c5e9ae4309" alt width="100%">
<em></em>
</center>

<br>

따라서 이를 비교하여 손실된 픽셀 렌더링을 방지한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a5447479-343f-490e-97f5-06481941d131" alt width="100%">
<em></em>
</center>

<br>


이러한 과정을 통해 기존보다 향상된 결과를 얻을 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a70c4e6f-5373-490d-8ca7-45893993766a" alt width="100%">
<em></em>
</center>