---
categories:
  - DX11
tags:
  - DirectX11
  - Direct3D
  - DX11
  - graphics
  - programming
  - cpp
---
# 구현
---


DX11에서 버퍼에 입력된 정점 데이터는 shader를 통해 그래픽 카드(GPU)에서 처리된다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8c448c38-ecf2-425d-8660-430aa36e671f" alt width=400>
<em>DX11 렌더링 파이프라인</em>
</center>

<br>


이를 구현하기 위해 프레임워크를 업데이트한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/861fa02e-e421-412a-ab6b-024c2789c44f" alt width=400>
<em>프레임 워크 구성</em>
</center>


<br>

shader 코드 작성

```c++
color.vs

cbuffer MatrixBuffer
{
    matrix worldMatrix;
    matrix viewMatrix;
    matrix projectionMatrix;
};

struct VertexInputType
{
    float4 position : POSITION;
    float4 color : COLOR;
};

struct PixelInputType
{
    float4 position : SV_POSITION;
    float4 color : COLOR;
};

PixelInputType ColorVertexShader(VertexInputType input)
{
    PixelInputType output;
    input.position.w = 1.0f;
    output.position = mul(input.position, worldMatrix);
    output.position = mul(output.position, viewMatrix);
    output.position = mul(output.position, projectionMatrix);
    output.color = input.color;
    
    return output;
}
```

```c++
color.ps

struct PixelInputType
{
    float4 position : SV_POSITION;
    float4 color : COLOR;
};

float4 ColorPixelShader(PixelInputType input) : SV_TARGET
{
    return input.color;
}

```


<br>


shader 코드에 정점 데이터를 전달하기 위한 버퍼를 가진 modelclass를 정의 

```c++
modelclass.h

class ModelClass
{
private:
	struct VertexType
	{
		XMFLOAT3 position;
		XMFLOAT2 texture;
		XMFLOAT3 normal;
	};

public:
	ModelClass();
	ModelClass(const ModelClass&);
	~ModelClass();

	bool Initialize(ID3D11Device*, ID3D11DeviceContext*, char*, char*);
	void Shutdown();
	void Render(ID3D11DeviceContext*);
	int GetIndexCount();

private:
	bool InitializeBuffers(ID3D11Device*);
	void ShutdownBuffers();
	void RenderBuffers(ID3D11DeviceContext*);

private:
	ID3D11Buffer* m_vertexBuffer;
	ID3D11Buffer* m_indexBuffer;
	int m_vertexCount;
	int m_indexCount;
};

```

<br>


