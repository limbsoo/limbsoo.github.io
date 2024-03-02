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
  - motionblur
---
# 문제
___


레퍼런스와 depthpeeling 간 비교를 위해 많은 삼각형의 모델들과 조명 효과 등을 통해 높은 성능이 요구되는 환경에서 이를 비교하고자 한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/3b14b3a7-6f2d-4f7e-a038-ae1db76199a9" alt width="100%">
<em></em>
</center>

<br>

그러나 조명이 적절하지 않게 적용되는 경우가 발생한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d047c759-7d05-4c57-ad93-55b5516fbd93" alt width="100%">
<em></em>
</center>









# 해결
___


이는 다른 부분에서는 적절한 조명 효과가 적용되었으나, 일부분에서 발생하였는데, 특히 일부 텍스처에서 발생하였다. 이러한 문제로 인해 셰이더를 확인하였으나 문제가 없었고, 문제는 d3d11_input_element_desc에서 발생하였다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/be575e7c-10fb-4441-b6d8-be19e71efcb4" alt width="100%">
<em></em>
</center>


<br>

d3d11_input_element_desc에서의 AlignedByteOffset은 꼭짓점의 시작부터 오프셋(바이트)인데, DXGI_FORMAT_R32G32B32_FLOAT은 총 12바이트, DXGI_FORMAT_R32G32_FLOAT는 8바이트로, normal은 20으로, 적절 값이 적용되지 않았다.
<br>


따라서 이를 20으로 수정하면 적절한 결과를 얻을 수 있다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ce68e8d1-6374-4814-aba2-b82b46951be0" alt width="100%">
<em></em>
</center>

<br>

그리고 제일 쉬운 방법은 D3D11_APPEND_ALIGNED_ELEMENT를 사용, 이전 요소 바로 다음에 현재 요소를 정의하는 것이다.