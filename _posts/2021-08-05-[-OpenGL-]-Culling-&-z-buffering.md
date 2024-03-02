---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
---

# Culling & z-buffering
___

## 1. Culling


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/76338019-52ba-4594-8636-d35fc13e14d0" alt width="100%">
<em></em>
</center>

<br>

```c++

bool Renderer::isBackFace(int nFace)
{
	float normalZ = 0;
	Vector4 v1, v2, v3;
	v1.x = m_tramsformedVertex[m_face[nFace].m_vertex[0] - 1][0];
	v1.y = m_tramsformedVertex[m_face[nFace].m_vertex[0] - 1][1];
	v2.x = m_tramsformedVertex[m_face[nFace].m_vertex[1] - 1][0];
	v2.y = m_tramsformedVertex[m_face[nFace].m_vertex[1] - 1][1];
	v3.x = m_tramsformedVertex[m_face[nFace].m_vertex[2] - 1][0];
	v3.y = m_tramsformedVertex[m_face[nFace].m_vertex[2] - 1][1];
	normalZ = ((v2.x - v1.x) * (v3.y - v1.y)) - ((v2.y - v1.y) * (v3.x - v1.x));

	if (normalZ < 0) return true;
	return false;
}



void Renderer::render()
{
	for (int i = 0; i < m_nNumFace; i++)
	{
		clearEdgetable();
		if (!isBackFace(i))
		{
			buildEdgetable(i);
			fill(m_face[i].m_color);
		}
	}
}
```

<br>

## 2. Z-버퍼링

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/b5f3d5e9-4ec7-4250-8653-12caeb755a4b" alt width="100%">
<em>z - 버퍼링</em>
</center>

<br>

```c++
void Renderer::clearZBuffer()
{
	for (int i = 0; i < checkImageHeight; i++)
	{
		for (int j = 0; j < checkImageWidth; j++)
		{
			m_zBuffer[i][j] = 1;
		}
	}
}

void Renderer::render()
{
	for (int i = 0; i < m_nNumFace; i++)
	{
		clearEdgetable();
		clearZBuffer();
		
		if (!isBackFace(i))
		{
			buildEdgetable(i);
			fill(m_face[i].m_color);
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
<br>
