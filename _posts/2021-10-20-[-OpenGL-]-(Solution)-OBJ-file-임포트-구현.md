---
categories:
  - OpenGL
tags:
  - OpenGL
  - programming
  - cpp
  - solution
---
# 문제
___
<br>
기존 msh 파일만 읽어 오는 임포트 코드를 OBJ 파일을 읽어올 수 있게 코드 수정 시도

```
# cube.obj
mtllib cube.mtl
o cube

v -0.500000 -0.500000 0.500000
v 0.500000 -0.500000 0.500000
v -0.500000 0.500000 0.500000
v 0.500000 0.500000 0.500000
v -0.500000 0.500000 -0.500000
v 0.500000 0.500000 -0.500000
v -0.500000 -0.500000 -0.500000
v 0.500000 -0.500000 -0.500000

vt 0.000000 0.000000
vt 1.000000 0.000000
vt 0.000000 1.000000
vt 1.000000 1.000000

vn 0.000000 0.000000 1.000000
vn 0.000000 1.000000 0.000000
vn 0.000000 0.000000 -1.000000
vn 0.000000 -1.000000 0.000000
vn 1.000000 0.000000 0.000000
vn -1.000000 0.000000 0.000000

g cube
usemtl cube
s 1
f 1/1/1 2/2/1 3/3/1
f 3/3/1 2/2/1 4/4/1
s 2
f 3/1/2 4/2/2 5/3/2
f 5/3/2 4/2/2 6/4/2
s 3
f 5/4/3 6/3/3 7/2/3
f 7/2/3 6/3/3 8/1/3
s 4
f 7/1/4 8/2/4 1/3/4
f 1/3/4 8/2/4 2/4/4
s 5
f 2/1/5 8/2/5 4/3/5
f 4/3/5 8/2/5 6/4/5
s 6
f 7/1/6 1/2/6 5/3/6
f 5/3/6 1/2/6 3/4/6

```

<br>

obj 파일 파싱을 통한 임포트 수도 코드 

```c++


void Object::readOBJFile(char* pFileName)
{
	//읽을 목적의 파일 선언
	ifstream readFile(pFileName);
	
	//파일이 열렸는지 확인
	if (readFile.is_open())
	{
		//파일 끝까지 읽었는지 확인
		while (!readFile.eof())
		{
			readFile.getline(text, 256);
			line.push_back(text);
		}
		
		//한줄씩 읽어오기
		for (int i = 0; i < line.size(); i++)
		{
			if(문장이 'v'라면)
			{
				첫 번째 수->m_vertex[vertexNum][0];
				두 번째 수->m_vertex[vertexNum][1];
				세 번째 수->m_vertex[vertexNum][2];
				vertexNum++;
			}
			
			else if(문장이 'vt'라면)
			{
				첫 번째 수->m_texture[textureNum][0];
				두 번째 수->m_texture[textureNum][1];
				세 번째 수->m_texture[textureNum][2];
				textureNum++;
			}
			
			else if(문장이 'vn'라면)
			{
				첫 번째 수->m_vertexNormal[normalNum][0];
				두 번째 수->m_vertexNormal[normalNum][1];
				세 번째 수->m_vertexNormal[normalNum][2];
				normalNum++;
			}
			
			else if(문장이 'f'라면)
			{
				if('/'포함)
				{
					첫 번째 수->m_face[faceNum][0];
					두 번째 수->m_face_texture[faceNum][0];
					세 번째 수->m_face_normal[faceNum][0];
					
					첫 번째 수->m_face[faceNum][1];
					두 번째 수->m_face_texture[faceNum][1];
					세 번째 수->m_face_normal[faceNum][1];
					
					첫 번째 수->m_face[faceNum][2];
					두 번째 수->m_face_texture[faceNum][2];
					세 번째 수->m_face_normal[faceNum][2];
					faceNum++;
				}
			
				else
				{
					첫 번째 수->m_face[faceNum][0];
					두 번째 수->m_face[faceNum][1];
					세 번째 수->m_face[faceNum][2];
					faceNum++;
				}
			}
		}

	}
	readFile.close();
}



```

