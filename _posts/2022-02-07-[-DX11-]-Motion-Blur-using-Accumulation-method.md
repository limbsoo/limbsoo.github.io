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


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/64be750a-acce-4ce2-bca0-d25aebd1bc02" alt width=400>
<em>Motion blur</em>
</center>

<br>

모션 블러는 움직임이 흐릿함 을 뜻하는 말로,사진이나 애니메이션에서 동작의 빠름을 효과적으로 나타내기 위해 사용하는 기법.

<br>



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ce0e51d9-74f6-4efd-81c6-c1c42927d0cb" alt width=600>
<em>Motion blur</em>
</center>

<br>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/544d21f9-6471-47f8-9c2e-15a5e6e6a990" alt width=300>
<em>Motion blur</em>
</center>

<br>


카메라 셔터가 닫힐 때까지 들어오는 빛을 저장하는 과정에서 피사체의 움직임이 셔터 스피드보다 빠르게 움직일 경우 발생한다. 따라서 물체의 움직임에 따른 여러 이미지를 생성하고 이를 하나의 최종 이미지로 변환하여 모션 블러가 적용된 이미지를 출력한다.

<br>


여러 장의 렌더 텍스처를 관리하는 클래스 RenderTexture를 생성한다.


```c++

class RenderTexture
{
public:
	RenderTexture();
	RenderTexture(const RenderTexture&);
	~RenderTexture();

	void Shutdown();
	void ClearRenderTarget(ID3D11DeviceContext*, ID3D11DepthStencilView*, float, float, float, float);
	void SetRenderTarget(ID3D11DeviceContext*, ID3D11DepthStencilView*);

	bool Initialize(ID3D11Device*, int, int);
	bool Initialize_DepthBuffer(ID3D11Device* pdevice, int textureWidth, int textureHeight);

	ID3D11ShaderResourceView*	GetShaderResourceView();
	ID3D11RenderTargetView*		GetRenderTargetView();
	ID3D11DepthStencilView*		GetDepthStencilView();
	ID3D11RenderTargetView** ppRTVs;
	ID3D11ShaderResourceView** ppSRVs;

private:

	IDXGISwapChain*				m_swapChain = nullptr;
	ID3D11Device*				m_device = nullptr;
	ID3D11DeviceContext*		m_deviceContext = nullptr;
	ID3D11Texture2D*			m_renderTargetTexture;
	ID3D11Texture2D*			m_depthStencilTexture;
	ID3D11RenderTargetView*		m_renderTargetView;
	ID3D11DepthStencilView*		m_depthStencilView;
	ID3D11ShaderResourceView*	m_shaderResourceView;
};
```

<br>



