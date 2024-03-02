---
categories:
  - Reference
tags:
  - reference
  - paper
  - graphics
---
# 개요
___

## 1. Stencil Routed K-Buffer

스텐실 라우팅을 사용, 지오메트리 패스당 픽셀당 8개의 조각 레이어를 캡처

1. 픽셀당 최대 16개의 조각이 2개의 지오메트리 패스에서 래스터화 순서로 캡처

2. fullscreen shader pass는 bitonic 정렬을 사용하여 픽셀당 16개의 조각을 정렬

3. 다른 fullscreen shader pass는  알파 혼합 또는 개별 레이어를 기반으로 하는 반투명 효과를 렌더링

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/14d50841-3ae4-4bde-9739-24d27e1dba9f" alt width=500>
<em></em>
</center>


레이어를 깊이 순서대로 캡처하는 깊이 필링과 비교 시, 정렬보다 지오메트리를 렌더링하는 데 비용이 더 많이 들고 모든 조각을 캡처해야 하는 경우 더 빠르다.



