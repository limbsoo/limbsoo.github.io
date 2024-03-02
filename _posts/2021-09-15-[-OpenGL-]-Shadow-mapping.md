---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
---
# Shadow mapping
___

쉐도우 매핑 구현을 위해 쉐도우 매핑 2패스를 구현한다.


먼저, 기존 코드는 하나의 모델만을 렌더링하는 코드로, 쉐도우 매핑을 위해 코드를 수정

```c++

Object.h

#include"Matrix4.h"
#include"Vector4.h"
#ifdef _DEBUG
#include <assert.h>
#define ASSERT(p) assert(p)
#define VERIFY(p) assert(p)
#else
#define ASSERT(p)
#define VERIFY(p) p
#endif

#include "glut.h"
#define NUM_MAX_VERTEX_PER_FACE 10
#define NUM_MAX_VERTEX 10000
#define NUM_MAX_FACE 10000

typedef struct
{
	int m_nNumVertex;
	unsigned char m_color[3];
	int m_vertex[NUM_MAX_VERTEX_PER_FACE];

} Face_t;

class Object
{

public:

	Object() : m_nNumVertex(0), m_nNumFace(0)
	{
		m_world = identityMatrix();
	}

	Matrix4 m_world;
	int m_nNumVertex, m_nNumFace;
	float m_vertex[3000][3];
	Face_t m_face[6000];
	float m_tramsformedVertex[3000][3];
	float m_uv[3000][2] = {};
	float m_rVector[6000][3] = { 0 };
	float m_worldVertex[3000][3];
	float m_worldVertexNormal[6000][3] = { 0 };
	float m_depthVector[6000][3] = { 0 };
	float m_faceNormal[6000][3] = { 0 };
	float m_transformedFaceNormal[6000][3] = { 0 };
	float m_vertexNormal[6000][3] = { 0 };
	float m_transformedVertexNormal[6000][3] = { 0 };
	void readFile(char* pFileName);
	void getNewLine(char* buff, int count, FILE* pFile);
	void makefaceNormals();
	void makeVertexNormals();
	void makeVertexUVs();
	void applyMatrix(Matrix4 m, Matrix4 n);
	void makeRVector(Vector4 eye);
	void makeWorldVertex();
	void makeWorldVertexNormal();
	void makeVeritces();
	void makeWorldLightingDepth(Vector4 directionLight);
	void makeWorldCameraDepth(Vector4 eye);
};
```
<br>

```c++
Object.cpp
#include "Object.h"
#include "Renderer.h"
void Object::makeVertexNormals()
{
	for (int i = 0; i < m_nNumFace; i++) // //점의 노말은 점이 속한 삼각형 노말의 평균
	{
		for (int j = 0; j < 3; j++)
		{
			m_vertexNormal[m_face[i].m_vertex[j] - 1][0] += m_faceNormal[i][0];
			m_vertexNormal[m_face[i].m_vertex[j] - 1][1] += m_faceNormal[i][1];
			m_vertexNormal[m_face[i].m_vertex[j] - 1][2] += m_faceNormal[i][2];
		}
	}

	for (int i = 0; i < m_nNumVertex; i++)
	{
		Vector4 v;
		v.x = m_vertexNormal[i][0];
		v.y = m_vertexNormal[i][1];
		v.z = m_vertexNormal[i][2];
		v = normalize(v);
		m_vertexNormal[i][0] = v.x;
		m_vertexNormal[i][1] = v.y;
		m_vertexNormal[i][2] = v.z;
	}
}
...
```

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/6a033c71-ca81-4597-b5b4-fb16990f74d5" alt width=500>
<em>1 Pass, 깊이맵 생성</em>
</center>

<br>

쉐도우맵을 위해 광원과 카메라 위치에 따른 깊이 값이 모두 필요하므로 먼저 광원과 카메라의 위치를 일치시켜 광원에 대한 깊이맵을 생성한다.