```c++
bool RenderTexture::Initialize_DepthBuffer(ID3D11Device* pdevice, int textureWidth, int textureHeight)
{
	HRESULT result;

	D3D11_TEXTURE2D_DESC depthStencilDesc;

	depthStencilDesc.Width = textureWidth;
	depthStencilDesc.Height = textureHeight;
	depthStencilDesc.MipLevels = 1;
	depthStencilDesc.ArraySize = 1;
	depthStencilDesc.Format = DXGI_FORMAT_R32_TYPELESS;
	depthStencilDesc.SampleDesc.Count = 1;
	depthStencilDesc.SampleDesc.Quality = 0;
	depthStencilDesc.Usage = D3D11_USAGE_DEFAULT;
	depthStencilDesc.BindFlags = D3D11_BIND_DEPTH_STENCIL | D3D11_BIND_SHADER_RESOURCE;
	depthStencilDesc.CPUAccessFlags = 0;
	depthStencilDesc.MiscFlags = 0;

	result = pdevice->CreateTexture2D(&depthStencilDesc, 0, &m_depthStencilTexture);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'DepthBufferDepthStencil'", L"Error", MB_OK | MB_ICONERROR); }

	D3D11_DEPTH_STENCIL_VIEW_DESC dsv_desc;
	dsv_desc.Flags = 0;
	dsv_desc.Format = DXGI_FORMAT_D32_FLOAT;
	dsv_desc.ViewDimension = D3D11_DSV_DIMENSION_TEXTURE2D;
	dsv_desc.Texture2D.MipSlice = 0;

	result = pdevice->CreateDepthStencilView(m_depthStencilTexture, &dsv_desc, &m_depthStencilView);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'DepthBufferDepthStencilView'", L"Error", MB_OK | MB_ICONERROR); }

	D3D11_SHADER_RESOURCE_VIEW_DESC sr_desc;
	sr_desc.Format = DXGI_FORMAT_R32_FLOAT;
	sr_desc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;
	sr_desc.Texture2D.MostDetailedMip = 0;
	sr_desc.Texture2D.MipLevels = 1;

	result = pdevice->CreateShaderResourceView(m_depthStencilTexture, &sr_desc, &m_shaderResourceView);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'DepthBufferShaderResourceView'", L"Error", MB_OK | MB_ICONERROR); }

	return true;
}

bool RenderTexture::Initialize(ID3D11Device* device, int Width, int Height)
{
	HRESULT result;

	D3D11_TEXTURE2D_DESC textureDesc; /// Setup the render target texture description.
	ZeroMemory(&textureDesc, sizeof(textureDesc)); /// Initialize the render target texture description.
	textureDesc.Width = Width;
	textureDesc.Height = Height;
	textureDesc.MipLevels = 0;
	textureDesc.ArraySize = N_LAYER;
	textureDesc.Format = DXGI_FORMAT_R32G32B32A32_FLOAT; // DXGI_FORMAT_R16G16B16A16_FLOAT
	textureDesc.SampleDesc.Count = 1;
	textureDesc.SampleDesc.Quality = 0;
	textureDesc.Usage = D3D11_USAGE_DEFAULT;
	textureDesc.BindFlags = D3D11_BIND_RENDER_TARGET | D3D11_BIND_SHADER_RESOURCE;
	textureDesc.CPUAccessFlags = 0;
	textureDesc.MiscFlags = D3D11_RESOURCE_MISC_GENERATE_MIPS;

	result = device->CreateTexture2D(&textureDesc, NULL, &m_renderTargetTexture);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'Texture2D'", L"Error", MB_OK | MB_ICONERROR); }

	result = device->CreateRenderTargetView(m_renderTargetTexture, NULL, &m_renderTargetView);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerRenderTargetView'", L"Error", MB_OK | MB_ICONERROR); }

	result = device->CreateShaderResourceView(m_renderTargetTexture, NULL, &m_shaderResourceView);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerShaderResourceView'", L"Error", MB_OK | MB_ICONERROR); }

	D3D11_RENDER_TARGET_VIEW_DESC layerViewDesc;
	m_renderTargetView->GetDesc(&layerViewDesc);
	layerViewDesc.ViewDimension = D3D11_RTV_DIMENSION_TEXTURE2DARRAY;
	layerViewDesc.Texture2DArray.ArraySize = 1;
	layerViewDesc.Format = DXGI_FORMAT_R32G32B32A32_FLOAT;

	ppRTVs = new ID3D11RenderTargetView * [N_LAYER];

	ppSRVs = new ID3D11ShaderResourceView * [N_LAYER];

	for (int i = 0; i < N_LAYER; ++i)
	{
		layerViewDesc.Texture2DArray.FirstArraySlice = i;

		result = device->CreateRenderTargetView(m_renderTargetTexture, &layerViewDesc, &ppRTVs[i]);
		if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerTexture2DArrayRenderTargetView'", L"Error", MB_OK | MB_ICONERROR); }
	}

	result = device->CreateShaderResourceView(m_renderTargetTexture, NULL, &ppSRVs[0]);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerTexture2DArrayShaderResourceView'", L"Error", MB_OK | MB_ICONERROR); }

	//------------------------------------------------------------------------------
	// Also Create depth stencil buffer
	// In order to a situation that back buffer size bigger than window size
	//------------------------------------------------------------------------------
	D3D11_TEXTURE2D_DESC textureDesc_; /// Setup the render target texture description.
	ZeroMemory(&textureDesc_, sizeof(textureDesc_)); /// Initialize the render target texture description.
	textureDesc_.Width = Width;
	textureDesc_.Height = Height;
	textureDesc_.MipLevels = 1;
	textureDesc_.ArraySize = 1;
	textureDesc_.Format = DXGI_FORMAT_R24G8_TYPELESS;
	textureDesc_.SampleDesc.Count = 1;
	textureDesc_.SampleDesc.Quality = 0;
	textureDesc_.Usage = D3D11_USAGE_DEFAULT;
	textureDesc_.BindFlags = D3D11_BIND_DEPTH_STENCIL | D3D11_BIND_SHADER_RESOURCE;
	textureDesc_.CPUAccessFlags = 0;
	textureDesc_.MiscFlags = 0;
	/// Create the render target texture.

	result = device->CreateTexture2D(&textureDesc_, NULL, &m_depthStencilTexture);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'Texture2D'", L"Error", MB_OK | MB_ICONERROR); }

	/// Create the depth stencil view desc
	D3D11_DEPTH_STENCIL_VIEW_DESC depthStencilViewDesc;
	ZeroMemory(&depthStencilViewDesc, sizeof(depthStencilViewDesc));
	depthStencilViewDesc.Format = DXGI_FORMAT_D24_UNORM_S8_UINT;
	depthStencilViewDesc.ViewDimension = D3D11_DSV_DIMENSION_TEXTURE2D;

	depthStencilViewDesc.Texture2D.MipSlice = 0;
	result = device->CreateDepthStencilView(m_depthStencilTexture, &depthStencilViewDesc, &m_depthStencilView);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'Depth Stencil View'", L"Error", MB_OK | MB_ICONERROR); }

	return true;
}

void RenderTexture::SetRenderTarget(ID3D11DeviceContext* deviceContext, ID3D11DepthStencilView* depthStencilView)
{
	// Bind the render target view and depth stencil buffer to the output render pipeline.
	deviceContext->OMSetRenderTargets(1, &m_renderTargetView, depthStencilView);

	return;
}

void RenderTexture::ClearRenderTarget(ID3D11DeviceContext* deviceContext, ID3D11DepthStencilView* depthStencilView,
	float red, float green, float blue, float alpha)
{
	float color[4];

	// Setup the color to clear the buffer to.
	color[0] = red;
	color[1] = green;
	color[2] = blue;
	color[3] = alpha;

	// Clear the back buffer.
	deviceContext->ClearRenderTargetView(m_renderTargetView, color);

	// Clear the depth buffer.
	deviceContext->ClearDepthStencilView(depthStencilView, D3D11_CLEAR_DEPTH, 1.0f, 0);

	return;
}

ID3D11ShaderResourceView* RenderTexture::GetShaderResourceView()
{
	return m_shaderResourceView;
}

ID3D11RenderTargetView* RenderTexture::GetRenderTargetView()
{
	return m_renderTargetView;
}

ID3D11DepthStencilView* RenderTexture::GetDepthStencilView()
{
	return m_depthStencilView;
}
```


