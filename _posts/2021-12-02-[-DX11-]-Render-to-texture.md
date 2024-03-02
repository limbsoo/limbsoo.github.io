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
  - tutorial
---
# 구현
---


백버퍼에 렌더링되는 장면을 텍스처에 그린다.

<br>


RenderTextureClass를 정의한다.

```c++
RenderTextureClass.h

class RenderTextureClass
{
public:
	RenderTextureClass();
	RenderTextureClass(const RenderTextureClass&);
	~RenderTextureClass();

	bool Initialize(ID3D11Device*, int, int);
	void Shutdown();

	void SetRenderTarget(ID3D11DeviceContext*, ID3D11DepthStencilView*);
	void ClearRenderTarget(ID3D11DeviceContext*, ID3D11DepthStencilView*, float, float, float, float);
	ID3D11ShaderResourceView* GetShaderResourceView();

private:
	ID3D11Texture2D* m_renderTargetTexture = nullptr;;
	ID3D11RenderTargetView* m_renderTargetView = nullptr;;
	ID3D11ShaderResourceView* m_shaderResourceView = nullptr;;
};

```

<br>


렌더 타겟 텍스처를 생성하여 렌더링이 텍스처에 발생하도록 하며 ID3D11ShaderResourceView를 통해 데이터를 사용한다.

```c++
RenderTextureClass.cpp

bool RenderTextureClass::Initialize(ID3D11Device* device, int textureWidth, int textureHeight)
{
	// 렌더 타겟 텍스처 설명을 초기화합니다.
	D3D11_TEXTURE2D_DESC textureDesc;
	ZeroMemory(&textureDesc, sizeof(textureDesc));

	//800,600

	// 렌더 타겟 텍스처 설명을 설정합니다.
	textureDesc.Width = textureWidth;
	textureDesc.Height = textureHeight;
	textureDesc.MipLevels = 1;
	textureDesc.ArraySize = 1;
	textureDesc.Format = DXGI_FORMAT_R32G32B32A32_FLOAT;
	textureDesc.SampleDesc.Count = 1;
	textureDesc.Usage = D3D11_USAGE_DEFAULT;
	textureDesc.BindFlags = D3D11_BIND_RENDER_TARGET | D3D11_BIND_SHADER_RESOURCE;
	textureDesc.CPUAccessFlags = 0;
	textureDesc.MiscFlags = 0;

	// 렌더 타겟 텍스처를 만듭니다.
	HRESULT result = device->CreateTexture2D(&textureDesc, NULL, &m_renderTargetTexture);
	if (FAILED(result))
	{
		return false;
	}

	// 렌더 타겟 뷰의 설명을 설정합니다.
	D3D11_RENDER_TARGET_VIEW_DESC renderTargetViewDesc;
	renderTargetViewDesc.Format = textureDesc.Format;
	renderTargetViewDesc.ViewDimension = D3D11_RTV_DIMENSION_TEXTURE2D;
	renderTargetViewDesc.Texture2D.MipSlice = 0;

	// 렌더 타겟 뷰를 생성한다.
	result = device->CreateRenderTargetView(m_renderTargetTexture, &renderTargetViewDesc, &m_renderTargetView);
	if (FAILED(result))
	{
		return false;
	}

	// 셰이더 리소스 뷰의 설명을 설정합니다.
	D3D11_SHADER_RESOURCE_VIEW_DESC shaderResourceViewDesc;
	shaderResourceViewDesc.Format = textureDesc.Format;
	shaderResourceViewDesc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;
	shaderResourceViewDesc.Texture2D.MostDetailedMip = 0;
	shaderResourceViewDesc.Texture2D.MipLevels = 1;

	// 셰이더 리소스 뷰를 만듭니다.
	result = device->CreateShaderResourceView(m_renderTargetTexture, &shaderResourceViewDesc, &m_shaderResourceView);
	if (FAILED(result))
	{
		return false;
	}

	return true;
}

void RenderTextureClass::SetRenderTarget(ID3D11DeviceContext* deviceContext, ID3D11DepthStencilView* depthStencilView)
{
	// 렌더링 대상 뷰와 깊이 스텐실 버퍼를 출력 렌더 파이프 라인에 바인딩합니다.
	deviceContext->OMSetRenderTargets(1, &m_renderTargetView, depthStencilView);
}

//렌더타겟텍스처 초기화, 백버퍼에 그리지 않으므로 렌더타겟텍스처를 초기화해준다.
void RenderTextureClass::ClearRenderTarget(ID3D11DeviceContext* deviceContext, ID3D11DepthStencilView* depthStencilView,
	float red, float green, float blue, float alpha)
{
	// 버퍼를 지울 색을 설정합니다.
	float color[4] = { red, green, blue, alpha };

	// 백 버퍼를 지운다.
	deviceContext->ClearRenderTargetView(m_renderTargetView, color);

	// 깊이 버퍼를 지운다.
	deviceContext->ClearDepthStencilView(depthStencilView, D3D11_CLEAR_DEPTH, 1.0f, 0);
}

ID3D11ShaderResourceView* RenderTextureClass::GetShaderResourceView()
{
	return m_shaderResourceView;
}

```

