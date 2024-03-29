---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
---

# Lighting
___

라이팅을 위해 suface normal, vertex normal을 계산한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d567ab3b-aeda-4c3d-ba50-5d7bfb58b8b6" alt width=600>
<em>P1,P2,P3 삼각형의 suface normal</em>
</center>

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d0ea8268-2574-48f7-a350-7feea46898b4" alt width=700>
<em>surface normal의 평균을 통해 vertex normal 계산</em>
</center>

<br>


```c++
void Renderer ::makefaceNormals() //외적 
{
	for (int i = 0; i < m_nNumFace; i++) // 각 face의 normal
	{
		Vector4 v1, v2;

		v1.x = m_vertex[m_face[i].m_vertex[0] - 1][0] - m_vertex[m_face[i].m_vertex[1] - 1][0];
		v1.y = m_vertex[m_face[i].m_vertex[0] - 1][1] - m_vertex[m_face[i].m_vertex[1] - 1][1];

		v1.z = m_vertex[m_face[i].m_vertex[0] - 1][2] - m_vertex[m_face[i].m_vertex[1] - 1][2];

		v2.x = m_vertex[m_face[i].m_vertex[1] - 1][0] - m_vertex[m_face[i].m_vertex[2] - 1][0];
		v2.y = m_vertex[m_face[i].m_vertex[1] - 1][1] - m_vertex[m_face[i].m_vertex[2] - 1][1];

		v2.z = m_vertex[m_face[i].m_vertex[1] - 1][2] - m_vertex[m_face[i].m_vertex[2] - 1][2];

		Vector4 ret;

		ret = crossProduct(v1, v2);
		ret = normalize(ret);

		m_faceNormal[i][0] = ret.x;
		m_faceNormal[i][1] = ret.y;
		m_faceNormal[i][2] = ret.z;
	}

}

void Renderer::makeVertexNormals()
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
```

<br>

광원을 지정하고 빛벡터를 생성한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/98e56f43-4a16-4236-88a9-ef209e445176" alt width="90%">
<em>디퓨즈의 빛의 강도 계산</em>
</center>

<br>

```c++
Vector4 directionLight(5.f, 5.f, 5.f);
Vector4 ndirectionLight = normalize(directionLight);
```

엣지 테이블에 노말을 추가하고 빛의 강도에 따른 계산을 진행하여 라이팅을 적용해 렌더링한다.

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/9440102b-e610-44de-8804-6cec43026f79" alt width="80%">
<em>식을 결합, 렌더링</em>
</center>


<br>