<br>



각각의 변수에서 m_shaderResourceView에 바인딩, m_renderTargetView, m_depthStencilView 등을 초기화한다. 그리고 모델을 그릴 준비 한다.


```c++

//--------------------------------------------------------------------------------------
// Create any D3D resources that aren't dependant on the back buffer
//--------------------------------------------------------------------------------------
HRESULT CALLBACK OnD3D11CreateDevice(ID3D11Device* pd3dDevice, const DXGI_SURFACE_DESC* pBackBufferSurfaceDesc, void* pUserContext)
{
    HRESULT hr = S_OK;

    ID3D11DeviceContext* pd3dImmediateContext = DXUTGetD3D11DeviceContext();
    V_RETURN(g_DialogResourceManager.OnD3D11CreateDevice(pd3dDevice, pd3dImmediateContext));
    V_RETURN(g_D3DSettingsDlg.OnD3D11CreateDevice(pd3dDevice));
    g_pTxtHelper = new CDXUTTextHelper(pd3dDevice, pd3dImmediateContext, &g_DialogResourceManager, 15);

    DWORD dwShaderFlags = D3DCOMPILE_ENABLE_STRICTNESS;
    dwShaderFlags |= D3DCOMPILE_DEBUG;
    dwShaderFlags |= D3DCOMPILE_SKIP_OPTIMIZATION;

    hr = D3DX11CompileFromFile(L"DynamicShaderLinkageFX11.fx", 0, 0, 0, "fx_5_0", dwShaderFlags, 0, 0, &g_pEffectBlob, &g_pErrorsBlob, 0);
    assert(SUCCEEDED(hr) && g_pEffectBlob);

    if (g_pErrorsBlob)
    {
        MessageBoxA(0, (char*)g_pErrorsBlob->GetBufferPointer(), 0, 0);
        g_pErrorsBlob->Release();
    }

    hr = D3DX11CreateEffectFromMemory(g_pEffectBlob->GetBufferPointer(), g_pEffectBlob->GetBufferSize(), 0, pd3dDevice, &g_pEffect);
    assert(SUCCEEDED(hr));
    g_pEffectBlob->Release();

    D3DX11_PASS_DESC passDesc;
    g_pTech = g_pEffect->GetTechniqueByName("ReferenceRendering");
    g_pTech->GetPassByIndex(0)->GetDesc(&passDesc);

    const D3D11_INPUT_ELEMENT_DESC layout[] =
    {
        {"POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0  },
        {"TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT, 0, D3D11_APPEND_ALIGNED_ELEMENT, D3D11_INPUT_PER_VERTEX_DATA, 0  },    //12
        {"NORMAL", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, D3D11_APPEND_ALIGNED_ELEMENT, D3D11_INPUT_PER_VERTEX_DATA, 0  },
    };

    V_RETURN(pd3dDevice->CreateInputLayout(layout, ARRAYSIZE(layout), passDesc.pIAInputSignature, passDesc.IAInputSignatureSize, &g_pLayout));

    LoadModels(pd3dDevice, pd3dImmediateContext);

    //GetSRVEffect();
    g_Ref_Primary_effectSRV_colorMap = g_pEffect->GetVariableByName("g_Ref_Primary_colorMap")->AsShaderResource();
    g_Ref_Secondary_effectSRV_colorMap = g_pEffect->GetVariableByName("g_Ref_Secondary_colorMap")->AsShaderResource();
    g_Ref_Tertiary_effectSRV_colorMap = g_pEffect->GetVariableByName("g_Ref_Tertiary_colorMap")->AsShaderResource();
    g_Ref_Quaternary_effectSRV_colorMap = g_pEffect->GetVariableByName("g_Ref_Quaternary_colorMap")->AsShaderResource();


    //Initialize
    g_Ref_Primary_pColorMap_RenderingClass = new RenderTexture;
    g_Ref_Primary_pColorMap_RenderingClass->Initialize(pd3dDevice, SCREENWIDTH, SCREENHEIGHT);

    g_Ref_Secondary_pColorMap_RenderingClass = new RenderTexture;
    g_Ref_Secondary_pColorMap_RenderingClass->Initialize(pd3dDevice, SCREENWIDTH, SCREENHEIGHT);

    g_Ref_Tertiary_pColorMap_RenderingClass = new RenderTexture;
    g_Ref_Tertiary_pColorMap_RenderingClass->Initialize(pd3dDevice, SCREENWIDTH, SCREENHEIGHT);

    g_Ref_Quaternary_pColorMap_RenderingClass = new RenderTexture;
    g_Ref_Quaternary_pColorMap_RenderingClass->Initialize(pd3dDevice, SCREENWIDTH, SCREENHEIGHT);

    g_Ref_pDepthBuffer_RenderingClass = new RenderTexture;
    g_Ref_pDepthBuffer_RenderingClass->Initialize(pd3dDevice, SCREENWIDTH, SCREENHEIGHT);


    const XMVECTORF32 vecEye = { 5.0f, -26.f, 16.f, 0.f };
    const XMVECTORF32 vecAt = { 17.0f, -26.f, 0.f, 0.f };
    g_Camera.SetViewParams(vecEye, vecAt);

    return hr;
}

void LoadModels(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext)
{
    g_pFSQ->MakeLoadModel("./Models/city/FSQ.obj", pd3dDevice, pd3dImmediateContext);
    g_pModel_1->MakeLoadModel("./Objects/cars/car_1.obj", pd3dDevice, pd3dImmediateContext);
    g_pModel_2->MakeLoadModel("./Objects/cars/car_2.obj", pd3dDevice, pd3dImmediateContext);
    g_pModel_3->MakeLoadModel("./Objects/cars/car_3.obj", pd3dDevice, pd3dImmediateContext);
    g_pModel_4->MakeLoadModel("./Objects/cars/car_4.obj", pd3dDevice, pd3dImmediateContext);
    g_pBackgroundModel->MakeLoadModel("./Objects/city/123123444.obj", pd3dDevice, pd3dImmediateContext);
}


```