```c++  
Vector4 lightPosition(2.f, -2.f, -2.f);
Vector4 ndirectionLight = normalize(lightPosition);

//카메라의 eye 값을 광원의 lighting Position과 일치시킨다.
eye.x = 2.f;	
eye.y = -2.f;	
eye.z = -2.f;

// matrix에 적용
m_view = lookAt(eye, at, upVector);
mvp = m_proj * m_view * g_cow.m_world;
```

<br>


```c++  
void Renderer::render()
{
	g_cow.makeWorldLightingDepth(lightPosition);
	g_cow.makeVeritces();
	g_cow.makeRVector(eye);
	g_cow.applyMatrix(mvp);
	render(&g_cow);
	lightingDepthBuffer = m_zBuffer;
}

void Object::makeWorldLightingDepth(Vector4 lightPosition)
{
	for (int i = 0; i < m_nNumVertex; i++)
	{
		Vector4 postition(m_worldVertex[i], 1);
		postition = lightPosition - postition;
		m_depthVector[i][0] = postition.x;
		m_depthVector[i][1] = postition.y;
		m_depthVector[i][2] = postition.z;
	}
}

void Object::makeVeritces()
{
	makeVertexUVs();
	makeWorldVertex();
	makefaceNormals();
	makeVertexNormals();
	makeWorldVertexNormal();
}

void Object::makeRVector(Vector4 eye)
{
	float cos;
	Vector4 s;

	for (int i = 0; i < m_nNumVertex; i++)
	{
		Vector4 normal(m_worldVertexNormal[i], 1);
		Vector4 postition(m_worldVertex[i], 1);
		postition = eye - postition;
		postition = normalize(postition);
		cos = normal * postition;
		s = normal * cos;

		s.x = (s.x * 2) - postition.x;
		s.y = (s.y * 2) - postition.y;
		s.z = (s.z * 2) - postition.z;

		m_rVector[i][0] = s.x;
		m_rVector[i][1] = s.y;
		m_rVector[i][2] = s.z;
	}
}

```

<br>


```c++


void Renderer::buildEdgetable(Object *pObject, int nFace)
{
	float vertices[2][3] = { 0 };
	float uvVertices[2][2] = { 0 };
	float nVertices[2][3] = { 0 };
	float dVertices[2][3] = { 0 };
	float rVertices[2][3];
	int ymin = 0;
	float savedY = 0;
	float yMax = 0;

	
	for (int i = 0; i < pObject->m_face[nFace].m_nNumVertex; i++)
	{
		for (int j = 0; j < 3; j++)
		{
			vertices[0][j] = pObject->m_tramsformedVertex[pObject->m_face[nFace].m_vertex[i] - 1][j];
			vertices[1][j] = pObject->m_tramsformedVertex[pObject->m_face[nFace].m_vertex[(i + 1) % pObject->m_face[nFace].m_nNumVertex] - 1][j];
			nVertices[0][j] = pObject->m_worldVertexNormal[pObject->m_face[nFace].m_vertex[i] - 1][j];
			nVertices[1][j] = pObject->m_worldVertexNormal[pObject->m_face[nFace].m_vertex[(i + 1) % pObject->m_face[nFace].m_nNumVertex] - 1][j];
			rVertices[0][j] = pObject->m_rVector[pObject->m_face[nFace].m_vertex[i] - 1][j];
			rVertices[1][j] = pObject->m_rVector[pObject->m_face[nFace].m_vertex[(i + 1) % pObject->m_face[nFace].m_nNumVertex] - 1][j];

			dVertices[0][j] = pObject->m_depthVector[pObject->m_face[nFace].m_vertex[i] - 1][j];
			dVertices[1][j] = pObject->m_depthVector[pObject->m_face[nFace].m_vertex[(i + 1) % pObject->m_face[nFace].m_nNumVertex] - 1][j];
		}

		for (int j = 0; j < 2; j++)
		{
			uvVertices[0][j] = pObject->m_uv[pObject->m_face[nFace].m_vertex[i] - 1][j];
			uvVertices[1][j] = pObject->m_uv[pObject->m_face[nFace].m_vertex[(i + 1) % pObject->m_face[nFace].m_nNumVertex] - 1][j];
		}
...
}


void Renderer::fill(GLubyte color[3])
{
	...
	for (k = xmin; k < xmax; k++)
	{
		Vector4 Normal(m_AET[j].nx + deltaNX, m_AET[j].ny + deltaNY, m_AET[j].nz + deltaNZ);
		float light = Normal * ndirectionLight;
		light = max(0.f, light * 255);

		sx = (m_AET[j].nx + deltaNX) * (ndirectionLight.x);
		sy = (m_AET[j].ny + deltaNY) * (ndirectionLight.y);
		sz = (m_AET[j].nz + deltaNZ) * (ndirectionLight.z);

		scalaNL = sx + sy + sz;

		if (m_AET[j].z + deltaZ < m_zBuffer[i][k])
		{
			int s, t;
			Vector4 r;
			r.x = m_AET[j].rx + deltaRX;
			r.y = m_AET[j].ry + deltaRY;
			r.z = m_AET[j].rz + deltaRZ;
			scalaN = sqrt(pow(r.x, 2) + pow(r.y, 2) + pow(r.z + 1, 2));
			s = (r.x / scalaN + 1) * 320;
			t = (r.y / scalaN + 1) * 240;

			checkImage[i][k][0] = (GLubyte)m_AET[j].lx;
			checkImage[i][k][1] = (GLubyte)m_AET[j].ly;
			checkImage[i][k][2] = (GLubyte)m_AET[j].lz;
			m_zBuffer[i][k] = m_AET[j].z + deltaZ;
		}

		deltaU += uPerX;
		deltaV += vPerX;
		deltaZ += zPerX;

		deltaNX += nxPerX;
		deltaNY += nyPerX;
		deltaNZ += nzPerX;

		deltaRX += rxPerX;
		deltaRY += ryPerX;
		deltaRZ += rzPerX;

		deltaLX += lxPerX;
		deltaLY += lyPerX;
		deltaLZ += lzPerX;
	}
}

```