```c++
modelclass.cpp

ModelClass::ModelClass()
{
	m_vertexBuffer = 0;
	m_indexBuffer = 0;
}


ModelClass::ModelClass(const ModelClass& other)
{
}


ModelClass::~ModelClass()
{
}

bool ModelClass::Initialize(ID3D11Device* device)
{
	bool result;
	// 정점 버퍼와 인덱스 버퍼 초기화
	result = InitializeBuffers(device);
	if(!result) return false;

	return true;
}

void ModelClass::Shutdown()
{
	// 정점 버퍼와 인덱스 버퍼 해제
	ShutdownBuffers();
	return;
}


void ModelClass::Render(ID3D11DeviceContext* deviceContext)
{
	// 정점 버퍼와 인덱스 버퍼를 그래픽스 파이프라인에 입력
	RenderBuffers(deviceContext);
	return;
}

int ModelClass::GetIndexCount()
{
	return m_indexCount;
}

bool ModelClass::InitializeBuffers(ID3D11Device* device)
{
	m_indexCount = m_nNumFace * 3;
	// 정점 배열 생성
	VertexType* vertices = new VertexType[m_indexCount];
	if (!vertices)
	{
		return false;
	}

	// 인덱스 배열 생성
	unsigned long* indices = new unsigned long[m_indexCount];
	if (!indices)
	{
		return false;
	}

	for (int i = 0; i < m_nNumFace; i++) // 면의 갯수 만큼
	{
		vertices[i * 3].position = XMFLOAT3(m_vertex[m_face[i].m_vertex[0]][0], m_vertex[m_face[i].m_vertex[0]][1], m_vertex[m_face[i].m_vertex[0]][2]);
		vertices[i * 3].texture = XMFLOAT2(m_uv[m_face[i].m_uv[0]][0], m_uv[m_face[i].m_uv[0]][1]);
		vertices[i * 3 + 1].position = XMFLOAT3(m_vertex[m_face[i].m_vertex[1]][0], m_vertex[m_face[i].m_vertex[1]][1], m_vertex[m_face[i].m_vertex[1]][2]);
		vertices[i * 3 + 1].texture = XMFLOAT2(m_uv[m_face[i].m_uv[1]][0], m_uv[m_face[i].m_uv[1]][1]);

		vertices[i * 3 + 2].position = XMFLOAT3(m_vertex[m_face[i].m_vertex[2]][0], m_vertex[m_face[i].m_vertex[2]][1], m_vertex[m_face[i].m_vertex[2]][2]);
		vertices[i * 3 + 2].texture = XMFLOAT2(m_uv[m_face[i].m_uv[2]][0], m_uv[m_face[i].m_uv[2]][1]);


		if (haveFaceNormal)
		{
			vertices[i * 3].normal = MFLOAT3(m_vertexNormal[m_face[i].m_vertexNormal[0]][0], m_vertexNormal[m_face[i].m_vertexNormal[0]][1], m_vertexNormal[m_face[i].m_vertexNormal[0]][2]);
			vertices[i * 3 + 1].normal = XMFLOAT3(m_vertexNormal[m_face[i].m_vertexNormal[1]][0], m_vertexNormal[m_face[i].m_vertexNormal[1]][1], m_vertexNormal[m_face[i].m_vertexNormal[1]][2]);
			vertices[i * 3 + 2].normal = XMFLOAT3(m_vertexNormal[m_face[i].m_vertexNormal[2]][0], m_vertexNormal[m_face[i].m_vertexNormal[2]][1], m_vertexNormal[m_face[i].m_vertexNormal[2]][2]);
		}

		else
		{
			vertices[i * 3].normal = XMFLOAT3(m_vertexNormal[m_face[i].m_vertex[0]][0], m_vertexNormal[m_face[i].m_vertex[0]][1], m_vertexNormal[m_face[i].m_vertex[0]][2]);
			vertices[i * 3 + 1].normal = XMFLOAT3(m_vertexNormal[m_face[i].m_vertex[1]][0], m_vertexNormal[m_face[i].m_vertex[1]][1], m_vertexNormal[m_face[i].m_vertex[1]][2]);
			vertices[i * 3 + 2].normal = XMFLOAT3(m_vertexNormal[m_face[i].m_vertex[2]][0], m_vertexNormal[m_face[i].m_vertex[2]][1], m_vertexNormal[m_face[i].m_vertex[2]][2]);
		}
	}

	for (int i = 0; i < m_indexCount; i++)
	{
		indices[i] = i;
	}


	// 정적 정점 버퍼의 구조체 설정
	D3D11_BUFFER_DESC vertexBufferDesc;
	vertexBufferDesc.Usage = D3D11_USAGE_DEFAULT;
	vertexBufferDesc.ByteWidth = sizeof(VertexType) * m_indexCount;
	vertexBufferDesc.BindFlags = D3D11_BIND_VERTEX_BUFFER;
	vertexBufferDesc.CPUAccessFlags = 0;
	vertexBufferDesc.MiscFlags = 0;
	vertexBufferDesc.StructureByteStride = 0;

	// 보조 리소스 구조에 정점 데이터에 대한 포인터 제공
	D3D11_SUBRESOURCE_DATA vertexData;
	vertexData.pSysMem = vertices;
	vertexData.SysMemPitch = 0;
	vertexData.SysMemSlicePitch = 0;

	// 정점 버퍼 생성
	if (FAILED(device->CreateBuffer(&vertexBufferDesc, &vertexData, &m_vertexBuffer)))
	{
		return false;
	}

	// 정적 인덱스 버퍼의 구조체 설정
	D3D11_BUFFER_DESC indexBufferDesc;
	indexBufferDesc.Usage = D3D11_USAGE_DEFAULT;
	indexBufferDesc.ByteWidth = sizeof(unsigned long) * m_indexCount;
	indexBufferDesc.BindFlags = D3D11_BIND_INDEX_BUFFER;
	indexBufferDesc.CPUAccessFlags = 0;
	indexBufferDesc.MiscFlags = 0;
	indexBufferDesc.StructureByteStride = 0;

	// 인덱스 데이터를 가리키는 보조 리소스 구조체를 작성
	D3D11_SUBRESOURCE_DATA indexData;
	indexData.pSysMem = indices;
	indexData.SysMemPitch = 0;
	indexData.SysMemSlicePitch = 0;

	// 인덱스 버퍼 생성
	if (FAILED(device->CreateBuffer(&indexBufferDesc, &indexData, &m_indexBuffer)))
	{
		return false;
	}

	// 값이 할당된 정점 버퍼와 인덱스 버퍼 해제
	delete[] vertices;
	vertices = 0;

	delete[] indices;
	indices = 0;

	return true;
}

void ModelClass::ShutdownBuffers()
{
	// 인덱스 버퍼 해제
	if (m_indexBuffer)
	{
		m_indexBuffer->Release();
		m_indexBuffer = 0;
	}

	// 정점 버퍼 해제
	if (m_vertexBuffer)
	{
		m_vertexBuffer->Release();
		m_vertexBuffer = 0;
	}
}

void ModelClass::RenderBuffers(ID3D11DeviceContext* deviceContext)
{
	// 정점 버퍼의 단위와 오프셋을 설정
	UINT stride = sizeof(VertexType);
	UINT offset = 0;

	// 렌더링 할 수 있도록 입력 어셈블러에서 정점 버퍼를 활성으로 설정
	deviceContext->IASetVertexBuffers(0, 1, &m_vertexBuffer, &stride, &offset);

	// 렌더링 할 수 있도록 입력 어셈블러에서 인덱스 버퍼를 활성으로 설정
	deviceContext->IASetIndexBuffer(m_indexBuffer, DXGI_FORMAT_R32_UINT, 0);

	// 정점 버퍼로 그릴 기본형을 설정
	deviceContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
}

```