<br>


지정한 레이어에 맞게 렌더링을 진행, 행렬을 업데이트하여 물체의 위치를 변화시키고 다시 렌더링을 반복한다. 이러한 작업이 끝난 후 셰이더를 호출한다.

```c++


void CALLBACK OnD3D11FrameRender(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext, double fTime, float fElapsedTime, void* pUserContext)
{
    HRESULT hr;

    D3D11_VIEWPORT mViewport;
    ZeroMemory(&mViewport, sizeof(mViewport));
    mViewport.TopLeftX = 0.0f;
    mViewport.TopLeftY = 0.0f;
    mViewport.Width = static_cast<float>(1920);
    mViewport.Height = static_cast<float>(1080);
    mViewport.MinDepth = 0.0f;
    mViewport.MaxDepth = 1.0f;
    pd3dImmediateContext->RSSetViewports(1, &mViewport);

    float ClearColor[4] = { 0.0f, 0.0f, 0.0f, 0.0f };

    g_pTech = g_pEffect->GetTechniqueByName("ReferenceRendering");
    g_pTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);

    for (int layer = 0; layer < 8; layer++)
    {
        pd3dImmediateContext->ClearRenderTargetView(g_Ref_Primary_pColorMap_RenderingClass->ppRTVs[layer], ClearColor);
        pd3dImmediateContext->ClearDepthStencilView(g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView(), D3D11_CLEAR_DEPTH, 1.0, 0);
        pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Primary_pColorMap_RenderingClass->ppRTVs[layer], NULL);
        pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Primary_pColorMap_RenderingClass->ppRTVs[layer], g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView());
        renderPerObjects(pd3dImmediateContext);
    }

    g_Ref_Primary_effectSRV_colorMap->SetResource(g_Ref_Primary_pColorMap_RenderingClass->ppSRVs[0]);

    if (MAXSUBFRAME > 8)
    {
        for (int layer = 0; layer < 8; layer++)
        {
            pd3dImmediateContext->ClearRenderTargetView(g_Ref_Secondary_pColorMap_RenderingClass->ppRTVs[layer], ClearColor);
            pd3dImmediateContext->ClearDepthStencilView(g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView(), D3D11_CLEAR_DEPTH, 1.0, 0);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Secondary_pColorMap_RenderingClass->ppRTVs[layer], NULL);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Secondary_pColorMap_RenderingClass->ppRTVs[layer], g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView());
            renderPerObjects(pd3dImmediateContext);

        }

        g_Ref_Secondary_effectSRV_colorMap->SetResource(g_Ref_Secondary_pColorMap_RenderingClass->ppSRVs[0]);
    }

    if (MAXSUBFRAME > 16)
    {
        for (int layer = 0; layer < 8; layer++)
        {
            pd3dImmediateContext->ClearRenderTargetView(g_Ref_Tertiary_pColorMap_RenderingClass->ppRTVs[layer], ClearColor);
            pd3dImmediateContext->ClearDepthStencilView(g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView(), D3D11_CLEAR_DEPTH, 1.0, 0);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Tertiary_pColorMap_RenderingClass->ppRTVs[layer], NULL);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Tertiary_pColorMap_RenderingClass->ppRTVs[layer], g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView());
            renderPerObjects(pd3dImmediateContext);

        }

        g_Ref_Tertiary_effectSRV_colorMap->SetResource(g_Ref_Tertiary_pColorMap_RenderingClass->ppSRVs[0]);
    }

    if (MAXSUBFRAME > 24)
    {
        for (int layer = 0; layer < 8; layer++)
        {
            pd3dImmediateContext->ClearRenderTargetView(g_Ref_Quaternary_pColorMap_RenderingClass->ppRTVs[layer], ClearColor);
            pd3dImmediateContext->ClearDepthStencilView(g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView(), D3D11_CLEAR_DEPTH, 1.0, 0);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Quaternary_pColorMap_RenderingClass->ppRTVs[layer], NULL);
            pd3dImmediateContext->OMSetRenderTargets(1, &g_Ref_Quaternary_pColorMap_RenderingClass->ppRTVs[layer], g_Ref_pDepthBuffer_RenderingClass->GetDepthStencilView());
            renderPerObjects(pd3dImmediateContext);

        }

        g_Ref_Quaternary_effectSRV_colorMap->SetResource(g_Ref_Quaternary_pColorMap_RenderingClass->ppSRVs[0]);
    }

//-----------------------------------------------------------------------------
// 	   Reference: MakeMotionBlur
//-----------------------------------------------------------------------------

    ID3D11DepthStencilView* FQ_DSV = DXUTGetD3D11DepthStencilView();
    ID3D11RenderTargetView* pRTV = DXUTGetD3D11RenderTargetView();

    pd3dImmediateContext->ClearRenderTargetView(pRTV, ClearColor);
    pd3dImmediateContext->ClearDepthStencilView(FQ_DSV, D3D11_CLEAR_DEPTH, 1.0f, 0);
    pd3dImmediateContext->OMSetRenderTargets(1, &pRTV, FQ_DSV);

    g_pTech = g_pEffect->GetTechniqueByName("Ref_MotionBlur");
    g_pTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    pd3dImmediateContext->Draw(3, 0);
}

void renderPerObjects(ID3D11DeviceContext* pd3dImmediateContext)
{
    static float movePos_1 = 50.f;
    static float movePos_2 = 50.f;;
    static float movePos_3 = 50.f;;
    static float movePos_4 = -50.f;

    if (g_isMove)
    {
        movePos_1 = g_pModel_1->SetMovePosition(movePos_1, 0.5f / MAXSUBFRAME, 50.f, -50.f);
        movePos_2 = g_pModel_2->SetMovePosition(movePos_2, 0.5f / MAXSUBFRAME, 50.f, -50.f);
        movePos_3 = g_pModel_3->SetMovePosition(movePos_3, 0.5f / MAXSUBFRAME, 50.f, -50.f);
        movePos_4 = g_pModel_4->SetMovePosition(movePos_4, 0.5f / MAXSUBFRAME, -50.f, 50.f);
    }


    g_pModel_1->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 0.02f, -movePos_1, -0.0f, -0.f);
    g_pModel_2->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 0.02f, -movePos_2, -0.0f, -0.f);
    g_pModel_3->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 0.02f, -movePos_3, -0.0f, -0.f);
    g_pModel_4->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 0.02f, -movePos_4, -0.0f, -0.f);
    g_pBackgroundModel->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 0.02f, 0.0f, 0.0f, 0.f);

    g_pModel_1->Render(pd3dImmediateContext, g_pLayout, g_pTech, techDesc, g_pEffect);
    g_pModel_2->Render(pd3dImmediateContext, g_pLayout, g_pTech, techDesc, g_pEffect);
    g_pModel_3->Render(pd3dImmediateContext, g_pLayout, g_pTech, techDesc, g_pEffect);
    g_pModel_4->Render(pd3dImmediateContext, g_pLayout, g_pTech, techDesc, g_pEffect);
    g_pBackgroundModel->Render(pd3dImmediateContext, g_pLayout, g_pTech, techDesc, g_pEffect);

    g_pModel_1->AfterFrameRender();
    g_pModel_2->AfterFrameRender();
    g_pModel_3->AfterFrameRender();
    g_pModel_4->AfterFrameRender();
    g_pBackgroundModel->AfterFrameRender();
}

```