<br>

렌더 타겟 텍스처는 2D이미지이므로 2D이미지를 렌더링하는 클래스 DebugWindowClass를 정의

```c++
DebugWindowClass.h

class DebugWindowClass
{
private:
	struct VertexType
	{
		XMFLOAT3 position;
		XMFLOAT2 texture;
	};

public:
	DebugWindowClass();
	DebugWindowClass(const DebugWindowClass&);
	~DebugWindowClass();

	bool Initialize(ID3D11Device*, int, int, int, int);
	void Shutdown();
	bool Render(ID3D11DeviceContext*, int, int);

	int GetIndexCount();

private:
	bool InitializeBuffers(ID3D11Device*);
	void ShutdownBuffers();
	bool UpdateBuffers(ID3D11DeviceContext*, int, int);
	void RenderBuffers(ID3D11DeviceContext*);

private:
	ID3D11Buffer *m_vertexBuffer = nullptr;
	ID3D11Buffer *m_indexBuffer = nullptr;
	int m_vertexCount = 0;
	int m_indexCount = 0;
	int m_screenWidth = 0;
	int m_screenHeight = 0;
	int m_bitmapWidth = 0;
	int m_bitmapHeight = 0;
	int m_previousPosX = 0;
	int m_previousPosY = 0;

	ModelClass* m_Model = nullptr;

};

```

<br>

