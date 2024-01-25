---
categories:
  - 컴퓨터 그래픽스
tags:
  - OpenGL
  - 컴퓨터그래픽스
  - 구현
  -  C++
---
# 구현
___

sphere environment mapping을 진행하였다.


환경맵은 주변 환경을 반사하는 매끄러운 물체를 렌더링하는 기법으로 주변 환경을 텍스처에 저장한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/fb6510ed-14bd-42c9-b554-c7ff6fb154c7" alt width="80%">
<em>환경맵</em>
</center>




일반적으로 많이 사용하는 큐브 맵의 반사색상 결정 공식이다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7a4d5602-836e-4e6f-94fc-887450eccdd1" alt width="80%">
<em>지역 조명</em>
</center>



큐브 맵 대신 Sphere map을 사용하는 경우 사용하는 식



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d3710771-ee52-4ff7-8417-88184c1defc3" alt width="80%">
<em>지역 조명</em>
</center>



해당 Sphere map을 임포트하고 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4d89e974-92ee-472e-b877-eed3494f802f" alt width="40%">
<em>지역 조명</em>
</center>


Sphere map 공식을 적용한 렌더링을 진행했다.

```C++

void Renderer::fill(GLubyte color[3])
{
	...
	for (k = xmin; k < xmax; k++)
	{
		sx = (m_AET[j].nx + deltaNX) * (ndirectionLight.x);
		sy = (m_AET[j].ny + deltaNY) * (ndirectionLight.y);
		sz = (m_AET[j].nz + deltaNZ) * (ndirectionLight.z);

		scalaNL = sx + sy + sz;
		
		rx = 2 * (m_AET[j].nx + deltaNX) * scalaNL - ndirectionLight.x;
		ry = 2 * (m_AET[j].ny + deltaNY) * scalaNL - ndirectionLight.y;
		rz = 2 * (m_AET[j].nz + deltaNZ) * scalaNL - ndirectionLight.z;

		n = sqrt(pow(rx, 2) + pow(ry, 2) + pow(rz+1, 2));

		if (m_AET[j].z + deltaZ < m_zBuffer[i][k])
		{
			s = max(rx / n,0.f) * 640;
			t = max(ry / n,0.f) * 480;

			checkImage[i][k][0] = (GLubyte)m_texture[t][s][0];
			checkImage[i][k][1] = (GLubyte)m_texture[t][s][1];
			checkImage[i][k][2] = (GLubyte)m_texture[t][s][2];
			m_zBuffer[i][k] = m_AET[j].z + deltaZ;
		}
		deltaU += uPerX;
		deltaV += vPerX;
		deltaZ += zPerX;
		deltaNX += nxPerX;
		deltaNY += nyPerX;
		deltaNZ += nzPerX;
	}
}
```

# 출력
___

sphere map을 통한 환경맵이 적용된 모델이 렌더링 되는 것을 확인할 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/92b52dbe-db87-40bc-95b2-2c2c6382bc35" alt width="80%">
<em>쉐도우맵 출력 </em>
</center>