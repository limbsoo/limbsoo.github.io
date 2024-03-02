---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
---
# Projection transform
___

면밀한 변환 과정 관찰을 위해 임포트 모델을 2D 모델에서 3D 모델로 변경.

```
#$Vertex          2904
#$Face            5804
Vertex         1   0.151632  -0.043319   -0.08824
Vertex         2   0.163424  -0.033934   -0.08411
Vertex         3   0.163118  -0.053632  -0.080509
Vertex         4   0.176307  -0.028912  -0.075048
Vertex         5   0.174429  -0.051613  -0.073945
Vertex         6   0.186153  -0.032952  -0.063704
Vertex         7   0.189315  -0.049201  -0.055717
Vertex         8   0.173804  -0.063906  -0.070661
...

Face           1          1          2          3
Face           2          2          4          5
Face           3          5          4          6
Face           4          7          8          5
Face           5          8          9          3
Face           6          9         10         11
...
```


<br>

임포트 코드 수정

```c++
void Renderer::readFile(char* pFileName)
{
	char buff[512], buff2[512];
	int i, j;
	FILE* pFile = fopen(pFileName, "rt");
	getNewLine(buff, 512, pFile);
	sscanf(buff, "%s %d", buff2, &m_nNumVertex);
	ASSERT(stricmp(buff2, "$Vertex") == 0);
	getNewLine(buff, 512, pFile);
	sscanf(buff, "%s %d", buff2, &m_nNumFace);
	ASSERT(stricmp(buff2, "$Faces") == 0);

	for (i = 0; i < m_nNumVertex; i++)
	{
		int nNum;
		getNewLine(buff, 512, pFile);
		sscanf(buff, "%s %d %f %f %f", buff2, &nNum, &m_vertex[i][0], &m_vertex[i][1], &m_vertex[i][2]);
		sscanf(buff, "%s ", buff2);
		for (int j = 0; j < 3; j++)	//cow
		m_vertex[i][j] = (m_vertex[i][j]) ;	//cow
		ASSERT(stricmp(buff2, "Vertex") == 0);
		ASSERT((nNum - 1) == i);
	}


	for (i = 0; i < m_nNumFace; i++)
	{
		int nNum;
		int nCurrPos;
		getNewLine(buff, 512, pFile);
		sscanf(buff, "%s %d", buff2, &nNum);		//cow
		m_face[i].m_nNumVertex = 3;					//cow
		m_face[i].m_color[0] = (GLubyte)0;			//cow
		m_face[i].m_color[1] = (GLubyte)0;			//cow
		m_face[i].m_color[2] = (GLubyte)255;		//cow

		ASSERT(stricmp(buff2, "Face") == 0);
		ASSERT((nNum - 1) == i);
		nCurrPos = 0;
		for (j = 0; j < 1; j++)						//cow
		{
			nCurrPos += strcspn(buff + nCurrPos, " \t");
			nCurrPos += strspn(buff + nCurrPos, " \t");
		}
		for (j = 0; j < m_face[i].m_nNumVertex; j++)
		{
			nCurrPos += strcspn(buff + nCurrPos, " \t");
			nCurrPos += strspn(buff + nCurrPos, " \t");
			sscanf(buff + nCurrPos, "%d", &m_face[i].m_vertex[j]);
		}
	}
}

```

<br>


Z-buffering을 통해 출력할 프래그먼트 선택

```c++
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
		
		if (vertices[0][1] != vertices[1][1])
		{
			inverseOfSlope = (vertices[1][0] - vertices[0][0]) / (vertices[1][1] - vertices[0][1]); //xperY
		}

		zPerY = (vertices[1][2] - vertices[0][2]) / (vertices[1][1] - vertices[0][1]);

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
			float zPerX = (m_AET[j + 1].z - m_AET[j].z) / (xmax - xmin);
			float deltaZ = 0;
		
			for (k = xmin; k < xmax; k++)
			{
				if (m_AET[j].z + deltaZ < zBuffer[i][k])
				{
					checkImage[i][k][0] = (GLubyte)color[0];
					checkImage[i][k][1] = (GLubyte)color[1];
					checkImage[i][k][2] = (GLubyte)color[2];
					zBuffer[i][k] = m_AET[j].z + deltaZ;
					deltaZ += zPerX;
				}
			}
		}
	}
}
```

<br>


NDC 구현

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/b340a397-d162-48fc-b79b-af64eed78572" alt width="100%">
<em>NDC </em>
</center>

<br>

```c++
 Matrix4 projectMatrix(float fovy, float aspect, float n, float f)
 {
     Matrix4 ret;
     for (int i = 0; i < 4; i++)
     {
         for (int j = 0; j < 4; j++)
         {
             ret.arr[i][j] = 0;
         }
     }
     ret.arr[0][0] = (1 / tan(fovy / 2 * PI / 180)) / aspect;
     ret.arr[1][1] = 1 / tan(fovy / 2 * PI / 180);
     ret.arr[2][2] = - ((f + n) / (f - n));
     ret.arr[2][3] = - ((2 * (n * f)) / (f - n));
     ret.arr[3][2] = -1;
     return ret;
 }
```

<br>

```c++
g_renderer.m_proj = projectMatrix(90.0f, 1.3333f, 1.0f, 150.0f);
...

Matrix4 mvp = m_proj * m_view * m_world;
```

<br>

# 출력
___

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/06d58600-2b7a-48f9-9089-a1554847340e" alt width="100%">
<em>Cow 모델 임포트 & projection 적용 </em>
</center>




