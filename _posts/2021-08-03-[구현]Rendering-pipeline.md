---
categories:
  - 컴퓨터 그래픽스
tags:
  - OpenGL
  - 컴퓨터그래픽스
  - 구현
  -  C++
---
## 구현
___

<br>
렌더링 파이프 라인을 openGL 환경에서 엣지 테이블(edge table)을 통한 스캔라인(scanline)기법을 통해 구현하였다.
<br>
## 1. Import


```  
# model.msh

# 점의 개수
$Vertex           6  
# 면의 개수  
$Faces            2   
# 점번호 x y z 
Vertex         1   0 0 0  
Vertex         2   639 479 0
Vertex         3   0 479 0
Vertex         4   300.5 70.4 0 
Vertex         5   500.8 50.5 0
Vertex         6   400.8 450.1 0
# 면번호 r g b “점의 개수” “면을 이루는 점들”(점의 개수는 유동적)..
Face 1 255 0 0 4 4 5 6 1
Face 2 0 0 255 3 1 2 3 
```

2차원 물체의 정점 정보가 저장된 msh 파일을 readFile 함수를 통해 임포트(import)하였다.

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

		ASSERT(stricmp(buff2, "Vertex") == 0);
		ASSERT((nNum - 1) == i);
	}

	for (i = 0; i < m_nNumFace; i++)
	{
		int nNum;
		int nCurrPos;

		getNewLine(buff, 512, pFile);
		sscanf(buff, "%s %d %d %d %d %d", buff2, &nNum, &m_face[i].m_color[0], &m_face[i].m_color[1], &m_face[i].m_color[2], &m_face[i].m_nNumVertex);
		ASSERT(stricmp(buff2, "Face") == 0);
		ASSERT((nNum - 1) == i);
		nCurrPos = 0;
		for (j = 0; j < 5; j++)
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

void Renderer::getNewLine(char* buff, int count, FILE* pFile)
{
	fgets(buff, count, pFile);
	while (buff[0] == '#') VERIFY(fgets(buff, count, pFile));
}

```


## 2. Main 함수

임포트한 정점 정보들을 사용하여 렌더링 하기 위해 openGL의 **glut** 라이브러리를 사용, 렌더링 환경을 구성하였다.


```c++  
#main.cpp

#include "glut.h"

int main(int argc, char** argv)
{
	g_renderer.readFile("model.msh");
	// GLUT라이브러리를 초기화하고 기반 플랫폼의 윈도우 시스템과 연결
	glutInit(&argc, argv);
	//단일버퍼(single buffer)의 RGB 색상 모드를 갖는 창으로 설정
	glutInitDisplayMode(GLUT_SINGLE | GLUT_RGBA);
	// 렌더링할 윈도우 크기 설정(width, height)
	glutInitWindowSize(640, 480);
	//윈도우 위치 설정(x, y)
	glutInitWindowPosition(100, 100);
	//윈도우 생성
	glutCreateWindow(argv[0]);
	//렌더링 처리에 사용할 함수 'display'를 통해 렌더링
	glutDisplayFunc(display);
	//윈도우의 크기를 변경할 때 인자로 주어진 함수를 호출
	glutReshapeFunc(reshape);
	//이벤트 처리 루프 생성, 다음 이벤트 입력까지 대기
	glutMainLoop();
	
	return 0;
}
}```

## 3. 렌더링

glut에서 제공하는 함수를 통해 쉽게 이를 렌더링 할 수 있으나, 렌더링 시 정점 정보 처리에 대한 이해를 위해 이를 엣지 테이블을 통한 스캔라인 기법을 구현, 사용하였다.

```c++  
void display(void)
{
	glClear(GL_COLOR_BUFFER_BIT);
	glBegin(GL_TRIANGLES);
	glVertex3f(-0.5,-0.5,0.0);
	glVertex3f(0.5,0.0,0.0);
	glVertex3f(0.0,0.5,0.0);
	glEnd();
	glFlush();
}
}```

```c++  
void makeCheckImage(void)
{
	g_renderer.render();
}