```c++
DebugWindowClass.cpp

bool DebugWindowClass::Initialize(ID3D11Device* device, int screenWidth, int screenHeight, int bitmapWidth, int bitmapHeight)
{
	// 화면 크기를 저장
	m_screenWidth = screenWidth;
	m_screenHeight = screenHeight;

	// 이 비트맵을 렌더링 할 픽셀의 크기를 저장합니다.
	m_bitmapWidth = bitmapWidth;
	m_bitmapHeight = bitmapHeight;

	// 이전 렌더링 위치를 음수로 초기화합니다.
	m_previousPosX = -1;
	m_previousPosY = -1;

	// 정점 및 인덱스 버퍼를 초기화합니다.
	m_Model = new ModelClass;
	if (!m_Model)
	{
		return false;
	}

	//// 텍스처 오브젝트를 초기화한다.
	//return m_Model->InitializeBuffers(device);


	return InitializeBuffers(device);
}


void DebugWindowClass::Shutdown()
{
	// 버텍스 및 인덱스 버퍼를 종료합니다.
	ShutdownBuffers();
}


bool DebugWindowClass::Render(ID3D11DeviceContext* deviceContext, int positionX, int positionY)
{
	// 화면의 다른 위치로 렌더링하기 위해 동적 정점 버퍼를 다시 빌드합니다.
	//if (!UpdateBuffers(deviceContext, positionX, positionY))
	//{
	//	return false;
	//}

	// 그리기를 준비하기 위해 그래픽 파이프 라인에 꼭지점과 인덱스 버퍼를 놓습니다.
	RenderBuffers(deviceContext);

	return true;
}


int DebugWindowClass::GetIndexCount()
{
	return m_indexCount;
}


bool DebugWindowClass::InitializeBuffers(ID3D11Device* device)
{
	// 정점 배열의 정점 수를 설정합니다.
	m_vertexCount = 6;

	// 인덱스 배열의 인덱스 수를 설정합니다.
	m_indexCount = m_vertexCount;

	// 정점 배열을 만듭니다.
	VertexType* vertices = new VertexType[m_vertexCount];
	if (!vertices)
	{
		return false;
	}

	// 인덱스 배열을 만듭니다.
	unsigned long* indices = new unsigned long[m_indexCount];
	if (!indices)
	{
		return false;
	}

	// 처음에는 정점 배열을 0으로 초기화합니다.
	memset(vertices, 0, (sizeof(VertexType) * m_vertexCount));

	// 데이터로 인덱스 배열을로드합니다.
	for (int i = 0; i<m_indexCount; i++)
	{
		indices[i] = i;
	}

	// 정적 정점 버퍼의 설명을 설정한다.
	D3D11_BUFFER_DESC vertexBufferDesc;
	vertexBufferDesc.Usage = D3D11_USAGE_DYNAMIC;
	vertexBufferDesc.ByteWidth = sizeof(VertexType) * m_vertexCount;
	vertexBufferDesc.BindFlags = D3D11_BIND_VERTEX_BUFFER;
	vertexBufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
	vertexBufferDesc.MiscFlags = 0;
	vertexBufferDesc.StructureByteStride = 0;

	// subresource 구조에 정점 데이터에 대한 포인터를 제공합니다.
	D3D11_SUBRESOURCE_DATA vertexData;
	vertexData.pSysMem = vertices;
	vertexData.SysMemPitch = 0;
	vertexData.SysMemSlicePitch = 0;

	// 이제 정점 버퍼를 만듭니다.
	if (FAILED(device->CreateBuffer(&vertexBufferDesc, &vertexData, &m_vertexBuffer)))
	{
		return false;
	}

	// 정적 인덱스 버퍼의 설명을 설정합니다.
	D3D11_BUFFER_DESC indexBufferDesc;
	indexBufferDesc.Usage = D3D11_USAGE_DEFAULT;
	indexBufferDesc.ByteWidth = sizeof(unsigned long) * m_indexCount;
	indexBufferDesc.BindFlags = D3D11_BIND_INDEX_BUFFER;
	indexBufferDesc.CPUAccessFlags = 0;
	indexBufferDesc.MiscFlags = 0;
	indexBufferDesc.StructureByteStride = 0;

	// 하위 리소스 구조에 인덱스 데이터에 대한 포인터를 제공합니다.
	D3D11_SUBRESOURCE_DATA indexData;
	indexData.pSysMem = indices;
	indexData.SysMemPitch = 0;
	indexData.SysMemSlicePitch = 0;

	// 인덱스 버퍼를 만듭니다.
	if (FAILED(device->CreateBuffer(&indexBufferDesc, &indexData, &m_indexBuffer)))
	{
		return false;
	}

	// 이제 버텍스와 인덱스 버퍼가 생성되고로드 된 배열을 해제하십시오.
	delete[] vertices;
	vertices = 0;

	delete[] indices;
	indices = 0;

	return true;
}


bool DebugWindowClass::UpdateBuffers(ID3D11DeviceContext* deviceContext, int positionX, int positionY)
{
	//이 비트 맵을 렌더링 할 위치가 변경되지 않은 경우 정점 버퍼를 업데이트하지 마십시오.
	// 현재 올바른 매개 변수가 있습니다.
	if ((positionX == m_previousPosX) && (positionY == m_previousPosY))
	{
		return true;
	}

	// 변경된 경우 렌더링되는 위치를 업데이트합니다.
	m_previousPosX = positionX;
	m_previousPosY = positionY;

	// 비트 맵 왼쪽의 화면 좌표를 계산합니다.
	float left = (float)((m_screenWidth / 2) * -1) + (float)positionX;

	// 비트 맵 오른쪽의 화면 좌표를 계산합니다.
	float right = left + (float)m_bitmapWidth;

	// 비트 맵 상단의 화면 좌표를 계산합니다.
	float top = (float)(m_screenHeight / 2) - (float)positionY;

	// 비트 맵 아래쪽의 화면 좌표를 계산합니다.
	float bottom = top - (float)m_bitmapHeight;

	// 정점 배열을 만듭니다.
	VertexType* vertices = new VertexType[m_vertexCount];
	if (!vertices)
	{
		return false;
	}

	// 정점 배열에 데이터를로드합니다.
	// 첫 번째 삼각형.
	vertices[0].position = XMFLOAT3(left, top, 0.0f);  // Top left.
	vertices[0].texture = XMFLOAT2(0.0f, 0.0f);

	vertices[1].position = XMFLOAT3(right, bottom, 0.0f);  // Bottom right.
	vertices[1].texture = XMFLOAT2(1.0f, 1.0f);

	vertices[2].position = XMFLOAT3(left, bottom, 0.0f);  // Bottom left.
	vertices[2].texture = XMFLOAT2(0.0f, 1.0f);

	// 두 번째 삼각형.
	vertices[3].position = XMFLOAT3(left, top, 0.0f);  // Top left.
	vertices[3].texture = XMFLOAT2(0.0f, 0.0f);

	vertices[4].position = XMFLOAT3(right, top, 0.0f);  // Top right.
	vertices[4].texture = XMFLOAT2(1.0f, 0.0f);

	vertices[5].position = XMFLOAT3(right, bottom, 0.0f);  // Bottom right.
	vertices[5].texture = XMFLOAT2(1.0f, 1.0f);

	// 버텍스 버퍼를 쓸 수 있도록 잠급니다.
	D3D11_MAPPED_SUBRESOURCE mappedResource;
	if (FAILED(deviceContext->Map(m_vertexBuffer, 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedResource)))
	{
		return false;
	}

	// 정점 버퍼의 데이터를 가리키는 포인터를 얻는다.
	VertexType* verticesPtr = (VertexType*)mappedResource.pData;

	// 데이터를 정점 버퍼에 복사합니다.
	memcpy(verticesPtr, (void*)vertices, (sizeof(VertexType) * m_vertexCount));

	// 정점 버퍼의 잠금을 해제합니다.
	deviceContext->Unmap(m_vertexBuffer, 0);

	// 더 이상 필요하지 않은 꼭지점 배열을 해제합니다.
	delete[] vertices;
	vertices = 0;

	return true;
}


void DebugWindowClass::RenderBuffers(ID3D11DeviceContext* deviceContext)
{
	// 정점 버퍼 보폭 및 오프셋을 설정합니다.
	unsigned int stride = sizeof(VertexType);
	unsigned int offset = 0;

	// 렌더링 할 수 있도록 입력 어셈블러에서 정점 버퍼를 활성으로 설정합니다.
	deviceContext->IASetVertexBuffers(0, 1, &m_vertexBuffer, &stride, &offset);

	// 렌더링 할 수 있도록 입력 어셈블러에서 인덱스 버퍼를 활성으로 설정합니다.
	deviceContext->IASetIndexBuffer(m_indexBuffer, DXGI_FORMAT_R32_UINT, 0);

	// 이 꼭지점 버퍼에서 렌더링되어야하는 프리미티브 유형을 설정합니다.이 경우에는 삼각형입니다.
	deviceContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
}

```

