---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
---

# Texture mapping
___
<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/c9145e65-b6f2-48ca-a3ff-70e15bc1358e" alt width=300>
<em>speckled213.raw 텍스처 </em>
</center>

<br>


텍스처를 임포트하고 텍스처 좌표를 생성한다.

```c++
g_renderer.readTextureFile("speckled213.raw");
g_renderer.makeUVVertex();

void Renderer::readTextureFile(char* pFileName)
{
	FILE* input_file;

	char input_data[checkImageHeight][checkImageWidth][3];

	input_file = fopen(pFileName, "rb");

	fread(input_data, sizeof(char), checkImageWidth * checkImageHeight * 3, input_file);

	fclose(input_file);

	for (int i = 0; i < checkImageHeight; i++)
	{
		for (int j = 0; j < checkImageWidth; j++)
		{
			m_texture[i][j][0] = (GLubyte)input_data[i][j][0];
			m_texture[i][j][1] = (GLubyte)input_data[i][j][1];
			m_texture[i][j][2] = (GLubyte)input_data[i][j][2];
		}
	}
}

void Renderer::makeUVVertex()
{
	for (int i = 0; i < m_nNumVertex; i++)
	{
		Vector4 v(m_vertex[i], 1);
		m_uv[i][0] = (v.x + 1) * 320;
		m_uv[i][1] = (v.y + 1) * 240;
	}
}

```

<br>

엣지 테이블에 텍스처 좌표를 추가하여 각 픽셀에 대응하는 텍셀의 색상을 출력한다.