<br>



그리고 카메라를 다시 원래 위치로 되돌려 렌더해 카메라에 대한 깊이 맵을 생성한다.

```c++  
//카메라를 원래의 eye 위치에 위치시킨다.
eye.x = 1.f;
eye.y = -1.f;
eye.z = 1.f;
m_view = lookAt(eye, at, upVector);

// matrix에 적용
m_view = lookAt(eye, at, upVector);
mvp = m_proj * m_view * g_cow.m_world;

void Renderer::render()
{
	g_cow.makeWorldLightingDepth(lightPosition);
	g_cow.makeVeritces();
	g_cow.makeRVector(eye);
	g_cow.applyMatrix(mvp);
	render(&g_cow);
	cameraDepthBuffer = m_zBuffer;
}
```


<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/60d80d97-a37c-4d8b-a0de-c87281c77b22" alt width=500>
<em>2 Pass, 그림자 강도 계산</em>
</center>

<br>


카메라 기준 깊이 값이 광원 기준 깊이 값보다 작으면 모델의 색상을 렌더링하고, 아니라면 그림자를 렌더링한다.

```c++

void Renderer::fill(GLubyte color[3])
{
	...
	if (cameraDepthBuffer[i][k] < lightingDepthBuffer[i][k])
	{
		checkImage[i][k][0] = (GLubyte)m_AET[j].lx;
		checkImage[i][k][1] = (GLubyte)m_AET[j].ly;
		checkImage[i][k][2] = (GLubyte)m_AET[j].lz;
	}

	else
	{
		checkImage[i][k][0] = (GLubyte)0;
		checkImage[i][k][1] = (GLubyte)0;
		checkImage[i][k][2] = (GLubyte)0;
	}
}
```

<br>

<br>

# 출력
___

원하는 위치에 그림자가 생성되는 것을 확인할 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/b9da463b-1f15-45a9-a435-336754a474f7" alt width=500>
<em>쉐도우맵 출력 </em>
</center>