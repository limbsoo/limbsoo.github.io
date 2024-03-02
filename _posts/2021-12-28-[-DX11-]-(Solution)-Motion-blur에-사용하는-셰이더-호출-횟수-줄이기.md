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

모션블러 효과를 생성하기 위해 여러 개의 이미지들을 생성, 이러한 작업을 프레임마다 진행해야 한다. 이로 인해 프래그먼트 셰이더의 호출이 매우 빈번해져 성능에 영향을 줄 수 있으므로 fullscreen quad를 사용, 프래그먼트 셰이더 호출 횟수를 줄인다.

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/c85c034c-c06e-4e4d-a6f2-3999f4bd9e2a" alt width=700>
<em>1st pass – Make Texture</em>
</center>

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0fda0bf5-4704-4235-b19d-6b7fdaa33781" alt width=700>
<em>2nd  Pass – Draw Motion Blur in Full Screen Quad</em>
</center>


<br>
<br>


# 해결
___

뷰포트를 생성하기 위해 하나의 큰 사각형을 만들어야 하나, 사각형은 2개의 삼각형으로 이루어져, 프래그먼트 셰이더를 2번 호출해야 한다. 그러나 하나의 큰 삼각형을 그려, 화면을 채운다면 1개의 삼각형으로도 뷰포트를 생성할 수 있다. 이는 레스터라이저에서 내부 픽셀에 대해서만 프래그먼트를 생성하는 것을 이용하여 다른 영향 없이 프래그먼트 셰이더의 호출 횟수를 감소시킬 수 있다.


<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a043e50e-0751-43a4-84f5-8079abf2d87c" alt width="70%">
<em>Full Screen Quad</em>
</center>


<br>
<br>

사각형의 정보를 임포트하여,

```
#FSQ

v  -1 -1 0
v  -1 1 0
v  1 -1 0
v  1 1 0

f 2/2/1 1/1/1 3/3/1 
f 2/2/1 3/3/1 4/4/1 
```

<br>
<br>

쉐이더에서 단일 삼각형의 뷰포트를 생성할 수 있다.

```
struct QuadOutput
{
   float4 pos : SV_Position;
   float2 tex : TEXCOORD0;
};

QuadOutput FullScreenTriVS(uint id : SV_VertexID)
{
   QuadOutput output = (QuadOutput)0.0f;
   output.tex = float2((id << 1) & 2, id & 2);
   output.pos = float4(output.tex * float2(2.0f, -2.0f) + float2(-1.0f, 1.0f), 0.0f, 1.0f);

   return output;
}
```

<br>
<br>

그리고 이 정보를 토대로 픽셀 셰이더에 넘겨, 풀스크린쿼드 이미지를 생성한다.

```
technique11 Rendering
{
   pass P0
   {
      SetVertexShader(CompileShader(vs_5_0, FullScreenTriVS()));
      SetPixelShader(CompileShader(ps_5_0, Rendering()));
      SetRasterizerState(rasterizerState_New);
   }
}
```