```c++
void Renderer::render()
{
	if (!texureMappingEnabled)
	{
		clearCheckImage();
		clearZBuffer();
		Matrix4 mvp = m_proj * m_view * m_world; //m_world 부터 역순으로 vector에 곱함
		applyMatrix(mvp, m_world);
		clearEdgetable();

		for (int i = 0; i < m_nNumFace; i++)
		{
			clearEdgetable();
			if (isCullEnabled)
			{
				if (!isBackFace(i))
				{
					buildEdgetable(i);
					fill(m_face[i].m_color);
				}
			}
			else
			{
				buildEdgetable(i);
				fill(m_face[i].m_color);
			}
		}
	}
}

void Renderer::buildEdgetable(int nFace)
{
	float vertices[2][3];
	int ymin;
	float yMax, x, inverseOfSlope, z, zPerY;
	
	for (int i = 0; i < m_face[nFace].m_nNumVertex; i++)
	{
		for (int j = 0; j < 3; j++)
		{
			vertices[0][j] = m_tramsformedVertex[m_face[nFace].m_vertex[i] - 1][j];
			vertices[1][j] = m_tramsformedVertex[m_face[nFace].m_vertex[(i + 1) % m_face[nFace].m_nNumVertex] - 1][j];
		}

		float uvVertices[2][2];
		float u, v, uPerY, vPerY;
		for (int j = 0; j < 2; j++)
		{
			uvVertices[0][j] = m_uv[m_face[nFace].m_vertex[i] - 1][j];
			uvVertices[1][j] = m_uv[m_face[nFace].m_vertex[(i + 1) % m_face[nFace].m_nNumVertex] - 1][j];
		}

		if (vertices[0][1] == vertices[1][1]) continue;
		else
		{
			inverseOfSlope = (vertices[1][0] - vertices[0][0]) / (vertices[1][1] - vertices[0][1]); //xperY
		}
		
		zPerY = (vertices[1][2] - vertices[0][2]) / (vertices[1][1] - vertices[0][1]);
		uPerY = (uvVertices[1][0] - uvVertices[0][0]) / (vertices[1][1] - vertices[0][1]); 
		vPerY = (uvVertices[1][1] - uvVertices[0][1]) / (vertices[1][1] - vertices[0][1]);

		float savedY;
		float savedV;

		if (vertices[0][1] < vertices[1][1]) // 작을때 ceiling 클 때 floor 
		{
			savedY = vertices[0][1];
			ymin = ceil(vertices[0][1]);
			ymin = max(ymin, 0);

			if (ymin > checkImageHeight - 1) continue;

			m_ET[ymin][m_indexCount[ymin]].x = vertices[0][0];

			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * inverseOfSlope;
			}

			m_ET[ymin][m_indexCount[ymin]].zperY = zPerY;
			m_ET[ymin][m_indexCount[ymin]].yMax = vertices[1][1];
			m_ET[ymin][m_indexCount[ymin]].inverseOfSlope = inverseOfSlope;
			m_ET[ymin][m_indexCount[ymin]].z = vertices[1][2];

			if (savedY != 0) v = ymin * uvVertices[0][1] / savedY;
			else v = 0;

			m_ET[ymin][m_indexCount[ymin]].u = uvVertices[0][0];
			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].u += (ymin - savedY) * uPerY;
			}
			m_ET[ymin][m_indexCount[ymin]].v = v;
			m_ET[ymin][m_indexCount[ymin]].uPerY = uPerY;
			m_ET[ymin][m_indexCount[ymin]].vperY = vPerY;
			m_indexCount[ymin]++;
		}
		else
		{
			savedY = vertices[1][1];
			ymin = ceil(vertices[1][1]);
			ymin = max(ymin, 0);

			if (ymin > checkImageHeight - 1) continue;
			m_ET[ymin][m_indexCount[ymin]].x = vertices[1][0];
			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * inverseOfSlope;
			}
			m_ET[ymin][m_indexCount[ymin]].yMax = vertices[0][1];  
			m_ET[ymin][m_indexCount[ymin]].inverseOfSlope = inverseOfSlope;
			m_ET[ymin][m_indexCount[ymin]].z = vertices[0][2];
			savedV = uvVertices[1][1];
			if (savedY != 0)
			{
				v = ymin * uvVertices[1][1] / savedY;
			}
			else v = 0;
			m_ET[ymin][m_indexCount[ymin]].u = uvVertices[1][0];
			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].u += (ymin - savedY) * uPerY;
			}
			m_ET[ymin][m_indexCount[ymin]].v = v;
			m_ET[ymin][m_indexCount[ymin]].uPerY = uPerY;
			m_ET[ymin][m_indexCount[ymin]].vperY = vPerY;
			m_indexCount[ymin]++;
		}
	}
}

void Renderer::fill(GLubyte color[3])
{
	// AET
	for (int i = 0; i < checkImageHeight; i++)
	{

		//update intersection
		for (int j = 0; j < m_numEdgeInAET; j++)
		{
			m_AET[j].x += m_AET[j].inverseOfSlope;
			m_AET[j].z += m_AET[j].zperY;
			m_AET[j].u += m_AET[j].uPerY;
			m_AET[j].v += m_AET[j].vperY;
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
			float uPerX = (m_AET[j + 1].u - m_AET[j].u) / (xmax - xmin);
			float deltaX = 0;
			float vPerX = (m_AET[j + 1].v - m_AET[j].v) / (xmax - xmin);
			float deltaV = 0;
			float zPerX = (m_AET[j + 1].z - m_AET[j].z) / (xmax - xmin);
			float deltaZ = 0;

			for (k = xmin; k < xmax; k++)
			{
				if (m_AET[j].z + deltaZ < zBuffer[i][k])
				{
					checkImage[i][k][0] = (GLubyte)texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaX)][0];
					checkImage[i][k][1] = (GLubyte)texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaX)][1];
					checkImage[i][k][2] = (GLubyte)texture[(int)(m_AET[j].v + deltaV)][(int)(m_AET[j].u + deltaX)][2];

					zBuffer[i][k] = m_AET[j].z + deltaZ;
					deltaX += uPerX;
					deltaV += vPerX;
					deltaZ += zPerX;
				}
			}
		}
	}
}

```

<br>


# 출력
___

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ef09cf66-c004-4a1c-8f59-4d6780f1cca9" alt width=400>
<em>이미지 텍스처링 </em>
</center>