<br>

shader를 호출하기 위한 colorshaderclass 정의 

```c++
colorshaderclass.h

class ColorShaderClass
{
private:

struct MatrixBufferType
	{
		D3DXMATRIX world;
		D3DXMATRIX view;
		D3DXMATRIX projection;
	};

public:
	ColorShaderClass();
	ColorShaderClass(const ColorShaderClass&);
	~ColorShaderClass();

bool Initialize(ID3D11Device*, HWND);
	void Shutdown();
	bool Render(ID3D11DeviceContext*, int, D3DXMATRIX, D3DXMATRIX, D3DXMATRIX);

private:
	bool InitializeShader(ID3D11Device*, HWND, WCHAR*, WCHAR*);
	void ShutdownShader();
	void OutputShaderErrorMessage(ID3D10Blob*, HWND, WCHAR*);

	bool SetShaderParameters(ID3D11DeviceContext*, D3DXMATRIX, D3DXMATRIX, D3DXMATRIX);
	void RenderShader(ID3D11DeviceContext*, int);

private:
	ID3D11VertexShader* m_vertexShader;
	ID3D11PixelShader* m_pixelShader;
	ID3D11InputLayout* m_layout;
	ID3D11Buffer* m_matrixBuffer;
};

```

<br>


```c++
colorshaderclass.cpp

ColorShaderClass::ColorShaderClass()
{
	m_vertexShader = 0;
	m_pixelShader = 0;
	m_layout = 0;
	m_matrixBuffer = 0;
}


ColorShaderClass::ColorShaderClass(const ColorShaderClass& other)
{
}


ColorShaderClass::~ColorShaderClass()
{
}

bool ColorShaderClass::Initialize(ID3D11Device* device, HWND hwnd)
{
	bool result;
	// 정점 셰이더와 픽셀 셰이더 초기화
	result = InitializeShader(device, hwnd, L"../Engine/color.vs", L"../Engine/color.ps");
	if(!result)
	{
		return false;
	}

	return true;
}

void ColorShaderClass::Shutdown()
{
	// 정점 셰이더와 픽실 셰이더 및 그와 관련된 것들을 반환
	ShutdownShader();

	return;
}

bool ColorShaderClass::Render(ID3D11DeviceContext* deviceContext, int indexCount, D3DXMATRIX worldMatrix, D3DXMATRIX viewMatrix,  D3DXMATRIX projectionMatrix)
{
	bool result;


	// 렌더링에 사용할 셰이더의 인자 입력
	result = SetShaderParameters(deviceContext, worldMatrix, viewMatrix, projectionMatrix);
	if(!result)
	{
		return false;
	}

	// 셰이더를 이용하여 준비된 버퍼 draw
	RenderShader(deviceContext, indexCount);

	return true;
}

bool ColorShaderClass::InitializeShader(ID3D11Device* device, HWND hwnd, WCHAR* vsFilename, WCHAR* psFilename)
{
	HRESULT result;
	ID3D10Blob* errorMessage;
	ID3D10Blob* vertexShaderBuffer;
	ID3D10Blob* pixelShaderBuffer;
	D3D11_INPUT_ELEMENT_DESC polygonLayout[2];
	unsigned int numElements;
	D3D11_BUFFER_DESC matrixBufferDesc;


	// 포인터들을 null로 설정
	errorMessage = 0;
	vertexShaderBuffer = 0;
	pixelShaderBuffer = 0;


	// 정점 셰이더 컴파일
	result = D3DX11CompileFromFile(vsFilename, NULL, NULL, "ColorVertexShader", "vs_5_0", D3D10_SHADER_ENABLE_STRICTNESS, 0, NULL, &vertexShaderBuffer, &errorMessage, NULL);
	if(FAILED(result))
	{
		// 셰이더가 컴파일에 실패하면 에러 메세지를 기록
		if(errorMessage)
		{
			OutputShaderErrorMessage(errorMessage, hwnd, vsFilename);
		}
		// 에러 메세지가 없다면 셰이더 파일을 찾지 못한 것
		else
		{
			MessageBox(hwnd, vsFilename, L"Missing Shader File", MB_OK);
		}

		return false;
	}

	// 픽셀 셰이더를 컴파일
	result = D3DX11CompileFromFile(psFilename, NULL, NULL, "ColorPixelShader", "ps_5_0", D3D10_SHADER_ENABLE_STRICTNESS, 0, NULL, 
									&pixelShaderBuffer, &errorMessage, NULL);  
	if(FAILED(result))
	{
		// 셰이더 컴파일이 실패하면 에러 메세지를 기록
		if(errorMessage)
		{
			OutputShaderErrorMessage(errorMessage, hwnd, psFilename);
		}
		// 에러 메세지가 없다면 단순히 셰이더 파일을 찾지 못한 것
		else
		{
			MessageBox(hwnd, psFilename, L"Missing Shader File", MB_OK);
		}

		return false;
	}
	
    // 버퍼로부터 정점 셰이더를 생성
	result = device->CreateVertexShader(vertexShaderBuffer->GetBufferPointer(), vertexShaderBuffer->GetBufferSize(), NULL, &m_vertexShader);
	if(FAILED(result))
	{
		return false;
	}

	// 버퍼로부터 픽셀 셰이더를 생성
	result = device->CreatePixelShader(pixelShaderBuffer->GetBufferPointer(), pixelShaderBuffer->GetBufferSize(), NULL, &m_pixelShader);
	if(FAILED(result))
	{
		return false;
	}

	// layout description 작성,  ModelClass와 셰이더의 VertexType와 일치해야함
	polygonLayout[0].SemanticName = "POSITION";
	polygonLayout[0].SemanticIndex = 0;
	polygonLayout[0].Format = DXGI_FORMAT_R32G32B32_FLOAT;
	polygonLayout[0].InputSlot = 0;
	polygonLayout[0].AlignedByteOffset = 0;
	polygonLayout[0].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	polygonLayout[0].InstanceDataStepRate = 0;

	polygonLayout[1].SemanticName = "COLOR";
	polygonLayout[1].SemanticIndex = 0;
	polygonLayout[1].Format = DXGI_FORMAT_R32G32B32A32_FLOAT;
	polygonLayout[1].InputSlot = 0;
	polygonLayout[1].AlignedByteOffset = D3D11_APPEND_ALIGNED_ELEMENT;
	polygonLayout[1].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	polygonLayout[1].InstanceDataStepRate = 0;

	// 레이아웃의 요소 갯수를 가져옴
    numElements = sizeof(polygonLayout) / sizeof(polygonLayout[0]);

	// 정점 입력 레이아웃을 생성
	result = device->CreateInputLayout(polygonLayout, numElements, vertexShaderBuffer->GetBufferPointer(), vertexShaderBuffer->GetBufferSize(), &m_layout);
	if(FAILED(result))
	{
		return false;
	}

	// 더 이상 사용되지 않는 정점 셰이더 퍼버와 픽셀 셰이더 버퍼를 해제
	vertexShaderBuffer->Release();
	vertexShaderBuffer = 0;

	pixelShaderBuffer->Release();
	pixelShaderBuffer = 0;


	// 정점 셰이더에 있는 행렬 상수 버퍼의 description 작성
	matrixBufferDesc.Usage = D3D11_USAGE_DYNAMIC;
	matrixBufferDesc.ByteWidth = sizeof(MatrixBufferType);
	matrixBufferDesc.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
	matrixBufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
	matrixBufferDesc.MiscFlags = 0;
	matrixBufferDesc.StructureByteStride = 0;

	// 상수 버퍼 포인터를 만들어 이 클래스에서 정점 셰이더 상수 버퍼에 접근할 수 있게 함
	result = device->CreateBuffer(&matrixBufferDesc, NULL, &m_matrixBuffer);
	if(FAILED(result))
	{
		return false;
	}

	return true;
}
	 
void ColorShaderClass::ShutdownShader()
{
	// 상수 버퍼 해제
	if(m_matrixBuffer)
	{
		m_matrixBuffer->Release();
		m_matrixBuffer = 0;
	}

	// 레이아웃 해제
	if(m_layout)
	{
		m_layout->Release();
		m_layout = 0;
	}

	// 픽셀 셰이더 해제
	if(m_pixelShader)
	{
		m_pixelShader->Release();
		m_pixelShader = 0;
	}

	// 정점 셰이더 해제
	if(m_vertexShader)
	{
		m_vertexShader->Release();
		m_vertexShader = 0;
	}

	return;
}

```