<br>


이렇게 찍은 것들은 개별 렌더 텍스처에 그려지며 셰이더로 넘어간다.

```c++

class RenderObject: public Model
{
public:
	RenderObject();
	RenderObject(const RenderObject&);

	float SetMovePosition(float sphereMovePosition, float displacement, float startPoint, float destinationPoint);
	void BeforeFrameRender(XMMATRIX viewMatrix, XMMATRIX projectionMatrix, float scale, float xPos, float yPos, float zPos);
	void AfterFrameRender();
	void Render(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect);
	void RenderFSQ(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect);

	XMMATRIX m_scale;
	XMMATRIX m_rotation;
	XMMATRIX m_world;
	XMMATRIX m_worldView;
	XMMATRIX m_worldViewProjection;
	XMMATRIX m_previousWVP;
	XMMATRIX m_translation;

	ID3DX11EffectMatrixVariable* m_effect_world = NULL;
	ID3DX11EffectMatrixVariable* m_effect_worldView = NULL;
	ID3DX11EffectMatrixVariable* m_effect_worldViewProjection = NULL;
	ID3DX11EffectMatrixVariable* m_effect_PreviousWVP = NULL;
	ID3DX11EffectShaderResourceVariable* m_effect_texture = NULL;

	float xPos, yPos, zPos;

private:

};

```

