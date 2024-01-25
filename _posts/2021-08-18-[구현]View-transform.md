---
categories:
  - 컴퓨터 그래픽스
tags:
  - OpenGL
  - 컴퓨터그래픽스
  - 구현
  -  C++
---
# 1. 구현
<br>
변환을 사용하여 물체의 위치를 변환한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ae8df5f2-33bc-4c4b-a574-96a7458e6ade" alt width="100%">
<em>카메라 공간 </em>
</center>

## 1. 가상 카메라


```c++  
	//카메라(눈)의 위치
	Vector4 eye(5.0f, 0.0f, 5.0f);
	//카메라의 초점(참조 위치)
	Vector4 at(0.0f, 0.0f, 0.0f);
	// 카메라의 위쪽 방향 지정(x=1이면 x축으로 누워있고, y=1이면 y축을 중심으로 세워져있음
	Vector4 upVector(0.0f, 1.0f, 0.0f);
	
	Vector4 a;
	a.x = 0;
	a.y = 0;
	a.z = 1;
	a.w = 1;
	
	Vector4 forward = eye - at;
	forward = normalize(forward);

	// compute the left vector
	Vector4 left = crossProduct(upVector, forward); // cross product
	left = normalize(left);

	// recompute the orthonormal up vector
	Vector4 up = crossProduct(forward, left);    // cross product
	up = normalize(up);
```


## 2. 행렬, 벡터 기능 추가

```c++  
Vector4 input(float x, float y, float z)
{
	Vector4 ret;
	ret.x = x;
	ret.y = y;
	ret.z = z;
	return ret;
}

float operator*(Vector4 v1, Vector4 v2) //내적 구하기(스칼라 곱) 
{ 
	Vector4 ret;
	ret.x = v1.x * v2.x;
	ret.y = v1.y * v2.y;
	ret.z = v1.z * v2.z;
	return ret.x + ret.y +ret.z; 
}

float innerCos(Vector4 v1, Vector4 v2) //각도 구하기 
{ 
	float cos = (v1 * v2) / (magnitude(v1) * magnitude(v2));
	return cos; 
} 

Vector4 normalize(Vector4 v) // 정규화(단위벡터로 만들기) 
{ 
	Vector4 normal; 
	float magn = magnitude(v);
	normal.x = v.x / magn;
	normal.y = v.y / magn;
	normal.z = v.z / magn;
	return normal; 
} 

float magnitude(Vector4 v) // 벡터 크기 
{ 
	auto magnitude = sqrt(pow(v.x, 2) + pow(v.y, 2) + pow(v.z, 2)); 
	return magnitude; 
} 

static double Seta(float cos) //세타값 구하기 각도(디그리) 
{ 
	return acos(cos) * (180 / PI);
} 

static Vector4 proj(Vector4 v1, Vector4 v2) // 방향(normal) * 크기(magnitude = Proj(V1) 
{ 
	Vector4 ret; 
	ret.x = ((v1 * v2) / (v2 * v2)) * v2.x;
	ret.y = ((v1 * v2) / (v2 * v2)) * v2.y;
	ret.z = ((v1 * v2) / (v2 * v2)) * v2.z;
	return ret; 
} 
static Vector4 orthogonalProjection(Vector4 v1, Vector4 proj) // 직교화 
{
	Vector4 ortho; 
	ortho = v1 - proj;
	return ortho; 
} 

Vector4 crossProduct(Vector4 v1, Vector4 v2) //외적 
{
	Vector4 cross;
	cross.x = (v1.y * v2.z) - (v1.z * v2.y);
	//cross.y = (v1.x * v2.z - v1.z * v2.x);
	cross.y = (v1.z * v2.x) - (v1.x * v2.z);
	cross.z = (v1.x * v2.y) - (v1.y * v2.x);
	return cross;
}
```

## 3. 카메라 변환