<br>

GraphicsClass에서 만든 행렬을 셰이더에서 사용

```c++

bool ColorShaderClass::SetShaderParameters(ID3D11DeviceContext* deviceContext, D3DXMATRIX worldMatrix, D3DXMATRIX viewMatrix, D3DXMATRIX projectionMatrix)
{
	HRESULT result;
	D3D11_MAPPED_SUBRESOURCE mappedResource;
	MatrixBufferType* dataPtr;
	unsigned int bufferNumber;

	// 행렬을 transpose하여 셰이더에서 사용할 수 있게 함
	D3DXMatrixTranspose(&worldMatrix, &worldMatrix);
	D3DXMatrixTranspose(&viewMatrix, &viewMatrix);
	D3DXMatrixTranspose(&projectionMatrix, &projectionMatrix);

	// 상수 버퍼의 내용을 쓸 수 있도록 잠금
	result = deviceContext->Map(m_matrixBuffer, 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedResource);
	if(FAILED(result))
	{
		return false;
	}

	// 상수 버퍼의 데이터에 대한 포인터를 가져옴
	dataPtr = (MatrixBufferType*)mappedResource.pData;

	// 상수 버퍼에 행렬 복사
	dataPtr->world = worldMatrix;
	dataPtr->view = viewMatrix;
	dataPtr->projection = projectionMatrix;

	// 상수 버퍼의 잠금 해제
	deviceContext->Unmap(m_matrixBuffer, 0);

	// 정점 셰이더에서의 상수 버퍼의 위치 설정
	bufferNumber = 0;

	// 정점 셰이더의 상수 버퍼를 바뀐 값으로 대체
	deviceContext->VSSetConstantBuffers(bufferNumber, 1, &m_matrixBuffer);

	return true;
}

void ColorShaderClass::RenderShader(ID3D11DeviceContext* deviceContext, int indexCount)
{
	// 정점 입력 레이아웃 설정
	deviceContext->IASetInputLayout(m_layout);

	// 삼각형을 그릴 정점 셰이더와 픽셀 셰이더 설정
	deviceContext->VSSetShader(m_vertexShader, NULL, 0);
	deviceContext->PSSetShader(m_pixelShader, NULL, 0);

	// draw
	deviceContext->DrawIndexed(indexCount, 0, 0);
	return;
}

```

