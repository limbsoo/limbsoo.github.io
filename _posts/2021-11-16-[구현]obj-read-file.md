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

기존 msh 파일은 정점에 관한 정보만 있는 파일로, 텍스처 정보 등이 포함되어있지 않다.

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


대표적으로 사용하는 obj는 이러한 정보가 포함되어 있어, 이를 임포트하고 그정보로 텍스처링된 모델을 만들 수 있으나, 기존 임포트 코드 수정이 필요하다.

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


```C++
void Object::readOBJFile(char* pFileName)
{
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

