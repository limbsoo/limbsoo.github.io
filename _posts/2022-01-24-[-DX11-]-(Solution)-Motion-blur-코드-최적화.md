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
  - motionblur
---
# 문제
___


기존 코드는 pd3dDevice에 만들 물체의 개수만큼 buffer를 생성하여 순차적으로 그린다. 이는 모델이 추가될 때마다 코드 수정이 많이 필요하다.


```c++

ID3D11Buffer* g_pcbVSPerObject_1 = NULL;
ID3D11Buffer* g_pcbVSPerObject_2 = NULL;
ID3D11Buffer* g_pcbVSPerFrame_1 = NULL;
ID3D11Buffer* g_pcbVSPerFrame_2 = NULL;
Animation* g_pModel_1 = NULL;
Animation* g_pModel_2 = NULL;
ID3D11ShaderResourceView* g_pTextures_1 = NULL;
ID3D11ShaderResourceView* g_pTextures_2 = NULL;
...

//--------------------------------------------------------------------------------------
// Create any D3D resources that aren't dependant on the back buffer
//--------------------------------------------------------------------------------------
HRESULT CALLBACK OnD3D11CreateDevice(ID3D11Device* pd3dDevice, const DXGI_SURFACE_DESC* pBackBufferSurfaceDesc,void* pUserContext)
{
    HRESULT hr = S_OK;

    auto pd3dImmediateContext = DXUTGetD3D11DeviceContext();
    V_RETURN(g_DialogResourceManager.OnD3D11CreateDevice(pd3dDevice, pd3dImmediateContext));
    V_RETURN(g_D3DSettingsDlg.OnD3D11CreateDevice(pd3dDevice));
    g_pTxtHelper = new CDXUTTextHelper(pd3dDevice, pd3dImmediateContext, &g_DialogResourceManager, 15);

    // Compile and create the effect.
    DWORD dwShaderFlags = D3DCOMPILE_ENABLE_STRICTNESS;
    dwShaderFlags |= D3DCOMPILE_DEBUG;
    dwShaderFlags |= D3DCOMPILE_SKIP_OPTIMIZATION;

    hr = D3DX11CompileFromFile(L"DynamicShaderLinkageFX11.fx", 0, 0, 0, "fx_5_0", dwShaderFlags, 0, 0, &effectBlob, &errorsBlob, 0);

    if (errorsBlob)
    {
        MessageBoxA(0, (char*)errorsBlob->GetBufferPointer(), 0, 0);
        errorsBlob->Release();
    }

    hr = D3DX11CreateEffectFromMemory(effectBlob->GetBufferPointer(), effectBlob->GetBufferSize(), 0, pd3dDevice, &g_pEffect);
    assert(SUCCEEDED(hr));
    effectBlob->Release();
    
    g_pTextureClear_Tech = g_pEffect->GetTechniqueByName("TextureClear");
    g_pStencilBufferClear_Tech = g_pEffect->GetTechniqueByName("StencilBufferClear");
    g_pRendering_Tech = g_pEffect->GetTechniqueByName("Rendering");
    g_motionVectorMomentMap_Rendering_Tech = g_pEffect->GetTechniqueByName("motionVectorMomentMap_Rendering");
    g_pSorting_Tech = g_pEffect->GetTechniqueByName("Sorting");
    g_pMakeMotionBlur_Tech = g_pEffect->GetTechniqueByName("MakeMotionBlur");
    g_pDenoising_Tech = g_pEffect->GetTechniqueByName("Denoising");

    D3DX11_PASS_DESC passDesc;
    g_pRendering_Tech->GetPassByIndex(0)->GetDesc(&passDesc);

    const D3D11_INPUT_ELEMENT_DESC layout[] =
    {
        { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0 },
        { "NORMAL", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12, D3D11_INPUT_PER_VERTEX_DATA, 0 },
        { "TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT, 0, 24, D3D11_INPUT_PER_VERTEX_DATA, 0 },
    };

    V_RETURN(pd3dDevice->CreateInputLayout(layout, ARRAYSIZE(layout), passDesc.pIAInputSignature, passDesc.IAInputSignatureSize, &g_pLayout));

    // Create constant buffers of Main Object 1
    D3D11_BUFFER_DESC cbDesc_1;
    ZeroMemory(&cbDesc_1, sizeof(cbDesc_1));
    cbDesc_1.Usage = D3D11_USAGE_DYNAMIC;
    cbDesc_1.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
    cbDesc_1.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
    cbDesc_1.ByteWidth = sizeof(CB_VS_PER_OBJECT);
    V_RETURN(pd3dDevice->CreateBuffer(&cbDesc_1, NULL, &g_pcbVSPerObject_1));
    DXUT_SetDebugName(g_pcbVSPerObject_1, "CB_VS_PER_OBJECT");
    cbDesc_1.ByteWidth = sizeof(CB_VS_PER_Frame);
    V_RETURN(pd3dDevice->CreateBuffer(&cbDesc_1, NULL, &g_pcbVSPerFrame_1));
    DXUT_SetDebugName(g_pcbVSPerFrame_1, "CB_VS_PER_FRAME");

    // Create constant buffers of Main Object 2
    D3D11_BUFFER_DESC cbDesc_2;
    ZeroMemory(&cbDesc_2, sizeof(cbDesc_2));
    cbDesc_2.Usage = D3D11_USAGE_DYNAMIC;
    cbDesc_2.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
    cbDesc_2.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
    cbDesc_2.ByteWidth = sizeof(CB_VS_PER_OBJECT);
    V_RETURN(pd3dDevice->CreateBuffer(&cbDesc_2, NULL, &g_pcbVSPerObject_2));
    DXUT_SetDebugName(g_pcbVSPerObject_2, "CB_VS_PER_OBJECT");
    cbDesc_2.ByteWidth = sizeof(CB_VS_PER_Frame);
    V_RETURN(pd3dDevice->CreateBuffer(&cbDesc_2, NULL, &g_pcbVSPerFrame_2));
    DXUT_SetDebugName(g_pcbVSPerFrame_2, "CB_VS_PER_FRAME");

    LoadModels(pd3dDevice);
    LoadTextures(pd3dDevice, pd3dImmediateContext);
}


void LoadModels(ID3D11Device* pd3dDevice)
{
    g_pModel_1 = new Animation();
    g_pModel_2 = new Animation();
    g_pModel_1->LoadMesh("./models/Sphere/sphere.x", pd3dDevice);
    g_pModel_2->LoadMesh("./models/sphere/sphere.x", pd3dDevice);
}

HRESULT LoadTextures(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext)
{
    HRESULT hr = S_OK;
    V_RETURN(DXUTCreateShaderResourceViewFromFile(pd3dDevice, L"./models/Sphere/billiard13.jpg", &g_pTextures_1));
    V_RETURN(DXUTCreateShaderResourceViewFromFile(pd3dDevice, L"./models/Sphere/red.jpg", &g_pTextures_2));
}


void RenderObject(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext, ID3DX11EffectTechnique* pTech)
{
    // Prepare for Motion of object
    const XMMATRIX mView = g_Camera.GetViewMatrix();
    const XMMATRIX mProj = g_Camera.GetProjMatrix();
    const XMMATRIX scale = XMMatrixScaling(3.5f, 3.0f, 3.0f);

    const XMMATRIX translation1 = XMMatrixTranslation(movePos_1.x, movePos_1.y, movePos_1.z);
    const XMMATRIX mWorld1 = scale * translation1;
    g_WVP_1b = mWorld1 * mView * mProj;
    SetEffect(g_WVP_1b, g_pTextures_1, g_mPreProjectionMatrix1);
    RenderObject(g_pModel_1, pd3dImmediateContext, g_pLayout, pTech);

    const XMMATRIX translation2 = XMMatrixTranslation(movePos_2.x, movePos_2.y, movePos_2.z);
    const XMMATRIX mWorld2 = scale * translation2;
    g_WVP_2b = mWorld2 * mView * mProj;
    SetEffect(g_WVP_2b, g_pTextures_2, g_mPreProjectionMatrix2);
    RenderObject(g_pModel_2, pd3dImmediateContext, g_pLayout, pTech);
}
```