```c++
void Renderer::buildEdgetable(int nFace)
{
	float vertices[2][3] = { 0 };
	float uvVertices[2][2] = { 0 };
	float nVertices[2][3] = { 0 };
	float lightVertices[2][3] = { 0 };

	int ymin = 0;
	float savedY = 0;
	float yMax = 0;

	for (int i = 0; i < m_face[nFace].m_nNumVertex; i++)
	{
		for (int j = 0; j < 3; j++)
		{
			vertices[0][j] = m_tramsformedVertex[m_face[nFace].m_vertex[i] - 1][j];
			vertices[1][j] = m_tramsformedVertex[m_face[nFace].m_vertex[(i + 1) % m_face[nFace].m_nNumVertex] - 1][j];
			nVertices[0][j] = m_faceNormal[nFace][j];
			nVertices[1][j] = m_faceNormal[nFace][j];

		}

		for (int j = 0; j < 2; j++)
		{
			uvVertices[0][j] = m_uv[m_face[nFace].m_vertex[i] - 1][j];
			uvVertices[1][j] = m_uv[m_face[nFace].m_vertex[(i + 1) % m_face[nFace].m_nNumVertex] - 1][j];
		}

		if (vertices[0][1] == vertices[1][1]) continue;
		
		if (vertices[0][1] < vertices[1][1]) // 작을때 ceiling 클 때 floor 
		{
			savedY = vertices[0][1];
			ymin = ceil(vertices[0][1]);
			ymin = max(ymin, 0);

			if (ymin > checkImageHeight - 1) continue;

			m_ET[ymin][m_indexCount[ymin]].xperY = (vertices[1][0] - vertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].zPerY = (vertices[1][2] - vertices[0][2]) / (vertices[1][1] - vertices[0][1]);

			m_ET[ymin][m_indexCount[ymin]].uPerY = (uvVertices[1][0] - uvVertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].vPerY = (uvVertices[1][1] - uvVertices[0][1]) / (vertices[1][1] - vertices[0][1]);

			m_ET[ymin][m_indexCount[ymin]].nxPerY = (nVertices[1][0] - nVertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].nyPerY = (nVertices[1][1] - nVertices[0][1]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].nzPerY = (nVertices[1][2] - nVertices[0][2]) / (vertices[1][1] - vertices[0][1]);

			m_ET[ymin][m_indexCount[ymin]].yMax = vertices[1][1];
			m_ET[ymin][m_indexCount[ymin]].x = vertices[0][0];
			m_ET[ymin][m_indexCount[ymin]].z = vertices[0][2];

			m_ET[ymin][m_indexCount[ymin]].u = uvVertices[0][0];
			m_ET[ymin][m_indexCount[ymin]].v = uvVertices[0][1];

			m_ET[ymin][m_indexCount[ymin]].nx = nVertices[0][0];
			m_ET[ymin][m_indexCount[ymin]].ny = nVertices[0][1];
			m_ET[ymin][m_indexCount[ymin]].nz = nVertices[0][2];
	
			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].xperY;
				m_ET[ymin][m_indexCount[ymin]].z += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].zPerY;

				m_ET[ymin][m_indexCount[ymin]].u += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].uPerY;
				m_ET[ymin][m_indexCount[ymin]].v += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].vPerY;

				m_ET[ymin][m_indexCount[ymin]].nx += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nxPerY;
				m_ET[ymin][m_indexCount[ymin]].ny += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nyPerY;
				m_ET[ymin][m_indexCount[ymin]].nz += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nzPerY;
			}
		}

		else
		{
			savedY = vertices[1][1];
			ymin = ceil(vertices[1][1]);
			ymin = max(ymin, 0);

			if (ymin > checkImageHeight - 1) continue;

			m_ET[ymin][m_indexCount[ymin]].xperY = (vertices[1][0] - vertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].zPerY = (vertices[1][2] - vertices[0][2]) / (vertices[1][1] - vertices[0][1]);

			m_ET[ymin][m_indexCount[ymin]].uPerY = (uvVertices[1][0] - uvVertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].vPerY = (uvVertices[1][1] - uvVertices[0][1]) / (vertices[1][1] - vertices[0][1]);

			m_ET[ymin][m_indexCount[ymin]].nxPerY = (nVertices[1][0] - nVertices[0][0]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].nyPerY = (nVertices[1][1] - nVertices[0][1]) / (vertices[1][1] - vertices[0][1]);
			m_ET[ymin][m_indexCount[ymin]].nzPerY = (nVertices[1][2] - nVertices[0][2]) / (vertices[1][1] - vertices[0][1]);


			m_ET[ymin][m_indexCount[ymin]].yMax = vertices[0][1];
			m_ET[ymin][m_indexCount[ymin]].x = vertices[1][0];
			m_ET[ymin][m_indexCount[ymin]].z = vertices[1][2];
			
			m_ET[ymin][m_indexCount[ymin]].u = uvVertices[1][0];
			m_ET[ymin][m_indexCount[ymin]].v = uvVertices[1][1];
			
			m_ET[ymin][m_indexCount[ymin]].nx = nVertices[1][0];
			m_ET[ymin][m_indexCount[ymin]].ny = nVertices[1][1];
			m_ET[ymin][m_indexCount[ymin]].nz = nVertices[1][2];

			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].xperY;
				m_ET[ymin][m_indexCount[ymin]].z += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].zPerY;

				m_ET[ymin][m_indexCount[ymin]].u += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].uPerY;
				m_ET[ymin][m_indexCount[ymin]].v += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].vPerY;

				m_ET[ymin][m_indexCount[ymin]].nx += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nxPerY;
				m_ET[ymin][m_indexCount[ymin]].ny += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nyPerY;
				m_ET[ymin][m_indexCount[ymin]].nz += (ymin - savedY) * m_ET[ymin][m_indexCount[ymin]].nzPerY;
			}
		}
		m_indexCount[ymin]++;
	}

void Renderer::fill(GLubyte color[3])
{
	// AET
	for (int i = 0; i < checkImageHeight; i++)
	{
		//update intersection
		for (int j = 0; j < m_numEdgeInAET; j++)
		{
			m_AET[j].x += m_AET[j].xperY;
			m_AET[j].z += m_AET[j].zPerY;

			m_AET[j].u += m_AET[j].uPerY;
			m_AET[j].v += m_AET[j].vPerY;

			m_AET[j].nx += m_AET[j].nxPerY;
			m_AET[j].ny += m_AET[j].nyPerY;
			m_AET[j].nz += m_AET[j].nzPerY;
		}

		//Add new edge
		for (int j = 0; j < m_indexCount[i]; j++)
		{
			m_AET[m_numEdgeInAET + j] = m_ET[i][j];
		}
		m_numEdgeInAET += m_indexCount[i];

		//Delete edge
		for (int j = 0; j < m_numEdgeInAET; j++)
		{
			if (m_AET[j].yMax < i)
			{
				for (int k = j; k < m_numEdgeInAET; k++)
				{
					m_AET[k] = m_AET[k + 1];
				}
				j--;
				m_numEdgeInAET--;
			}
		}

		//Sort intersections
		Edge temp;
		for (int j = 0; j < m_numEdgeInAET - 1; j++)
		{
			for (int k = j + 1; k < m_numEdgeInAET; k++)
			{
				if (m_AET[j].x > m_AET[k].x)
				{
					temp = m_AET[j];
					m_AET[j] = m_AET[k];
					m_AET[k] = temp;
				}
			}
		}

		//Render
		for (int j = 0; j < m_numEdgeInAET; j += 2)
		{
			int k;
			int xmin = floor(m_AET[j].x);
			int xmax = floor(m_AET[j + 1].x);
			xmin = max(xmin, 0);
			xmax = min(xmax, checkImageWidth - 1);

			float deltaU = 0;	float deltaV = 0;	float deltaZ = 0;
			float deltaNX = 0;	float deltaNY = 0;	float deltaNZ = 0;
			float uPerX = 0;	float vPerX = 0;	float zPerX = 0;
			float nxPerX = 0;	float nyPerX = 0;	float nzPerX = 0;

			float lxPerX = 0;	float lyPerNY = 0;	float lzPerNZ = 0;
			float deltaLX = 0;	float deltaLY = 0;	float deltaLZ = 0;

			float sx, sy, sz; float scala = 0;

			uPerX = (m_AET[j + 1].u - m_AET[j].u) / (m_AET[j + 1].x - m_AET[j].x);
			vPerX = (m_AET[j + 1].v - m_AET[j].v) / (m_AET[j + 1].x - m_AET[j].x);
			zPerX = (m_AET[j + 1].z - m_AET[j].z) / (m_AET[j + 1].x - m_AET[j].x);

			nxPerX = (m_AET[j + 1].nx - m_AET[j].nx) / (m_AET[j + 1].x - m_AET[j].x);
			nyPerX = (m_AET[j + 1].ny - m_AET[j].ny) / (m_AET[j + 1].x - m_AET[j].x);
			nzPerX = (m_AET[j + 1].nz - m_AET[j].nz) / (m_AET[j + 1].x - m_AET[j].x);

			for (k = xmin; k < xmax; k++)
			{
				sx = (m_AET[j].nx + deltaNX) * (ndirectionLight.x);
				sy = (m_AET[j].ny + deltaNY) * (ndirectionLight.y);
				sz = (m_AET[j].nz + deltaNZ) * (ndirectionLight.z);

				// 빛의 영향을 계산을 위한 스칼라
				scala = sx + sy + sz;

				if (m_AET[j].z + deltaZ < m_zBuffer[i][k])
				{
					checkImage[i][k][0] = (GLubyte)m_texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaU)][0] * max(scala, 0.f);
					checkImage[i][k][1] = (GLubyte)m_texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaU)][1] * max(scala, 0.f);
					checkImage[i][k][2] = (GLubyte)m_texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaU)][2] * max(scala, 0.f);

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
	}
}

```

<br>

## 출력
___


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7d0c46e8-b43e-4a26-8f29-02ce4be8edcd" alt width=600>
<em>라이팅을 적용한 이미지 출력 </em>
</center>

<br>
<br>