<br>

렌더링에 필요한 가상 카메라를 생성하는 CameraClass 정의



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/26f611a7-1381-4b4e-a500-51336ac92098" alt width=400>
<em>카메라</em>
</center>

<br>


```c++
class CameraClass : public AlignedAllocationPolicy<16>
{
public:
	CameraClass();
	CameraClass(const CameraClass&);
	~CameraClass();

	void SetPosition(float, float, float);
	void SetRotation(float, float, float);

	XMFLOAT3 GetPosition();
	XMFLOAT3 GetRotation();

	void Render();
	void GetViewMatrix(XMMATRIX&);

private:
	XMFLOAT3 m_position;
	XMFLOAT3 m_rotation;
	XMMATRIX m_viewMatrix;
};
```

<br>


```c++
CameraClass::CameraClass()
{
	m_position = XMFLOAT3(0.0f, 0.0f, 0.0f);;
	m_rotation = XMFLOAT3(0.0f, 0.0f, 0.0f);
}


CameraClass::CameraClass(const CameraClass& other)
{
}


CameraClass::~CameraClass()
{
}


void CameraClass::SetPosition(float x, float y, float z)
{
	m_position.x = x;
	m_position.y = y;
	m_position.z = z;
}


void CameraClass::SetRotation(float x, float y, float z)
{
	m_rotation.x = x;
	m_rotation.y = y;
	m_rotation.z = z;
}


XMFLOAT3 CameraClass::GetPosition()
{
	return m_position;
}


XMFLOAT3 CameraClass::GetRotation()
{
	return m_rotation;
}


void CameraClass::Render()
{
	XMFLOAT3 up, position, lookAt;
	XMVECTOR upVector, positionVector, lookAtVector;
	float yaw, pitch, roll;
	XMMATRIX rotationMatrix;


	// 위쪽을 가리키는 벡터 설정
	up.x = 0.0f;
	up.y = 1.0f;
	up.z = 0.0f;

	// XMVECTOR 구조체에 로드
	upVector = XMLoadFloat3(&up);

	// 카메라의 위치 ​​설정
	position = m_position;

	// XMVECTOR 구조체에 로드
	positionVector = XMLoadFloat3(&position);

	// 카메라가 방향 설정
	lookAt.x = 0.0f;
	lookAt.y = 0.0f;
	lookAt.z = 1.0f;

	// XMVECTOR 구조체에 로드
	lookAtVector = XMLoadFloat3(&lookAt);

	// yaw (Y 축), pitch (X 축) 및 roll (Z 축)의 회전값을 라디안 단위로 설정
	pitch = m_rotation.x * 0.0174532925f;
	yaw = m_rotation.y * 0.0174532925f;
	roll = m_rotation.z * 0.0174532925f;

	//  yaw, pitch, roll 값을 통해 회전 행렬 생성
	rotationMatrix = XMMatrixRotationRollPitchYaw(pitch, yaw, roll);

	// lookAt 및 up 벡터를 회전 행렬로 변형
	lookAtVector = XMVector3TransformCoord(lookAtVector, rotationMatrix);
	upVector = XMVector3TransformCoord(upVector, rotationMatrix);

	// 회전 된 카메라 위치를 뷰어 위치로 변환
	lookAtVector = XMVectorAdd(positionVector, lookAtVector);

	// 뷰 행렬 생성
	m_viewMatrix = XMMatrixLookAtLH(positionVector, lookAtVector, upVector);
}


void CameraClass::GetViewMatrix(XMMATRIX& viewMatrix)
{
	viewMatrix = m_viewMatrix;
}
```

