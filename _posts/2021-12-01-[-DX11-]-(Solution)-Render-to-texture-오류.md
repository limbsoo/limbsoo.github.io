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
  - solution
  - tutorial
---
# 문제
___


Render-to-texture 작업 시, 오류 발생

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8847a3cf-637e-4fd0-9fcb-311d82379cd5" alt width=700>
<em></em>
</center>
<br>

# 해결
___


2D 화면을 그리기 위해서는 Z버퍼를 사용하지 않아야 한다. 그래야 해당 픽셀에 올바로 새로운 색상을 덮어쓰는 것이 가능해진다. 그리고 작업이 끝난 후 다시 z버퍼를 켜야 올바른 3D 모델링이 가능하다.

Initialize 함수에서 DepthEnable 변수가 false인 두 번째 깊이 스텐실을 생성하여 2D렌더링에 사용한다. 이를 통해 두 상태를 바꿔치기 하면서 Z버퍼를 켜고 끌 수 있게 된다.

```c++

bool D3DClass::Initialize(int screenWidth, int screenHeight, bool vsync, HWND hwnd, bool fullscreen,
	float screenDepth, float screenNear)
{
	...

	// 스텐실 상태 구조체를 초기화합니다
	D3D11_DEPTH_STENCIL_DESC depthStencilDesc;
	ZeroMemory(&depthStencilDesc, sizeof(depthStencilDesc));

	// 스텐실 상태 구조체를 작성합니다
	depthStencilDesc.DepthEnable = true;
	depthStencilDesc.DepthWriteMask = D3D11_DEPTH_WRITE_MASK_ALL;
	depthStencilDesc.DepthFunc = D3D11_COMPARISON_LESS;

	depthStencilDesc.StencilEnable = true;
	depthStencilDesc.StencilReadMask = 0xFF;
	depthStencilDesc.StencilWriteMask = 0xFF;

	// 픽셀 정면의 스텐실 설정입니다
	depthStencilDesc.FrontFace.StencilFailOp = D3D11_STENCIL_OP_KEEP;
	depthStencilDesc.FrontFace.StencilDepthFailOp = D3D11_STENCIL_OP_INCR;
	depthStencilDesc.FrontFace.StencilPassOp = D3D11_STENCIL_OP_KEEP;
	depthStencilDesc.FrontFace.StencilFunc = D3D11_COMPARISON_ALWAYS;

	// 픽셀 뒷면의 스텐실 설정입니다
	depthStencilDesc.BackFace.StencilFailOp = D3D11_STENCIL_OP_KEEP;
	depthStencilDesc.BackFace.StencilDepthFailOp = D3D11_STENCIL_OP_DECR;
	depthStencilDesc.BackFace.StencilPassOp = D3D11_STENCIL_OP_KEEP;
	depthStencilDesc.BackFace.StencilFunc = D3D11_COMPARISON_ALWAYS;

	// 깊이 스텐실 상태를 생성합니다
	if (FAILED(m_device->CreateDepthStencilState(&depthStencilDesc, &m_depthStencilState)))
	{
		return false;
	}

	// 깊이 스텐실 상태를 설정합니다
	m_deviceContext->OMSetDepthStencilState(m_depthStencilState, 1);

	// 깊이 스텐실 뷰의 구조체를 초기화합니다
	D3D11_DEPTH_STENCIL_VIEW_DESC depthStencilViewDesc;
	ZeroMemory(&depthStencilViewDesc, sizeof(depthStencilViewDesc));

	// 깊이 스텐실 뷰 구조체를 설정합니다
	depthStencilViewDesc.Format = DXGI_FORMAT_D24_UNORM_S8_UINT;
	depthStencilViewDesc.ViewDimension = D3D11_DSV_DIMENSION_TEXTURE2D;
	depthStencilViewDesc.Texture2D.MipSlice = 0;

	// 깊이 스텐실 뷰를 생성합니다
	if (FAILED(m_device->CreateDepthStencilView(m_depthStencilBuffer, &depthStencilViewDesc, &m_depthStencilView)))
	{
		return false;
	}

	// 렌더링 대상 뷰와 깊이 스텐실 버퍼를 출력 렌더 파이프 라인에 바인딩합니다
	m_deviceContext->OMSetRenderTargets(1, &m_renderTargetView, m_depthStencilView);

	// 그려지는 폴리곤과 방법을 결정할 래스터 구조체를 설정합니다
	D3D11_RASTERIZER_DESC rasterDesc;
	rasterDesc.AntialiasedLineEnable = false;
	rasterDesc.CullMode = D3D11_CULL_BACK;
	rasterDesc.DepthBias = 0;
	rasterDesc.DepthBiasClamp = 0.0f;
	rasterDesc.DepthClipEnable = true;
	rasterDesc.FillMode = D3D11_FILL_SOLID;
	rasterDesc.FrontCounterClockwise = false;
	rasterDesc.MultisampleEnable = false;
	rasterDesc.ScissorEnable = false;
	rasterDesc.SlopeScaledDepthBias = 0.0f;

	// 방금 작성한 구조체에서 래스터 라이저 상태를 만듭니다
	if (FAILED(m_device->CreateRasterizerState(&rasterDesc, &m_rasterState)))
	{
		return false;
	}

	// 이제 래스터 라이저 상태를 설정합니다
	m_deviceContext->RSSetState(m_rasterState);

	// 렌더링을 위해 뷰포트를 설정합니다
	D3D11_VIEWPORT viewport;
	viewport.Width = (float)screenWidth;
	viewport.Height = (float)screenHeight;
	viewport.MinDepth = 0.0f;
	viewport.MaxDepth = 1.0f;
	viewport.TopLeftX = 0.0f;
	viewport.TopLeftY = 0.0f;

	// 뷰포트를 생성합니다
	m_deviceContext->RSSetViewports(1, &viewport);

	// 투영 행렬을 설정합니다
	float fieldOfView = XM_PI / 4.0f;
	float screenAspect = (float)screenWidth / (float)screenHeight;

	// 3D 렌더링을위한 투영 행렬을 만듭니다
	m_projectionMatrix = XMMatrixPerspectiveFovLH(fieldOfView, screenAspect, screenNear, screenDepth);

	// 세계 행렬을 항등 행렬로 초기화합니다
	m_worldMatrix = XMMatrixIdentity();

	// 2D 렌더링을위한 직교 투영 행렬을 만듭니다
	m_orthoMatrix = XMMatrixOrthographicLH((float)screenWidth, (float)screenHeight, screenNear, screenDepth);

	// 이제 2D 렌더링을위한 Z 버퍼를 끄는 두 번째 깊이 스텐실 상태를 만듭니다. 유일한 차이점은
	// DepthEnable을 false로 설정하면 다른 모든 매개 변수는 다른 깊이 스텐실 상태와 동일합니다.
	D3D11_DEPTH_STENCIL_DESC depthDisabledStencilDesc;
	ZeroMemory(&depthDisabledStencilDesc, sizeof(depthDisabledStencilDesc));

	depthDisabledStencilDesc.DepthEnable = false;
	depthDisabledStencilDesc.DepthWriteMask = D3D11_DEPTH_WRITE_MASK_ALL;
	depthDisabledStencilDesc.DepthFunc = D3D11_COMPARISON_LESS;
	depthDisabledStencilDesc.StencilEnable = true;
	depthDisabledStencilDesc.StencilReadMask = 0xFF;
	depthDisabledStencilDesc.StencilWriteMask = 0xFF;
	depthDisabledStencilDesc.FrontFace.StencilFailOp = D3D11_STENCIL_OP_KEEP;
	depthDisabledStencilDesc.FrontFace.StencilDepthFailOp = D3D11_STENCIL_OP_INCR;
	depthDisabledStencilDesc.FrontFace.StencilPassOp = D3D11_STENCIL_OP_KEEP;
	depthDisabledStencilDesc.FrontFace.StencilFunc = D3D11_COMPARISON_ALWAYS;
	depthDisabledStencilDesc.BackFace.StencilFailOp = D3D11_STENCIL_OP_KEEP;
	depthDisabledStencilDesc.BackFace.StencilDepthFailOp = D3D11_STENCIL_OP_DECR;
	depthDisabledStencilDesc.BackFace.StencilPassOp = D3D11_STENCIL_OP_KEEP;
	depthDisabledStencilDesc.BackFace.StencilFunc = D3D11_COMPARISON_ALWAYS;

	// 장치를 사용하여 상태를 만듭니다.
	if (FAILED (m_device-> CreateDepthStencilState (& depthDisabledStencilDesc, & m_depthDisabledStencilState)))
	{
		return false;
	}
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