그리고 렌더링이 1번 진행될 때는 괜찮으나, 여러 번 진행 시 매트릭스가 여러 번 업데이트되어 오류가 발생하기 쉽다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/17a40eee-98b8-40ea-9f52-fc45339714ef" alt width=500>
<em></em>
</center>









<br>


# 해결
___

RenderObject Class로 그룹을 생성, 

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

	void RenderSkybox(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect);
	void BeforeFrameRenderSkybox(XMMATRIX viewMatrix, XMMATRIX projectionMatrix, float scale, float xPos, float yPos, float zPos);


	XMMATRIX m_scale;
	XMMATRIX m_rotation;
	XMMATRIX m_world;
	XMMATRIX m_worldView;
	XMMATRIX m_worldViewProjection;
	XMMATRIX m_previousWVP;
	XMMATRIX m_translation;

	ID3DX11EffectMatrixVariable* m_effect_worldViewProjection = NULL;
	ID3DX11EffectMatrixVariable* m_effect_PreviousWVP = NULL;
	ID3DX11EffectShaderResourceVariable* m_effect_texture = NULL;

	ID3DX11EffectMatrixVariable* m_effect_world = NULL;

	float xPos, yPos, zPos;

};


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


void RenderObject::Render(ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc, ID3DX11Effect* g_pEffect)
{
	m_effect_worldViewProjection = g_pEffect->GetVariableByName("g_worldViewProjection")->AsMatrix();
	m_effect_PreviousWVP = g_pEffect->GetVariableByName("g_previousWorldViewProjection")->AsMatrix();
	m_effect_texture = g_pEffect->GetVariableByName("g_modelTexture")->AsShaderResource();

	m_effect_world = g_pEffect->GetVariableByName("g_world")->AsMatrix();

	XMFLOAT4X4 tmp4x4;
	XMStoreFloat4x4(&tmp4x4, m_worldViewProjection);
	m_effect_worldViewProjection->SetMatrix(reinterpret_cast<float*>(&tmp4x4));
	XMStoreFloat4x4(&tmp4x4, m_previousWVP);
	m_effect_PreviousWVP->SetMatrix(reinterpret_cast<float*>(&tmp4x4));

	XMStoreFloat4x4(&tmp4x4, m_world);
	m_effect_world->SetMatrix(reinterpret_cast<float*>(&tmp4x4));

	g_pTech->GetDesc(&techDesc);
	pd3dImmediateContext->IASetInputLayout(g_pLayout);
	Draw(m_effect_texture, g_pTech, techDesc , pd3dImmediateContext, g_pLayout);
}


```


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7c82fa94-933a-431e-b838-208d9fa1d84a" alt width=500>
<em></em>
</center>


객체 별 렌더링을 구현,


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/dc508424-2173-47f1-ac2b-0c6497aa0f5d" alt width=500>
<em></em>
</center>

rendering pass 전에 매트릭스를 생성, pass가 완료되면 현재의 WVP가 previousWVP로 업데이트 된다.