<br>

D3DClass에 스텐실과 렌더 타겟 텍스처를 호출하는 함수를 정의하고 2D 렌더링 시 z-buffer를 꺼야 하므로 함수를 추가해준다.

```c++
D3DClass.cpp

...

ID3D11DepthStencilView* D3DClass::GetDepthStencilView()
{
	return m_depthStencilView;
}


void D3DClass::SetBackBufferRenderTarget()
{
	// 렌더링 대상 뷰와 깊이 스텐실 버퍼를 출력 렌더 파이프 라인에 바인딩합니다.
	m_deviceContext->OMSetRenderTargets(1, &m_renderTargetView, m_depthStencilView);
}

void D3DClass::TurnZBufferOn()
{
	m_deviceContext->OMSetDepthStencilState(m_depthStencilState, 1);
}


void D3DClass::TurnZBufferOff()
{
	m_deviceContext->OMSetDepthStencilState(m_depthDisabledStencilState, 1);
}


```

<br>

GraphicsClass에서 해당 객체를 생성하고 렌더링 한다.

```c++

GraphicsClass.cpp

bool GraphicsClass::Initialize(int screenWidth, int screenHeight, HWND hwnd)
{
	...

	// 렌더링 텍스처 객체를 생성한다.
	m_RenderTexture = new RenderTextureClass;
	if(!m_RenderTexture)
	{
		return false;
	}

	// 렌더링 텍스처 객체를 초기화한다.
	if(!m_RenderTexture->Initialize(m_Direct3D->GetDevice(), screenWidth, screenHeight))
	{
		return false;
	}

	// 디버그 창 객체를 만듭니다.
	m_DebugWindow = new DebugWindowClass;
	if(!m_DebugWindow)
	{
		return false;
	}

	// 디버그 창 객체를 초기화 합니다.
	if(!m_DebugWindow->Initialize(m_Direct3D->GetDevice(), screenWidth, screenHeight, 100, 100))
	{
		MessageBox(hwnd, L"Could not initialize the debug window object.", L"Error", MB_OK);
		return false;
	}

	// 텍스처 쉐이더 객체를 생성한다.
	m_TextureShader = new TextureShaderClass;
	if(!m_TextureShader)
	{
		return false;
	}

	// 텍스처 쉐이더 객체를 초기화한다.
	if(!m_TextureShader->Initialize(m_Direct3D->GetDevice(), hwnd))
	{
		MessageBox(hwnd, L"Could not initialize the texture shader object.", L"Error", MB_OK);
		return false;
	}

	return true;
}

bool GraphicsClass::Render()
{
	// 전체 장면을 먼저 텍스처로 렌더링합니다.
	if(!RenderToTexture())
	{
		return false;
	}

	// 씬을 그리기 위해 버퍼를 지웁니다
	m_Direct3D->BeginScene(0.0f, 0.0f, 0.0f, 1.0f);

	// 백 버퍼의 장면을 정상적으로 렌더링합니다.
	if(!RenderScene())
	{
		return false;
	}

	// 모든 2D 렌더링을 시작하려면 Z 버퍼를 끕니다.
	m_Direct3D->TurnZBufferOff();

	// 카메라 및 d3d 객체에서 월드, 뷰 및 투영 행렬을 가져옵니다
	XMMATRIX worldMatrix, viewMatrix, orthoMatrix;

	m_Camera->GetViewMatrix(viewMatrix);
	m_Direct3D->GetWorldMatrix(worldMatrix);
	m_Direct3D->GetOrthoMatrix(orthoMatrix);

	//디버그 윈도우 버텍스와 인덱스 버퍼를 그래픽 파이프 라인에 배치하여 그리기를 준비합니다.
	if(!m_DebugWindow->Render(m_Direct3D->GetDeviceContext(), 350, 250))
	{
		return false;
	}

	 //텍스처 셰이더를 사용해 디버그 윈도우를 렌더링한다.
	if(!m_TextureShader->Render(m_Direct3D->GetDeviceContext(), m_DebugWindow->GetIndexCount(), worldMatrix, viewMatrix, orthoMatrix, m_RenderTexture->GetShaderResourceView()))
	{
		return false;
	}

	// 모든 2D 렌더링이 완료되었으므로 Z 버퍼를 다시 킨다.
	m_Direct3D->TurnZBufferOn();
	
	// 버퍼의 내용을 화면에 출력합니다
	m_Direct3D->EndScene();

	return true;
}


bool GraphicsClass::RenderToTexture()
{
	// 렌더링 대상을 렌더링에 맞게 설정합니다.
	m_RenderTexture->SetRenderTarget(m_Direct3D->GetDeviceContext(), m_Direct3D->GetDepthStencilView());

	// 렌더링을 텍스처에 지웁니다.
	m_RenderTexture->ClearRenderTarget(m_Direct3D->GetDeviceContext(), m_Direct3D->GetDepthStencilView(), 0.0f, 0.0f, 1.0f, 1.0f);

	// 이제 장면을 렌더링하면 백 버퍼 대신 텍스처로 렌더링됩니다.

	if(!RenderTextureScene())
	{
		return false;
	}

	// 렌더링 대상을 원래의 백 버퍼로 다시 설정하고 렌더링에 대한 렌더링을 더 이상 다시 설정하지 않습니다.
	m_Direct3D->SetBackBufferRenderTarget();

	return true;
}


bool GraphicsClass::RenderScene()
{
	// 카메라의 위치에 따라 뷰 행렬을 생성합니다
	m_Camera->Render();

	// 카메라 및 d3d 객체에서 월드, 뷰 및 투영 행렬을 가져옵니다
	XMMATRIX worldMatrix, viewMatrix, projectionMatrix, orthoMatrix;
	m_Camera->GetViewMatrix(viewMatrix);
	m_Direct3D->GetWorldMatrix(worldMatrix);
	m_Direct3D->GetProjectionMatrix(projectionMatrix);
	m_Direct3D->GetOrthoMatrix(orthoMatrix);

	// 각 프레임의 rotation 변수를 업데이트합니다.
	static float rotation = 0.0f;
	rotation += (float)XM_PI * 0.0025f;
	if(rotation > 360.0f)
	{
		rotation -= 360.0f;
	}

	// 회전 값으로 월드 행렬을 회전합니다.
	worldMatrix = XMMatrixRotationY(rotation);

	// 모델 버텍스와 인덱스 버퍼를 그래픽 파이프 라인에 배치하여 렌더링 합니다.
	m_Model->Render(m_Direct3D->GetDeviceContext());

	return m_LightShader->Render(m_Direct3D->GetDeviceContext(), m_Model->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix,
		m_RenderTexture->GetShaderResourceView(), m_Light->GetDirection(), m_Light->GetDiffuseColor());

}

bool GraphicsClass::RenderTextureScene()
{
	// 카메라의 위치에 따라 뷰 행렬을 생성합니다
	m_Camera->Render();

	// 카메라 및 d3d 객체에서 월드, 뷰 및 투영 행렬을 가져옵니다
	XMMATRIX worldMatrix, viewMatrix, projectionMatrix, orthoMatrix;
	m_Camera->GetViewMatrix(viewMatrix);
	m_Direct3D->GetWorldMatrix(worldMatrix);
	m_Direct3D->GetProjectionMatrix(projectionMatrix);
	m_Direct3D->GetOrthoMatrix(orthoMatrix);

	// 각 프레임의 rotation 변수를 업데이트합니다.
	static float rotation = 0.0f;
	rotation += (float)XM_PI * 0.0025f;
	if (rotation > 360.0f)
	{
		rotation -= 360.0f;
	}

	// 회전 값으로 월드 행렬을 회전합니다.
	worldMatrix = XMMatrixRotationY(rotation);

	// 모델 버텍스와 인덱스 버퍼를 그래픽 파이프 라인에 배치하여 렌더링 합니다.
	m_Model->Render(m_Direct3D->GetDeviceContext());

	//m_renderTextureView = m_Model->GetTexture();

	// 라이트 쉐이더를 사용하여 모델을 렌더링합니다.
	return m_LightShader->Render(m_Direct3D->GetDeviceContext(), m_Model->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix,
		m_Model->GetTexture(), m_Light->GetDirection(), m_Light->GetDiffuseColor());

}

```

<br>
<br>


# 출력
---

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/685b1390-14ec-4c21-a7e5-dfbbc3dd274b" alt width=700>
<em>출력 이미지</em>
</center>


<br>
<br>
<br>



출처: www.rastertek.com/tutdx11win10.html

