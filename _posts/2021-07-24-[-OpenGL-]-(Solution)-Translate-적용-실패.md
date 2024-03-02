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
뷰 공간 구현 후, translate를 진행하려 했으나, 적절한 이동, 회전 등이 발생하지 않음


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8accdb6b-19a5-4c32-a601-943fc64e6ee8" alt width="100%">
<em>변환 적용 실패</em>
</center>

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d20c60b5-3742-47da-abb1-7edef2a3abd7" alt width="100%">
<em>행렬 값이 원하는 대로 적용X</em>
</center>

<br>
<br>

# 해결
___


Rotate translate
- $VRP(Y축 rotate) = rotateX(seta2) * rotateZ(seta1) * scale (l) * (1,0,0,1)
- VRP(X축 rotate) = rotateY(seta2) * rotateZ(seta1) * scale (l) * (1,0,0,1)

Rotate x와 Y를 따로 따로 놓고 보았을 때 VRP(Y축 rotate)는 Camera Positon의 Y값이 변하지 않아야 하고, VRP(X축 rotate)는 Camera Positon의 Z값이 변하지 않아야 한다.



```c++

//구현된 회전 행렬
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
```


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/621b2d1b-123b-4c19-aa5b-da7917b32283" alt width="100%">
<em>Rotate</em>
</center>

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ac86404a-32db-4e59-8c79-204f5bbe215c" alt width="100%">
<em>뷰변환</em>
</center>


<br>

행렬에 직접적으로 값 적용 시, 올바른 변환 발생

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/cb7bfce5-9e38-49d2-a491-1e3b0dbb4fc4" alt width="100%">
<em>값 적용</em>
</center>

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/bae52176-488a-4c5d-9623-4ee2ea905d7c" alt width="100%">
<em>값 적용</em>
</center>


<br>
<br>

행렬은 문제가 없으나, 적용 과정에 문제가 있음을 확인.


뷰 공간에서 회전 행렬 적용 시, 좌표 축이 아닌 임의의 단위 벡터를 회전 축으로한 회전 변환 행렬이 필요하다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/1982f838-adaa-40bc-a77d-2e8a2307b906" alt width="100%">
<em>좌표축이 아닌 임의의 단위벡터 u = (ux, uy, uz)을 회전축으로 한 회전변환 행렬</em>
</center>


<br>

따라서 뷰 공간에서의 회전 변환 행렬을 구현

```c++

Matrix4 RotateMatrix(Vector4 v, float angle)
{

    Matrix4 ret = identityMatrix();

    float c = cos(angle * PI / 180);
    float s = sin(angle * PI / 180);

    ret.arr[0][0] = c + (v.x * v.x) * (1 - c);          ret.arr[0][1] = (v.x * v.y * (1 - c)) - v.z * s;    ret.arr[0][2] = (v.x * v.z * (1 - c)) + v.z * s;
    ret.arr[1][0] = (v.y * v.x * (1 - c)) + v.z * s;    ret.arr[1][1] = c + (v.y * v.y * (1 - c));          ret.arr[1][2] = (v.y * v.z * (1 - c)) - v.x * s;
    ret.arr[2][0] = (v.z * v.x * (1 - c)) - v.y * s;    ret.arr[2][1] = (v.z * v.y * (1 - c)) - v.x * s;    ret.arr[2][2] = c + (v.z * v.z * (1 - c));

    return ret;
}

```

<br>

```c++

void keyboard(unsigned char key, int x, int y)
{
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
	float length = 5.0f;

	case 'w':
		seta -= 10.0f;
		eye = xRotateMatrix(seta) * zRotateMatrix(0.5f) * scaleMatrix(length, length, length) * a;
		g_renderer.m_view = lookAt(eye, at, upVector);
		glutPostRedisplay();
		break;
}
```

<br>
<br>

뷰 공간에서의 변환 적용 확인

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4cb3070b-6ca2-44f1-9d54-843d12c01be3" alt width="100%">
<em>변환</em>
</center>

<br>

행렬 값 또한 적절히 적용되는 것을 확인했다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7de90854-ac47-4dc0-a4b4-019514476bf0" alt width="100%">
<em>행렬 값</em>
</center>