```c++  
Matrix4 RotateMatrix(Vector4 v, float angle)
{
    Matrix4 ret = identityMatrix();
    float c = cos(angle * PI / 180);
    float s = sin(angle * PI / 180);

    ret.arr[0][0] = c + (v.x * v.x) * (1 - c);          
    ret.arr[0][1] = (v.x * v.y * (1 - c)) - v.z * s;    
    ret.arr[0][2] = (v.x * v.z * (1 - c)) + v.z * s;
    ret.arr[1][0] = (v.y * v.x * (1 - c)) + v.z * s;    
    ret.arr[1][1] = c + (v.y * v.y * (1 - c));          
    ret.arr[1][2] = (v.y * v.z * (1 - c)) - v.x * s;
    ret.arr[2][0] = (v.z * v.x * (1 - c)) - v.y * s;    
    ret.arr[2][1] = (v.z * v.y * (1 - c)) - v.x * s;    
    ret.arr[2][2] = c + (v.z * v.z * (1 - c));
    return ret;
}


 Matrix4 lookAt(Vector4 eye, Vector4 at, Vector4 upVector)
 {
     // compute the forward vector from target to eye
     Vector4 forward = eye - at;
     forward = normalize(forward); // make unit length
     // compute the left vector
     Vector4 left = crossProduct(upVector,forward); // cross product
     left = normalize(left);
     // recompute the orthonormal up vector
     Vector4 up = crossProduct(forward,left); // cross product
     up = normalize(up);

     // set rotation part, inverse rotation matrix: M^-1 = M^T for Euclidean transform
     Matrix4 R = identityMatrix();
     R.arr[0][0] = left.x; 
     R.arr[0][1] = left.y;      
     R.arr[0][2] = left.z;      
     R.arr[0][3] = -left.x * eye.x - left.y * eye.y - left.z * eye.z;
     R.arr[1][0] = up.x;          
     R.arr[1][1] = up.y;        
     R.arr[1][2] = up.z;        
     R.arr[1][3] = -up.x * eye.x - up.y * eye.y - up.z * eye.z;
     R.arr[2][0] = forward.x;     
     R.arr[2][1] = forward.y;   
     R.arr[2][2] = forward.z;   
     R.arr[2][3] = -forward.x * eye.x - forward.y * eye.y - forward.z * eye.z;
     return R;
 }

 const float degree = 3.141593f / 180.0f;

 Matrix4 xlookatRotate(float angle)
 {
     Matrix4 R = identityMatrix();

     float c = cos(angle * degree);
     float s = sin(angle * degree);
     float m1 = R.arr[0][0], m2 = R.arr[0][1],
           m5 = R.arr[1][0], m6 = R.arr[1][1],
           m9 = R.arr[2][0], m10 = R.arr[2][1],
           m13 = R.arr[3][0], m14 = R.arr[3][1];

     Matrix4 ret;
     ret.arr[0][0] = m1 * c + m2 * -s;
     ret.arr[0][1] = m1 * s + m2 * c;
     ret.arr[1][0] = m5 * c + m6 * -s;
     ret.arr[1][1] = m5 * s + m6 * c;
     ret.arr[2][0] = m9 * c + m10 * -s;
     ret.arr[2][1] = m9 * s + m10 * c;
     ret.arr[3][0] = m13 * c + m14 * -s;
     ret.arr[3][1] = m13 * s + m14 * c;
     return ret;
 }
```

## 4. 렌더링

```c++  
void Renderer::render()
{
	clearCheckImage();
	Matrix4 mv = m_view * m_world;
	applyMatrix(mv);
	for (int i = 0; i < m_nNumFace; i++)
	{
		clearEdgetable();
		buildEdgetable(i);
		fill(m_face[i].m_color);
	}
}
```

## 4. 입력

```c++
void keyboard(unsigned char key, int x, int y)
{
	switch (key) {
	case 'q':
		g_renderer.m_world = scaleMatrix(0.5f, 0.5f, 0.5f) * g_renderer.m_world;
		glutPostRedisplay();
		break;
	case 'i':
		g_renderer.m_world = xRotateMatrix(30.0f) * g_renderer.m_world;
		glutPostRedisplay();
		break;
	case 'o':
		g_renderer.m_world = yRotateMatrix(30.0f) * g_renderer.m_world;
		glutPostRedisplay();
		break;
	case 'p':
		g_renderer.m_world = zRotateMatrix(30.0f) * g_renderer.m_world; 
		glutPostRedisplay();
		break;
	case 'l':
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
	case 'w':
		seta -= 10.0f;
		eye = xRotateMatrix(seta) * zRotateMatrix(0.5f) * scaleMatrix(length, length, length) * a;
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
	case 's': 
		seta += 10.0f;
		eye = xRotateMatrix(seta) * zRotateMatrix(0.5f) * scaleMatrix(length, length, length) * a;
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
	case 'a':
		seta += 10.0f;
		eye = yRotateMatrix(seta) * xRotateMatrix(0.5f) * scaleMatrix(length, length, length) * a;
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
	case 'd':
		seta -= 10.0f;
		eye = yRotateMatrix(seta) * xRotateMatrix(0.5f) * scaleMatrix(length, length, length) * a;
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
	case 27:
		exit(0);
		break;
	default:
		break;
	}
}
```