<br>

```c++
#include<RenderObject.h>

RenderObject::RenderObject()
{
}

RenderObject::RenderObject(const RenderObject& other)
{
}

void RenderObject::BeforeFrameRender(XMMATRIX viewMatrix, XMMATRIX projectionMatrix, float scale, float xPos, float yPos, float zPos)
{
	m_translation = XMMatrixTranslation(xPos, yPos, zPos);
	m_scale = XMMatrixScaling(scale, scale, scale);
	m_rotation = XMMatrixRotationY(XMConvertToRadians(0.f));
	m_world = m_scale * m_translation;
	m_worldView = m_world * viewMatrix;
	m_worldViewProjection = m_world * viewMatrix * projectionMatrix;
}

void RenderObject::AfterFrameRender()
{
	m_previousWVP = m_worldViewProjection;
}


float RenderObject::SetMovePosition(float sphereMovePosition, float displacement, float startPoint, float destinationPoint)
{
	if (startPoint < destinationPoint)
	{
		if (sphereMovePosition <= destinationPoint)
		{
			sphereMovePosition += displacement;
		}
		
		else
		{
			sphereMovePosition = startPoint;
		}
	}

	else
	{
		if (sphereMovePosition >= destinationPoint)
		{
			sphereMovePosition -= displacement;
		}

		else
		{
			sphereMovePosition = startPoint;
		}

	}

	return sphereMovePosition;
}


void RenderObject::Render(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect)
{
	m_effect_world = g_pEffect->GetVariableByName("g_world")->AsMatrix();
	m_effect_worldView = g_pEffect->GetVariableByName("g_worldView")->AsMatrix();
	m_effect_worldViewProjection = g_pEffect->GetVariableByName("g_worldViewProjection")->AsMatrix();
	m_effect_PreviousWVP = g_pEffect->GetVariableByName("g_previousWorldViewProjection")->AsMatrix();
	m_effect_texture = g_pEffect->GetVariableByName("g_txDiffuse")->AsShaderResource();

	XMFLOAT4X4 tmp4x4;
	XMStoreFloat4x4(&tmp4x4, m_worldViewProjection);
	m_effect_worldViewProjection->SetMatrix(reinterpret_cast<float*>(&tmp4x4));
	XMStoreFloat4x4(&tmp4x4, m_previousWVP);
	m_effect_PreviousWVP->SetMatrix(reinterpret_cast<float*>(&tmp4x4));
	XMStoreFloat4x4(&tmp4x4, m_world);
	m_effect_world->SetMatrix(reinterpret_cast<float*>(&tmp4x4));
	XMStoreFloat4x4(&tmp4x4, m_worldView);
	m_effect_worldView->SetMatrix(reinterpret_cast<float*>(&tmp4x4));

	g_pTech->GetDesc(&techDesc);
	pd3dImmediateContext->IASetInputLayout(g_pLayout);
	Draw(m_effect_texture, g_pTech, techDesc , pd3dImmediateContext, g_pLayout);

}

void RenderObject::RenderFSQ(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect)
{
	g_pTech->GetDesc(&techDesc);
	pd3dImmediateContext->IASetInputLayout(g_pLayout);
	DrawFSQ(m_effect_texture, g_pTech, techDesc, pd3dImmediateContext, g_pLayout);
}
```