<br>


GraphicsClass에서 작성한 코드를 사용하여 렌더링한다.

```c++
GraphicsClass.h

#pragma once

const bool FULL_SCREEN = false;
const bool VSYNC_ENABLED = true;
const float SCREEN_DEPTH = 1000.0f;
const float SCREEN_NEAR = 0.1f;

class GraphicsClass
{
public:
	GraphicsClass();
	GraphicsClass(const GraphicsClass&);
	~GraphicsClass();

	bool Initialize(int, int, HWND);
	void Shutdown();
	bool Frame();
	bool Render();

private:
	D3DClass* m_Direct3D = nullptr;
	CameraClass* m_Camera;
	ModelClass* m_Model;
	ColorShaderClass* m_ColorShader;

};
```

<br>


```c++

SystemClass.cpp

#include "d3dclass.h"
#include "graphicsclass.h"

bool GraphicsClass::Initialize(int screenWidth, int screenHeight, HWND hwnd)
{
	m_Direct3D = new D3DClass;
	if (!m_Direct3D) return false;

	if (!m_Direct3D->Initialize(screenWidth, screenHeight, VSYNC_ENABLED, hwnd, FULL_SCREEN, SCREEN_DEPTH, SCREEN_NEAR)) return false;

	m_Camera = new CameraClass;
	if (!m_Camera)
	{
		return false;
	}

	m_Camera->SetPosition(0.0f, 0.0f, -5.0f);

	m_Model = new ModelClass;
	if (!m_Model)
	{
		return false;
	}

	if (!m_Model->Initialize(m_Direct3D->GetDevice(), m_Direct3D->GetDeviceContext(), "../DX11_Shadowmap/data/leather-texture-11614075174.tga", "../DX11_shadowMap_RE/data/sphere.obj"))
	{
		MessageBox(hwnd, L"Could not initialize the model object.", L"Error", MB_OK);
		return false;
	}

	m_ColorShader = new ColorShaderClass;
	if(!m_ColorShader)
	{
		return false;
	}

	result = m_ColorShader->Initialize(m_D3D->GetDevice(), hwnd);
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the color shader object.", L"Error", MB_OK);
		return false;
	}

	return true;
}

bool GraphicsClass::Render()
{
D3DXMATRIX worldMatrix, viewMatrix, projectionMatrix;
	bool result;

	m_D3D->BeginScene(0.0f, 0.0f, 0.0f, 1.0f);

	//변환 작업
	m_Camera->Render();
	m_Camera->GetViewMatrix(viewMatrix);
	m_D3D->GetWorldMatrix(worldMatrix);
	m_D3D->GetProjectionMatrix(projectionMatrix);

	// vertex and index buffer의 데이터를 shader에 전달하기 위해 변환
	m_Model->Render(m_D3D->GetDeviceContext());

	// shader에서 처리
	result = m_ColorShader->Render(m_D3D->GetDeviceContext(), m_Model->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix);
	if(!result)
	{
		return false;
	}

	// 렌더링된 화면을 호출
	m_D3D->EndScene();

	return true;
}

```

<br>

# 출력
---

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d9cb2704-8e24-45ec-97a4-4a34256400b4" alt width=400>
<em>출력 이미지</em>
</center>


<br>
<br>
<br>



출처: www.rastertek.com/tutdx11win10.html