<br>


그러나 OBJ File Loader의 face 중 4개 이상의 vertex들을 가진 face의 경우 임포트 에러 발생, 구체 등의 물체는 하나의 face가 3개 이상을 가진 경우가 많으므로 코드 수정

<br>


# 해결
___

하나의 face를 여러 삼각형으로 나눈다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/29b1a4d5-2201-4fa2-9d1a-0e3cff3d40fd" alt width="100%">
<em>코드 구현</em>
</center>

<br>
<br>


```c++
void Object::readOBJFile(char* pFileName)
{
	//ifstream readFile("dodecahedron.txt");

	ifstream readFile(pFileName);
	int faceVertex[20];
	int UV;
	int nomal;
	char text[256];
	vector<string>line;



	if (readFile.is_open())
	{
		while (!readFile.eof())
		{
			readFile.getline(text, 256);
			line.push_back(text);
		}

		for (int i = 0; i < line.size(); i++)
		{
			if (line[i][0] == 'v' && line[i][1] == ' ')
			{
				sscanf_s(line[i].c_str(), "v %f %f %f", &m_vertex[m_nNumVertex][0], &m_vertex[m_nNumVertex][1], &m_vertex[m_nNumVertex][2]);
				m_nNumVertex++;
			}

			else if (line[i][0] == 'v' && line[i][1] == 't')
			{
				sscanf_s(line[i].c_str(), "vt %f %f", &m_uv[m_nNumTexture][0], &m_uv[m_nNumTexture][1]);
				m_nNumTexture++;
				haveUV = true;
			}

			else if (line[i][0] == 'v' && line[i][1] == 'n')
			{
				sscanf_s(line[i].c_str(), "vn %f %f %f", &m_vertexNormal[m_nNumNormal][0], &m_vertexNormal[m_nNumNormal][1], &m_vertexNormal[m_nNumNormal][2]);
				m_nNumNormal++;
				haveNormal = true;
			}

			else if (i < line.size() && line[i][0] == 'f')
			{
				int j = 0, numVertex = 0;

				for (; j < line[i].size(); j++)
				{
					if (line[i][j] == ' ')
					{
						numVertex++;
					}
				}

				if (line[i][j - 1] == ' ')
				{
					numVertex--;
				}

				m_face[m_nNumFace].m_nNumVertex = numVertex;
				int faceVerNum = 0, faceVerNorNum = 0, faceUVNum = 0;

				if (line[i][1] == ' ' && haveUV && haveNormal)
				{

					for (int j = 1; j < line[i].size(); j++)
					{
						if (line[i][j] == ' ') {
							sscanf_s(&line[i][j + 1], "%d/%d/%d", &faceVertex[faceVerNum], &m_uv, &m_faceNormal);
							faceVerNum++;
						}
					}
					for (int j = 0; j < m_face[m_nNumFace].m_nNumVertex; j++)
					{
						m_face[m_nNumFace].m_vertex[j] = faceVertex[j];
					}
				}

				else if (line[i][1] == ' ' && !haveUV && haveNormal)
				{
					for (int j = 1; j < line[i].size(); j++)
					{
						if (line[i][j] == ' ')
						{
							sscanf_s(&line[i][j + 1], "%d//%d", &faceVertex[faceVerNum], &m_faceNormal);
							faceVerNum++;
						}
					}
					for (int j = 0; j < m_face[m_nNumFace].m_nNumVertex; j++)
					{
						m_face[m_nNumFace].m_vertex[j] = faceVertex[j];
					}
				}

				else
				{
					for (int j = 1; j < line[i].size(); j++)
					{
						if (line[i][j] == ' ')
						{
							sscanf_s(&line[i][j + 1], "%d", &faceVertex[faceVerNum]);
							faceVerNum++;
						}
					}
					for (int j = 0; j < m_face[m_nNumFace].m_nNumVertex; j++)
					{
						m_face[m_nNumFace].m_vertex[j] = faceVertex[j];
					}
				}

				m_nNumFace++;
			}

		}

	}
	readFile.close();
}

```


<br>
<br>