void display(void)
{
	//버퍼 초기화
	glClear(GL_COLOR_BUFFER_BIT);
	makeCheckImage();
	//픽셀 작업의 레스터 위치 지정
	glRasterPos2i(0, 480);
	//픽셀 확대/축소 요소를 지정
	glPixelZoom(1.f, -1.f);
	glDrawPixels(checkImageWidth, checkImageHeight, GL_RGB, GL_UNSIGNED_BYTE, g_renderer.checkImage);
	//openGL 함수 즉시 실행
	glFlush();
}

}```




모델을 화면에 렌더링 시, 레스터 변환을 사용한다. 레스터 변환은 정점 좌표를 화면 좌표로 변환하는 과정으로, 모델의 정점이 화면의 어디에 위치할 지를 정한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e9b2df32-8b94-439a-b96d-d4a2f749084f" alt width="80%">
<em>레스터 변환</em>
</center>

이러한 레스터 변환 기법에는 몇 가지가 존재하는데, 그 중에서 DDA 변환은 기울기를 기준으로 샘플링(sampling)하는 방법이다. 그러나 부동 소수 연산으로 정수 연산에 비해 느리고, 반올림 연산, 연산 결과의 정확도가 떨어지는 문제가 있다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/897944a3-ad3d-47fa-afc9-79719dba63e1" alt width="100%">
<em>DDA 알고리즘</em>
</center>
 
 브래스넘 알고리즘은 픽셀 중심과 선분 간의 수직 거리에 의한 판단하여 정수 연산이며 선분 생성 알고리즘과 유사한 알고리즘이다. 본 글은 브래스넘 알고리즘에 따른 스캔라인을 구현하였다.
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a4a20958-b3b9-413a-8f41-442fc3a01aea" alt width="100%">
<em>브래스넘 알고리즘</em>
</center>


브래스넘 알고리즘은 기울기에 따른 상하 판단을 진행하고 내부를 판단한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/c754920c-35d4-4534-aa83-0957a7cc7239" alt width="80%">
<em>상하 판단</em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/6d7f4fce-31c0-4ecb-a393-9c49af7023fd" alt width="50%">
<em>내부 판단</em>
</center>
 
 홀수 번째 교차 화소부터 짝수 번째 교차 화소 직전까지 채우고, 길이 보존을 위해 짝수 번째 포함하지 않는다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a523eaba-c2bf-477e-bd56-8c1256df809f" alt width="100%">
<em>Fill 알고리즘</em>
</center>


그리고 이러한 정보를 ET(edge table)과 AET(active edge table)를 활용하여 픽셀을 채운다. ET는 화면과 수평이 아닌 변들의 정보들을 저장,각 변(edge)은 그 변의 가장 작은 y값에 해당하는 스캔라인에 저장된다. AET는 현재 스캔라인과 교차하는 변들을 저장, 교차점의 x값에 따라 정렬해서 저장한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8ae09455-d3d3-49c8-93aa-6cd41e9855c4" alt width="100%">
<em>ET</em>
</center>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/f591c845-8b33-4ec6-b111-0e3af8707060" alt width="100%">
<em>AET</em>
</center>

이를 구현했을 때 아래와 같은 코드를 작성하였다.

```c++  
# renderer.h

class  Edge
{
public:
	float yMax;
	float x;
	float inverseOfSlope;
	float z;
};

class Renderer 
{
public:
	Renderer() : m_nNumVertex(0), m_nNumFace(0){}
	int m_nNumVertex;
	int m_nNumFace;
	float m_vertex[NUM_MAX_VERTEX][3];
	Face_t m_face[NUM_MAX_FACE];
	Edge m_ET[checkImageHeight][200];
	Edge m_AET[200];
	int m_numEdgeInAET;
	int m_indexCount[checkImageHeight];

};
```


```c++  

# renderer.cpp