<br>

셰이더에서 technique11 ReferenceRendering에서 하나의 이미지가 생성되고 technique11 Ref_MotionBlur에서 구한 이미지의 색상 평균을 구해, 하나의 모션블러 이미지가 생성된다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e33c9c9b-cfe8-4736-8609-81277e088aef" alt width=700>
<em></em>
</center>


```

cbuffer cbPerObject : register(b0)
{
	matrix             g_worldViewProjection;
	matrix			   g_previousWorldViewProjection; //previous only this  ,other all current//g_mPreviousWorldViewProjection
	matrix			   g_MVPInverseTransposed;
	matrix             g_worldView;
	matrix             g_world;
};


SamplerState    g_samLinear
{
	Filter = MIN_MAG_MIP_POINT;
	AddressU = WRAP;
	AddressV = WRAP;
	AddressW = WRAP;

};

RasterizerState rasterizerState_New
{
	FillMode = SOLID;
	CullMode = NONE;
};

BlendState NoMRT_BS
{
	RenderTargetWriteMask[0] = 0x0f;
};

DepthStencilState DepthNoStencil_DS
{
	DepthEnable = true;
	DepthFunc = Less_Equal;
	DepthWriteMask = All;
	StencilEnable = false;
};

RasterizerState BackCullNoMSAA_RS
{
	CullMode = Back;
	FillMode = Solid;
	MultisampleEnable = false;
};


//-----------------------------------------------------------------------------
// 	   Render reference per layer
//-----------------------------------------------------------------------------


Texture2D<float4>		g_txDiffuse;

struct VS_INPUT
{
	float4 Position    : POSITION; // vertex position 	
	float2 TextureUV   : TEXCOORD0;
};

struct VS_OUTPUT
{
	float4 Position     : SV_POSITION;
	float2 TexCoord     : TEXCOORD0;
};

VS_OUTPUT ReferenceRendering_VS(VS_INPUT InputVS)
{
	VS_OUTPUT OutputVS = (VS_OUTPUT)0;

	OutputVS.Position = mul(InputVS.Position, g_previousWorldViewProjection);
	OutputVS.TexCoord = InputVS.TextureUV;

	return OutputVS;
}

float4 ReferenceRendering_PS(VS_OUTPUT InputPS) : SV_TARGET
{
	float4 color = g_txDiffuse.Sample(g_samLinear, InputPS.TexCoord , 0);
	return color;
}

technique11 ReferenceRendering
{
	pass P0
	{
		SetVertexShader(CompileShader(vs_5_0, ReferenceRendering_VS()));
		SetGeometryShader(NULL);
		SetPixelShader(CompileShader(ps_5_0, ReferenceRendering_PS()));

		SetRasterizerState(BackCullNoMSAA_RS);
		SetDepthStencilState(DepthNoStencil_DS, 0x00000000);
		SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
	}
}


//-----------------------------------------------------------------------------
// 	   MakeMotionBlur
//-----------------------------------------------------------------------------


Texture2DArray<float4>	g_Ref_Primary_colorMap;
Texture2DArray<float4>	g_Ref_Secondary_colorMap;
Texture2DArray<float4>	g_Ref_Tertiary_colorMap;
Texture2DArray<float4>	g_Ref_Quaternary_colorMap;

struct QuadOutput
{
	float4 pos : SV_Position;
	float2 tex : TEXCOORD0;
};

QuadOutput FullScreenTriVS(uint id : SV_VertexID) //꼭지점 INDEX 0,1,2
{
	QuadOutput output = (QuadOutput)0.0f;
	output.tex = float2((id << 1) & 2, id & 2);
	output.pos = float4(output.tex * float2(2.0f, -2.0f) + float2(-1.0f, 1.0f), 0.0f, 1.0f);

	return output;
}

float4 Ref_MotionBlurPS(QuadOutput IN) : SV_TARGET
{
	float4 color = float4(0.0f, 0.0f, 0.0f,0.0f);

	for (int i = 0; i < 4; i++) 
	{
		color += float4(g_Ref_Primary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
	}

	if (MAXSUBFRAME > 8)
	{
		for (int i = 0; i < 8; i++)
		{
			color += float4(g_Ref_Secondary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
		}
	}
	
	if (MAXSUBFRAME > 16)
	{
		for (int i = 0; i < 8; i++)
		{
			color += float4(g_Ref_Tertiary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
		}
	}

	if (MAXSUBFRAME > 24)
	{
		for (int i = 0; i < 8; i++)
		{
			color += float4(g_Ref_Quaternary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
		}
	}

	color = color / MAXSUBFRAME;
	return color;
}

technique11 Ref_MotionBlur
{
	pass P0
	{
		SetVertexShader(CompileShader(vs_5_0, FullScreenTriVS()));
		SetGeometryShader(NULL);
		SetPixelShader(CompileShader(ps_5_0, Ref_MotionBlurPS()));
		SetRasterizerState(rasterizerState_New);
	}
}
```



<br>
<br>








# 출력
---



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/70c4144b-5b4e-449c-9673-8be76e17b2e7" alt width=600>
<em>모션블러가 적용된 이미지</em>
</center>


