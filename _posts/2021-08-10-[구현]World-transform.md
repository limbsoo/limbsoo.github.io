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

행렬과 벡터를 통한 transform 구현
<br>
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ea9e951b-787c-4417-b011-c741aa58546e" alt width="100%">
<em>월드 변환 </em>
</center>
<br>
## 1. 행렬, 벡터
<br>
행렬, 벡터 구현

```c++  
class Matrix4 
{
public:
    float arr[4][4];
};
```

```c++  
class Vector4
{
public:
    float x, y, z, w;

    Vector4(float v[4])
    {
        x = v[0];
        y = v[1];
        z = v[2];
        w = v[3];
    }
};
```


전치 행렬과 연산자 중복(operator overloading)을 통한 행렬곱, 행렬벡터곱 구현

```c++  
Matrix4 identityMatrix()
{
    Matrix4 m;
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            if (i == j)
            {
                m.arr[i][j] = 1;
            }
            else
            {
                m.arr[i][j] = 0;
            }
        }
    }
    return m;
}

Vector4  Matrix4::operator*(Vector4 v)
{
    Vector4 ret;
    ret.x = v.x * arr[0][0] + v.y * arr[0][1] + v.z * arr[0][2] + v.w * arr[0][3];
    ret.y = v.x * arr[1][0] + v.y * arr[1][1] + v.z * arr[1][2] + v.w * arr[1][3];
    ret.z = v.x * arr[2][0] + v.y * arr[2][1] + v.z * arr[2][2] + v.w * arr[2][3];
    ret.w = v.x * arr[3][0] + v.y * arr[3][1] + v.z * arr[3][2] + v.w * arr[3][3];
    return ret;// (ret.x, ret.y, ret.z, ret.w);
}

Matrix4 operator*(const Matrix4 m, const Matrix4 n)
{
    Matrix4 ret;

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            for (int k = 0; k < 4; k++)
            {
                ret.arr[i][j] += m.arr[i][k] * n.arr[k][j];
            }
        }
    }
    return ret;
}
```


## 2. 변환

```c++  
 Matrix4 scaleMatrix(float x, float y, float z)
 {
     Matrix4 m;
     m = identityMatrix();
     m.arr[0][0] = x;
     m.arr[1][1] = y;
     m.arr[2][2] = z;
     return m;
 }
```

```c++  
 Matrix4 translateMatrix(float x, float y, float z)
 {
     Matrix4 m;
     m = identityMatrix();
     m.arr[0][3] = x;
     m.arr[1][3] = y;
     m.arr[2][3] = z;
     return m;
 }
```

```c++  
#define PI 3.1415926;

Matrix4 xRotateMatrix(float angle)
 {
     Matrix4 m;
     m = identityMatrix();
     float c = cos(angle * PI / 180);
     float s = sin(angle * PI / 180);
     m.arr[1][1] = c;
     m.arr[1][2] = -s;
     m.arr[2][1] = s;
     m.arr[2][2] = c;
     return m;
 }

 Matrix4 yRotateMatrix(float angle)
 {
     Matrix4 m;
     m = identityMatrix();
     float c = cos(angle * PI / 180);
     float s = sin(angle * PI / 180);
     m.arr[0][0] = c;
     m.arr[0][2] = s;
     m.arr[2][0] = -s;
     m.arr[2][2] = c;
     return m;
 }

 Matrix4 zRotateMatrix(float angle)
 {
     Matrix4 m;
     m = identityMatrix();
     float c = cos(angle * PI / 180);
     float s = sin(angle * PI / 180);
     m.arr[0][0] = c;
     m.arr[0][1] = -s;
     m.arr[1][0] = s;
     m.arr[1][1] = c;
     return m;
 }
```


## 3. 렌더링

변환 발생 시 화면 픽셀 정보를 초기화하여 중복 현상을 방지한다.

```c++  
void Renderer::clearCheckImage()
{
	for (int i = 0; i < checkImageHeight; i++)
	{
		for (int j = 0; j < checkImageWidth; j++)
		{
			checkImage[i][j][0] = (GLubyte)0;
			checkImage[i][j][1] = (GLubyte)0;
			checkImage[i][j][2] = (GLubyte)0;
		}
	}
}
```

변환을 각 vertex에 전달하여 렌더한다.

```c++  

void Renderer::applyMatrix(Matrix4 m
{
	Matrix4 m1 = identityMatrix();
	m1.arr[0][0] = 320;
	m1.arr[0][3] = 320;
	m1.arr[1][1] = 240;
	m1.arr[1][3] = 240;
	
	for (int i = 0; i < m_nNumVertex; i++)
	{
		Vector4 v(m_vertex[i], 1);
		v = m * v;
		v = m1 * v;
		m_tramsformedVertex[i][0] = v.x / v.w;
		m_tramsformedVertex[i][1] = v.y / v.w;
		m_tramsformedVertex[i][2] = v.z / v.w;
	}
}

void Renderer::render()
{
	clearCheckImage();
	applyMatrix(m_world);
	
	for (int i = 0; i < m_nNumFace; i++)
	{
		clearEdgetable();
		buildEdgetable(i);
		fill(m_face[i].m_color);
	}
}
```

## 4. Main 함수

키보드 입력에 따라 일정 값에 따른 변환을 구현하였다.

```c++  
int main(int argc, char** argv)
{
	g_renderer.readFile("model.msh");
	glutInit(&argc, argv);
	glutInitDisplayMode(GLUT_SINGLE | GLUT_RGBA);
	glutInitWindowSize(640, 480);
	glutInitWindowPosition(100, 100);
	glutCreateWindow(argv[0]);
	init();
	glutDisplayFunc(display);
	glutReshapeFunc(reshape);
	//키보드 입력에 따른 변환 구현
	glutKeyboardFunc(keyboard);
	glutMainLoop();
	
	return 0;
}

void keyboard(unsigned char key, int x, int y)
{
	switch (key) {
	case 'q':
		g_renderer.scale(0.5f, 0.5f, 0.5f);
		
		g_renderer.clearGlubyte();
		//윈도우 재생 호출
		glutPostRedisplay();
		break;
	case 'Q':
		g_renderer.scale(2.0f, 2.0f, 2.0f);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'w':
		g_renderer.translate(2.0f, 0, 0);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'W':
		g_renderer.translate(-2.0f,0,0 );
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'a':
		g_renderer.translate(0, 2.0f, 0);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'A':
		g_renderer.translate(0, -2.0f, 0);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'i':
		g_renderer.rotateX(30.0f);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'o':
		g_renderer.rotateY(30.0f);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 'p':
		g_renderer.rotateZ(30.0f);
		g_renderer.clearGlubyte();
		glutPostRedisplay();
		break;
	case 27:
		exit(0);
		break;
	default:
		break;
	}
}
}```

# 2. 출력

이를 통해 렌더링 시, 아래와 같은 결과의 이미지의 출력을 화면에서 확인할 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/64dc28b7-89c9-4310-8598-0ece57e96cf8" alt width="90%">
<em>변환 구현</em>
</center>