void Renderer::render()
{
	for (int i = 0; i < m_nNumFace; i++)
	{
		clear();
		buildEdgetable(i);
		fill(m_face[i].m_color);
	}
}

void Renderer::clear()
{
	//ET 초기화
	for (int i = 0; i < checkImageHeight; i++)
	{
		for (int j = 0; j < 200; j++)
		{
			m_ET[i][j].inverseOfSlope = 0;
			m_ET[i][j].x = 0;
			m_ET[i][j].yMax = 0;
			m_indexCount[i] = 0;
		}
	}

	m_numEdgeInAET = 0;
}

void Renderer::buildEdgetable(int nFace)
{
	float vertices[2][3];
	int ymin;
	float flooringNum;
	float yMax, x, inverseOfSlope,z;

	for (int i = 0; i < m_face[nFace].m_nNumVertex; i++)
	{
		for (int j = 0; j < 3; j++)
		{
			vertices[0][j] = m_vertex[m_face[nFace].m_vertex[i]-1][j];
			vertices[1][j] = m_vertex[m_face[nFace].m_vertex[(i + 1) % m_face[nFace].m_nNumVertex]-1][j];
		}

		if (vertices[0][1] == vertices[1][1]) continue;

		else
			inverseOfSlope = (vertices[1][0] - vertices[0][0]) / (vertices[1][1] - vertices[0][1]);

		float savedY;

		if (vertices[0][1] < vertices[1][1]) // 작을때 ceiling 클 때 floor 
		{
			savedY = vertices[0][1];
			if ((int)vertices[0][1] == vertices[0][1])
			{
				vertices[0][1] += 1;
			}
			
			ymin = ceil(vertices[0][1]);
			ymin = max(ymin, 0);
			ymin = min(ymin, checkImageWidth);
			m_ET[ymin][m_indexCount[ymin]].x = vertices[0][0];

			if (ymin - savedY != 0)
			{
				m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * inverseOfSlope;
			}

			m_ET[ymin][m_indexCount[ymin]].yMax = vertices[1][1];
			m_ET[ymin][m_indexCount[ymin]].inverseOfSlope = inverseOfSlope;
			m_ET[ymin][m_indexCount[ymin]].z = vertices[1][2];
			m_indexCount[ymin]++;

		}
		else
		{
			savedY = vertices[1][1];
			if ((int)vertices[1][1] == vertices[1][1])
			{
				vertices[1][1] += 1;
			}
			
			ymin = ceil(vertices[1][1]);
			ymin = max(ymin, 0);
			ymin = min(ymin, checkImageWidth);

			m_ET[ymin][m_indexCount[ymin]].x = vertices[1][0];
			m_ET[ymin][m_indexCount[ymin]].x += (ymin - savedY) * inverseOfSlope;
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
			ASSERT(0 <= j && j < 200);
		}

		//Add new edge
		for (int j = 0; j < m_indexCount[i]; j++)
		{
			m_AET[m_numEdgeInAET + j] = m_ET[i][j];
			ASSERT(0 <= m_numEdgeInAET + j && j + m_numEdgeInAET < 200);
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
					ASSERT(0 <= k && k < 200);
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
			int xmin = ceil(m_AET[j].x);
			int xmax = floor(m_AET[j + 1].x);

			if (m_AET[j].x == (int)m_AET[j].x)
			{
				xmin += 1;
			}
			
			xmin = max(xmin, 0);
			xmax = min(xmax, checkImageWidth);

			for (k = xmin; k < xmax; k++)
			{
				checkImage[i][k][0] = (GLubyte)color[0];
				checkImage[i][k][1] = (GLubyte)color[1];
				checkImage[i][k][2] = (GLubyte)color[2];
			}
		}
	}
}
}```


# 2. 출력

이를 통해 렌더링 시, 아래와 같은 결과의 이미지의 출력을 화면에서 확인할 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/f00211f8-ef6d-4d17-b5b6-264430b0a87c" alt width="100%">
<em>렌더링 결과</em>
</center>



