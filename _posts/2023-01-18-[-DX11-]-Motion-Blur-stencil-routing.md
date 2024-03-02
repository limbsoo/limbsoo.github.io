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

원래 샘플링은 픽셀 셰이더에서 픽셀 당 한 번 시도한다. 그러나 멀티 샘플링의 경우, 지정 횟수 만큼의 샘플링을 시도하는 방식이다. 따라서 멀티 샘플링을 사용하는 경우 MSAA buffer가 생성되는데, 이를 활용하여 서로 다른 여러 이미지를 출력할 수 있다.

그리고 이러한 방법은 기존 a buffer를 사용하는 것보다 높은 성능을 보여주기에, 이를 여러 이미지를 사용하는 모션 블러에 적용하였다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/f4daa9a5-8aca-4633-963c-7a6987a5cc5b" alt width=700>
<em></em>
</center>




아래 과정을 통해 모션 블러 이미지를 생성한다.
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/763e46e9-1c8a-4b36-b1f1-d75a9edc4dd0" alt width=700>
<em></em>
</center>



# 구현
---



fullscreen quads와 스텐실 마스크를 사용, 4개의 서브 샘플(4xMSAA)이 있는 단일 픽셀에 대해 스텐실 버퍼를 다음과 같이 초기화한다고 가정했을 때

1 2  
3 4

조각이 이 픽셀에 도달할 때 스텐실 작업을 D3D11 STENCIL OP DECR SAT로 설정하면 스텐실이 다음과 같이 변경

0 1  
2 3

스텐실 테스트를 참조 값이 2인 D3D11 COMPARISON EQUAL로 설정하면 1인 서브 샘플만 사용, 이를 통해 다중 샘플 색상 텍스처의 하위 샘플로 조각을 라우팅(4xMSAA에서 픽셀당 4개의 조각)할 수 있다. 


Depth Peeling by Stencil Routed Rendering

- Turn off Depth Test & Set multiple render target(renderTarget은 각각의 Stencil buffer와 연결)

- Stencil buffer를 (StencilRef ~ StencilRef + renderTarget - 1)로 초기화

- polygon이 픽셀을 cover할 때 마다 stencilRef와 똑같은 stencil 값을 가지는 renderTarget에만 색상을 저장하고 모든 renderTarget의 stencil값 1 감소

  여러 개의 polygon이 동일한 픽셀을 cover하면 renderTarget 만큼 polygon의 색상이 renderTarget에 저장된다.

```
if(StencilRef == stencilBuffer)
   rendering & stencil decrease
else
    stencil decrease
```


그리고 각 픽셀을 depth 값에 의한 bitonic sort를 통해 정렬, depth 별 이미지를 생성한다.


```c++


void StencilBuffer(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext) {
    const float ClearColor[4] = { 0.0f, 0.0f, 0.0f, 0.0f };

    ID3D11RenderTargetView* pMSAARTV = g_color_Depth_motionVector_Class->GetRenderTargetView();
    pd3dImmediateContext->OMSetRenderTargets(1, &pMSAARTV, NULL);

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("TextureClear");
    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    pd3dImmediateContext->Draw(3, 0);

    //------------------------------------------------------------- stencil buffer clear
    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("StencilBufferClear");
    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    BeginStackRender(pd3dImmediateContext, &pMSAARTV, g_color_Depth_motionVector_Class.get(), 1, g_pStencilBufferTech);

    //------------------------------------------------------------- rendering (stencil buffer use)

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("Rendering");
    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    renderPerObjects(pd3dImmediateContext, g_pStencilBufferTech, g_pStencilBufferEffect);

    //------------------------------------------------------------- rendering

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("motionVectorMipMap_colorMap_Rendering");
    pd3dImmediateContext->OMSetDepthStencilState(g_motionVectorMipMap_Class->GetUseDepthTestDSS(), 1);

    ID3D11RenderTargetView* pmotionVectorMomentMapRTV[4];
    pmotionVectorMomentMapRTV[0] = g_motionVectorMipMap_Class->GetRenderTargetView();
    pmotionVectorMomentMapRTV[1] = g_firstLayerColor_Class->GetRenderTargetView();
    pmotionVectorMomentMapRTV[2] = g_motionVector_Class->GetRenderTargetView();
    pmotionVectorMomentMapRTV[3] = g_depth_Class.get()->GetRenderTargetView();;

    ID3D11DepthStencilView* pmotionVectorMomentMapDSV = g_motionVectorMipMap_Class->GetDepthStencilView();

    pd3dImmediateContext->ClearRenderTargetView(pmotionVectorMomentMapRTV[0], ClearColor);
    pd3dImmediateContext->ClearRenderTargetView(pmotionVectorMomentMapRTV[1], ClearColor);
    pd3dImmediateContext->ClearRenderTargetView(pmotionVectorMomentMapRTV[2], ClearColor);
    pd3dImmediateContext->ClearRenderTargetView(pmotionVectorMomentMapRTV[3], ClearColor);

    pd3dImmediateContext->ClearDepthStencilView(pmotionVectorMomentMapDSV, D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL, 1.0f, 0);

    pd3dImmediateContext->OMSetRenderTargets(4, pmotionVectorMomentMapRTV, pmotionVectorMomentMapDSV);

    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    boxDrawing = true;
    renderPerObjects(pd3dImmediateContext, g_pStencilBufferTech, g_pStencilBufferEffect);

    pd3dImmediateContext->GenerateMips(g_motionVectorMipMap_Class->GetShaderResourceView());

    //------------------------------------------------------------- sorting

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("Sorting");
    ID3D11RenderTargetView* pSortedRTV[MSAA_LEVEL] = {};
    for (int i = 0; i < MSAA_LEVEL; i++) pSortedRTV[i] = g_sorted_Color_Depth_MotionVector_Class->GetRenderTargetViewArray(i);

    pd3dImmediateContext->OMSetRenderTargets(MSAA_LEVEL, pSortedRTV, NULL);

    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    pd3dImmediateContext->Draw(3, 0);

    //------------------------------------------------------------- motion blur

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("MakeMotionBlur");
    ID3D11RenderTargetView* pMotionVectorRTV[2] = {};

    pMotionVectorRTV[0] = g_MotionBlurColor_Class->GetRenderTargetView();
    pMotionVectorRTV[1] = g_MotionBlurDepthWeight_Class->GetRenderTargetView();

    pd3dImmediateContext->OMSetRenderTargets(2, pMotionVectorRTV, NULL);

    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    pd3dImmediateContext->Draw(3, 0);

    //------------------------------------------------------------- denosing

    g_pStencilBufferTech = g_pStencilBufferEffect->GetTechniqueByName("Denoising");
    ID3D11RenderTargetView* pDXUTRTV = DXUTGetD3D11RenderTargetView();

    pd3dImmediateContext->OMSetRenderTargets(1, &pDXUTRTV, NULL);

    g_pStencilBufferTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    pd3dImmediateContext->Draw(3, 0);



}

void BeginStackRender(ID3D11DeviceContext* pd3dDeviceContext, ID3D11RenderTargetView** ppRTVs, RenderTexture* RenderClass, UINT nRTV, ID3DX11EffectTechnique* tech)
{
    tech->GetPassByIndex(0)->Apply(0, pd3dDeviceContext);

    pd3dDeviceContext->IASetInputLayout(NULL);
    pd3dDeviceContext->RSSetState(RenderClass->GetRasterizerState());

    pd3dDeviceContext->OMSetRenderTargets(0, NULL, RenderClass->GetDepthStencilView());

    UINT8 nStencilRef = MSAA_LEVEL;

    const FLOAT blendFactor[4] = { 0.0f, 0.0f, 0.0f, 0.0f };
    UINT nSampleMask = 1;

    pd3dDeviceContext->ClearDepthStencilView(RenderClass->GetDepthStencilView(), D3D11_CLEAR_STENCIL, 1.0f, nStencilRef);
    for (UINT8 i = 1; i < MSAA_LEVEL; ++i)
    {
        --nStencilRef;
        nSampleMask = nSampleMask << 1;

        pd3dDeviceContext->OMSetBlendState(RenderClass->GetBlendState(), blendFactor, nSampleMask);
        pd3dDeviceContext->OMSetDepthStencilState(RenderClass->GetWriteDepthStencilState(), nStencilRef);
        pd3dDeviceContext->Draw(3, 0);
    }

    pd3dDeviceContext->OMSetRenderTargets(nRTV, ppRTVs, RenderClass->GetDepthStencilView());
    pd3dDeviceContext->OMSetDepthStencilState(RenderClass->GetUseDepthStencilState(), 0x00000001);
}

void StencilBuffer_SetResouce() {
    g_color_Depth_motionVectorMap[0] = g_pStencilBufferEffect->GetVariableByName("g_color_Depth_motionVectorMap")->AsShaderResource();
    g_sorted_Color_Depth_MotionVectorMap = g_pStencilBufferEffect->GetVariableByName("g_sorted_Color_Depth_MotionVectorMap")->AsShaderResource();
    g_motionVectorMipMap = g_pStencilBufferEffect->GetVariableByName("g_motionVectorMipMap")->AsShaderResource();
    g_firstLayerColorMap = g_pStencilBufferEffect->GetVariableByName("g_firstLayerColorMap")->AsShaderResource();
    g_motionVectorMap = g_pStencilBufferEffect->GetVariableByName("g_motionVectorMap")->AsShaderResource();
    g_depthMap = g_pStencilBufferEffect->GetVariableByName("g_depthMap")->AsShaderResource();
    g_MotionBlurColorMap = g_pStencilBufferEffect->GetVariableByName("g_MotionBlurColorMap")->AsShaderResource();
    g_MotionBlurDepthWeightMap = g_pStencilBufferEffect->GetVariableByName("g_MotionBlurDepthWeightMap")->AsShaderResource();

    g_color_Depth_motionVectorMap[0]->SetResource(g_color_Depth_motionVector_Class->GetShaderResourceView());
    g_sorted_Color_Depth_MotionVectorMap->SetResource(g_sorted_Color_Depth_MotionVector_Class->GetShaderResourceViewArray(0));
    g_motionVectorMipMap->SetResource(g_motionVectorMipMap_Class->GetShaderResourceView());
    g_firstLayerColorMap->SetResource(g_firstLayerColor_Class->GetShaderResourceView());
    g_motionVectorMap->SetResource(g_motionVector_Class->GetShaderResourceView());
    g_MotionBlurColorMap->SetResource(g_MotionBlurColor_Class->GetShaderResourceView());
    g_MotionBlurDepthWeightMap->SetResource(g_MotionBlurDepthWeight_Class->GetShaderResourceView());
    g_depthMap->SetResource(g_depth_Class.get()->GetShaderResourceView());
}


```

```


#include "function.fx"

Texture2D g_modelTexture;

Texture2DMS<uint4, MSAA_LEVEL> g_color_Depth_motionVectorMap;

Texture2DArray<uint4> g_sorted_Color_Depth_MotionVectorMap;
Texture2DArray<uint4> g_denoised_Color_Depth_MotionVectorMap;
Texture2D g_motionVectorMipMap;
Texture2D g_firstLayerColorMap;
Texture2D g_motionVectorMap;
Texture2D g_depthMap;

Texture2D g_MotionBlurColorMap;
Texture2D g_MotionBlurDepthWeightMap;

//------------------------------------------------------------- stencil buffer clear pass

struct FSQVSOut
{
    float4 position : SV_POSITION;
    float2 uv : TEXCOORD0;
};

FSQVSOut FullScreenQuadVS(uint vertexID : SV_VertexID)
{
    FSQVSOut OutputVS;

    OutputVS.uv = float2((vertexID << 1) & 2, vertexID & 2);
    OutputVS.position = float4(OutputVS.uv * float2(2.0f, -2.0f) + float2(-1.0f, 1.0f), 0.0f, 1.0f);

    return OutputVS;
}

technique11 StencilBufferClear
{
    pass P0
    {
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(NULL);
    }
}

//------------------------------------------------------------- texture clear pass

uint4 ClearPS(FSQVSOut Input) : SV_TARGET
{
    return uint4(0, MAX_DEPTH, 2147450879, MAX_DEPTH / 2); //2147450879 == ((16bit)0 + (16bit)0), MAX_DEPTH/2 == (32bit)0
}

technique11 TextureClear
{
    pass P0
    {
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(CompileShader(ps_5_0, ClearPS()));
        SetRasterizerState(BackCullNoMSAA_RS);
        SetDepthStencilState(NoDepthNoStencil_DS, 0x00000000);
        SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
    }
}

//------------------------------------------------------------- multi Layer Rendering pass (stencil buffer) 

struct RenderingVSInput
{
    float4 Position : POSITION;
    float3 Normal : NORMAL;
    float2 uv : TEXCOORD0;
};

struct RenderingVSOut
{
    float4 position : SV_Position;
    float4 prePosition : POSITION1;
    float4 curPosition : POSITION2;
    float2 uv : TEXCOORD0;
};

RenderingVSOut RenderingVS(RenderingVSInput input)
{
    RenderingVSOut output;

    output.position = mul(input.Position, g_worldViewProjection);
    output.prePosition = mul(input.Position, g_previousWorldViewProjection);
    output.curPosition = mul(input.Position, g_worldViewProjection);

    output.uv = input.uv;

    return output;
}

uint4 RenderingPS(RenderingVSOut input) : SV_TARGET
{
    float3 currentPos = homogenious2uv(input.curPosition);
    float3 previousPos = homogenious2uv(input.prePosition);
    float3 motionVector = currentPos - previousPos;
    
    float3 color = float3(g_modelTexture.Sample(g_samPoint, input.uv).xyz);
    
    uint packed_Color = pack_rgb(color);
    uint packed_EyeZ = pack_depth(input.position.w);
    uint packed_MotionVector_XY = pack_MotionVector_XY(motionVector.xy);
    uint packed_MotionVector_Z = pack_MotionVector_Z(motionVector.z);
    
    return uint4(packed_Color, packed_EyeZ, packed_MotionVector_XY, packed_MotionVector_Z);
}

technique11 Rendering
{
    pass P0
    {
        SetRasterizerState(BackCullNoMSAA_RS);
        SetVertexShader(CompileShader(vs_5_0, RenderingVS()));
        SetPixelShader(CompileShader(ps_5_0, RenderingPS()));
        SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
    }
}

//------------------------------------------------------------- First Layer Rendering pass

struct motionVectorMipMap_colorMap_outPut
{
    float4 MotionVectorMipMap;
    float4 ColorMap;
    float4 MotionVectorMap;
    float4 depthMap;
};

RenderingVSOut motionVectorMipMap_colorMap_renderingVS(RenderingVSInput Input)
{
    RenderingVSOut OutputVS;

    OutputVS.position = mul(Input.Position, g_worldViewProjection);
    OutputVS.prePosition = mul(Input.Position, g_previousWorldViewProjection);
    OutputVS.curPosition = mul(Input.Position, g_worldViewProjection);

    OutputVS.uv = Input.uv;

    return OutputVS;
}

motionVectorMipMap_colorMap_outPut motionVectorMipMap_colorMap_renderingPS(RenderingVSOut input) : SV_TARGET
{
    motionVectorMipMap_colorMap_outPut output;
    
    float3 currentPos = homogenious2uv(input.curPosition);
    float3 previousPos = homogenious2uv(input.prePosition);
    float3 motionVector = currentPos - previousPos;
    
    output.MotionVectorMipMap = float4(motionVector.xy, motionVector.xy * motionVector.xy);
    output.ColorMap = g_modelTexture.Sample(g_samPoint, input.uv);
    output.MotionVectorMap = float4(motionVector, 0);
    output.depthMap = float4(input.position.w, input.position.w, input.position.w, 0);
    
    return output;
}

technique11 motionVectorMipMap_colorMap_Rendering
{
    pass P0
    {
        SetRasterizerState(NoCullNoMSAA_RS);
        SetVertexShader(CompileShader(vs_5_0, motionVectorMipMap_colorMap_renderingVS()));
        SetPixelShader(CompileShader(ps_5_0, motionVectorMipMap_colorMap_renderingPS()));
    }
}

//------------------------------------------------------------- sorting pass

struct FSQVSInput
{
    float4 Position : POSITION;
    float3 Normal : NORMAL;
    float2 uv : TEXCOORD0;
};

FSQVSOut FSQVS(FSQVSInput input)
{
    FSQVSOut OutputVS;

    OutputVS.position = input.Position;
    OutputVS.uv = input.uv;
    return OutputVS;
}

struct sortPSOutPut
{
    uint4 sorted_Layer_0;
    uint4 sorted_Layer_1;
    uint4 sorted_Layer_2;
    uint4 sorted_Layer_3;

    uint4 sorted_depth_0;
    uint4 sorted_depth_1;
    uint4 sorted_depth_2;
    uint4 sorted_depth_3;
    
    //uint4 sorted_Layer_4;
    //uint4 sorted_Layer_5;
    //uint4 sorted_Layer_6;
    //uint4 sorted_Layer_7;

};

sortPSOutPut SortingPS(FSQVSOut input) : SV_TARGET
{
    uint4 frag[8];
    float unpack_Depth[N_LAYER];
    [unroll]
    for (int i = 0; i < 8; ++i)
    {
        frag[i] = g_color_Depth_motionVectorMap.Load(input.position.xy, i);
    }
    BitonicSortF2B(frag, 8);

    float3 color = g_firstLayerColorMap.SampleLevel(g_samLinear, input.uv, 0).xyz;
    float3 motionVector = g_motionVectorMap.SampleLevel(g_samLinear, input.uv, 0).xyz;
    float dpeth = g_depthMap.SampleLevel(g_samLinear, input.uv, 0).x;

    uint packed_Color = pack_rgb(color);
    uint packed_EyeZ = pack_depth(dpeth);
    uint packed_MotionVector_XY = pack_MotionVector_XY(motionVector.xy);
    uint packed_MotionVector_Z = pack_MotionVector_Z(motionVector.z);

    sortPSOutPut output;
    output.sorted_Layer_0 = uint4(packed_Color, packed_EyeZ, packed_MotionVector_XY, packed_MotionVector_Z);
    //output.sorted_Layer_0 = frag[0];
    output.sorted_Layer_1 = frag[1];
    output.sorted_Layer_2 = frag[2];
    output.sorted_Layer_3 = frag[3];

    [unroll]
    for (int j = 0; j < N_LAYER; ++j)
    {
        unpack_Depth[j] = unpack_depth(frag[j].y);
    }

    output.sorted_depth_0 = uint4(pack_depth(unpack_Depth[0]), pack_depth(unpack_Depth[0] * unpack_Depth[0]), 0, 0);
    output.sorted_depth_1 = uint4(pack_depth(unpack_Depth[1]), pack_depth(unpack_Depth[1] * unpack_Depth[1]), 0, 0);
    output.sorted_depth_2 = uint4(pack_depth(unpack_Depth[2]), pack_depth(unpack_Depth[2] * unpack_Depth[2]), 0, 0);
    output.sorted_depth_3 = uint4(pack_depth(unpack_Depth[3]), pack_depth(unpack_Depth[3] * unpack_Depth[3]), 0, 0);
    //output.sorted_Layer_4 = frag[4];
    //output.sorted_Layer_5 = frag[5];
    //output.sorted_Layer_6 = frag[6];
    //output.sorted_Layer_7 = frag[7];
    
    //output.sorted_depth_0 = float4(unpack_depth(frag[0].y), unpack_depth(frag[0].y) * unpack_depth(frag[0].y), 0, 0);
    //output.sorted_depth_1 = float4(unpack_depth(frag[1].y), unpack_depth(frag[1].y) * unpack_depth(frag[1].y), 0, 0);
    //output.sorted_depth_2 = float4(unpack_depth(frag[2].y), unpack_depth(frag[2].y) * unpack_depth(frag[2].y), 0, 0);
    //output.sorted_depth_3 = float4(unpack_depth(frag[3].y), unpack_depth(frag[3].y) * unpack_depth(frag[3].y), 0, 0);

    return output;
}

technique11 Sorting
{
    pass P0
    {
        SetRasterizerState(NoCullNoMSAA_RS);
        SetDepthStencilState(NoDepthNoStencil_DS, 0x00000000);
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(CompileShader(ps_5_0, SortingPS()));
        SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
    }
}

//------------------------------------------------------------- denoising pass

struct denoisedPSOutPut
{
    uint4 denoised_Layer_0;
    uint4 denoised_Layer_1;
    uint4 denoised_Layer_2;
    uint4 denoised_Layer_3;
};

denoisedPSOutPut DepthCMP(uint4 _currentPixel[4], float2 _uv)
{

    denoisedPSOutPut output;

    float3 averageColor[4] = { float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0) };
    float averageDepth[4] = { 0, 0, 0, 0 };
    float3 averageMotionVector[4] = { float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0) };

    static const float depthCmp = 0.95;
    static const int filterSize = 25;

    for (int i = 0; i < filterSize; i++)
    {
        float2 aroundUV = _uv + offsets[i];
        for (int layer = 0; layer < 4; layer++)
        {
            averageColor[layer] += unpack_rgb(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).x);
            averageDepth[layer] = unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).y);
            averageMotionVector[layer] = float3(unpack_MotionVector_XY(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).z), pack_MotionVector_Z(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).y));
        }
    }

    for (int j = 0; j < 4; j++)
    {
        averageColor[j] /= filterSize;
        averageDepth[j] /= filterSize;
        averageMotionVector[j] /= filterSize;

        float d = max((1 - abs(unpack_depth(_currentPixel[j].y) - averageDepth[j]) / ZFAR), 0);

        if (d < depthCmp)
            _currentPixel[j] = uint4(pack_rgb(averageColor[j]), 0, 0, 0);
    }

    output.denoised_Layer_0 = _currentPixel[0];
    output.denoised_Layer_1 = _currentPixel[1];
    output.denoised_Layer_2 = _currentPixel[2];
    output.denoised_Layer_3 = _currentPixel[3];

    return output;
}

denoisedPSOutPut SD(uint4 _currentPixel[4], float2 _uv)
{

    denoisedPSOutPut output;

    float3 averageColor[4] = { float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0) };
    float averageDepth[4] = { 0, 0, 0, 0 };
    float3 averageMotionVector[4] = { float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0) };
    float meanDepth[4], squaMeanDepth[4], sd[4], Max[4], Min[4];

    static const int constant = 0.5;
    int cnt[4] = { 0, 0, 0, 0 };

    for (int k = 0; k < 4; k++)
    {

        meanDepth[k] = unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(_uv) / 8, k + 4, 3)).x);
        squaMeanDepth[k] = unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(_uv) / 8, k + 4, 3)).y);

        sd[k] = sqrt(squaMeanDepth[k] - meanDepth[k] * meanDepth[k]);
        Max[k] = meanDepth[k] + sd[k] * constant;
        Min[k] = meanDepth[k] - sd[k] * constant;
    }

    for (int i = 0; i < 25; i++)
    {
        float2 aroundUV = _uv + offsets[i];
        for (int layer = 0; layer < 4; layer++)
        {
            float depth = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).y;
            //if (Max[layer] > unpack_depth(depth) && unpack_depth(depth) > Min[layer]) {
            averageColor[layer] += unpack_rgb(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).x);
            averageDepth[layer] += depth;
            averageMotionVector[layer] += float3(unpack_MotionVector_XY(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).z), pack_MotionVector_Z(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), layer, 0)).w));
            cnt[layer]++;
            //}
        }
    }

    for (int j = 0; j < 4; j++)
    {
        averageColor[j] /= cnt[j];
        averageDepth[j] /= cnt[j];
        averageMotionVector[j] /= cnt[j];

        //if (Min[j] > unpack_depth(_currentPixel[j].y) && unpack_depth(_currentPixel[j].y) > Max[j]) {
        if (unpack_depth(_currentPixel[j].y) < Min[j] && unpack_depth(_currentPixel[j].y) > Max[j])
        {
            //_currentPixel[j] = uint4(pack_rgb(averageColor[j]), 0, 0, 0);
            _currentPixel[j] = uint4(pack_rgb(float3(1, 0, 0)), 0, 0, 0);
        }
    }
    
    output.denoised_Layer_0 = _currentPixel[0];
    output.denoised_Layer_1 = _currentPixel[1];
    output.denoised_Layer_2 = _currentPixel[2];
    output.denoised_Layer_3 = _currentPixel[3];

    return output;
}

denoisedPSOutPut ReplacePixelByCmpDepth(uint4 _currentPixel[4], float2 _uv)
{

    denoisedPSOutPut output;
    float averageDepth[4] = { 0, 0, 0, 0 };
    float3 averageColor[4] = { float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0), float3(0, 0, 0) };
    uint4 currentPixel[4] = _currentPixel;
    static const int constant = 1;
    float depthCmp = 0.97;

    for (int i = 0; i < 25; i++)
    {
        float2 aroundUV = _uv + offsets[i];
        for (int j = 1; j < 4; j++)
        {
            averageDepth[j] += unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), j, 0)).y);
            averageColor[j] += unpack_rgb(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(aroundUV), j, 0)).x);
        }
    }
    
    for (int k = 1; k < 4; k++)
    {

        averageDepth[k] /= 25;
        averageColor[k] /= 25;

        float meanDepth = unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(_uv) / 8, k + 4, 3)).x);
        float squaMeanDepth = unpack_depth(g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(_uv) / 8, k + 4, 3)).y);
        float sd = sqrt(squaMeanDepth - meanDepth * meanDepth);

        float Max = meanDepth + sd * constant;
        float Min = meanDepth - sd * constant;
        
        float d = max((1 - abs(unpack_depth(_currentPixel[k].y) - averageDepth[k]) / ZFAR), 0);

        //if (abs(unpack_depth(_currentPixel[k].y) - averageDepth[k]) > abs(unpack_depth(_currentPixel[k - 1].y) - averageDepth[k])) {
        if (abs(unpack_depth(_currentPixel[k].y) - averageDepth[k]) > abs(unpack_depth(_currentPixel[k - 1].y) - averageDepth[k]) && abs(unpack_depth(_currentPixel[k - 1].y) - averageDepth[k]) < 10)
        {
            currentPixel[k] = uint4(pack_rgb(float3(1, 0, 0)), _currentPixel[k - 1].yzw);
            //currentPixel[k] = _currentPixel[k - 1];
        }
        else if (unpack_depth(_currentPixel[k].y) < Min || unpack_depth(_currentPixel[k].y) > Max)
        {
            currentPixel[k] = uint4(pack_rgb(float3(1, 0, 0)), _currentPixel[k - 1].yzw);
            //currentPixel[k] = uint4(pack_rgb(averageColor[j]), pack_depth(averageDepth[j]), 0, 0);
        }
        else if (d < depthCmp && unpack_depth(_currentPixel[k].y) / ZFAR > 0.1f)
        {
            currentPixel[k] = uint4(pack_rgb(float3(1, 0, 0)), _currentPixel[k - 1].yzw);
            //currentPixel[k] = uint4(pack_rgb(averageColor[j]), pack_depth(averageDepth[j]), 0, 0);
        }
    }

    output.denoised_Layer_0 = currentPixel[0];
    output.denoised_Layer_1 = currentPixel[1];
    output.denoised_Layer_2 = currentPixel[2];
    output.denoised_Layer_3 = currentPixel[3];

    return output;
}

denoisedPSOutPut Denoising_ps(FSQVSOut input) : SV_TARGET
{
    denoisedPSOutPut output;
    uint4 currentPixel[4];

    for (int i = 0; i < 4; i++)
    {
        currentPixel[i] = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(input.uv), i, 0));
    }

    //output = SD(currentPixel, input.uv);
    //output = DepthCMP(currentPixel, input.uv);
    output = ReplacePixelByCmpDepth(currentPixel, input.uv);

    return output;
}

technique11 Denoising
{
    pass P0
    {
        SetRasterizerState(NoCullNoMSAA_RS);
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(CompileShader(ps_5_0, Denoising_ps()));
    }
}

//------------------------------------------------------------- motion blur pass

struct motionBlurPSOutput
{
    float4 color;
    float4 dpeth_successRate_motionVector;
};

float4 searchBoundary(float2 uv)
{
    const int constant = 2;

    float2 mean = g_motionVectorMipMap.SampleLevel(g_samLinear, uv, MIPMAP_LEVEL).xy;
    float2 squaMean = g_motionVectorMipMap.SampleLevel(g_samLinear, uv, MIPMAP_LEVEL).zw;

    float2 standardDeviation = sqrt(squaMean - mean * mean);
    float2 Max = mean + standardDeviation * constant;
    float2 Min = mean - standardDeviation * constant;

    float2 leftTopCorner = max(uv - Max, float2(0.0f, 0.0f));
    float2 rightBottomCorner = min(uv - Min, float2(1.0f, 1.0f));

    return float4(leftTopCorner, rightBottomCorner);
}

motionBlurPSOutput findLineAndGetColor(float2 uv)
{
    static const float epsilon = g_pixelSize.y * EPSILON;
    float2 randomUV = float2(0, 0);
    float sumDepth = 0;

    float2 intersecting_Motion_Vector[N_RANDOMPICK * N_LAYER];
    int nNumintersecting_Motion_Vector = 1;
    uint4 firstLayer = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(uv), 0, 0));
    intersecting_Motion_Vector[0] = unpack_MotionVector_XY(firstLayer.z);

    float4 motionBoundary = searchBoundary(uv);
    motionBlurPSOutput output;

    for (int sampleCnt = 0; sampleCnt < N_RANDOMPICK; sampleCnt++)
    {

        randomUV = float2(Random(float2(uv.x + sampleCnt * 0.0002468f, uv.y + sampleCnt * 0.0003456f)), Random(float2(uv.x + sampleCnt * 0.0001357f, uv.y + sampleCnt * 0.0004567f)));
        float2 currentCandidate = float2(motionBoundary.x + randomUV.x * abs(motionBoundary.z - motionBoundary.x), motionBoundary.y + randomUV.y * abs(motionBoundary.w - motionBoundary.y));

        for (int Layer = 0; Layer < N_LAYER; Layer++)
        {
            uint4 packed_Color_Depth_MotionVector = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(currentCandidate), Layer, 0));
            float2 candidate_MotionVector = unpack_MotionVector_XY(packed_Color_Depth_MotionVector.z);

            bool isFound = false;
            for (int j = 0; j < N_SEARCH; j++)
            {

                if (intersect(uv, candidate_MotionVector, currentCandidate, epsilon))
                {
                    isFound = true;
                    break;
                }

                currentCandidate = uv - candidate_MotionVector * sampleCnt / N_RANDOMPICK;

                uint4 packed_Color_Depth_MotionVector = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(currentCandidate), Layer, 0));
                candidate_MotionVector = unpack_MotionVector_XY(packed_Color_Depth_MotionVector.z);
            }

            if (isFound)
            {
                int k;
                for (k = 0; k < nNumintersecting_Motion_Vector; k++)
                {
                    if (distance(candidate_MotionVector, intersecting_Motion_Vector[k]) < g_pixelSize.y)
                        break;
                }

                if (k == nNumintersecting_Motion_Vector)
                {
                    intersecting_Motion_Vector[k] = candidate_MotionVector;
                    nNumintersecting_Motion_Vector++;
                }
            }
        }
    }

    if (nNumintersecting_Motion_Vector == 1 && -g_pixelSize.x < intersecting_Motion_Vector[0].x && intersecting_Motion_Vector[0].x < g_pixelSize.x && -g_pixelSize.y < intersecting_Motion_Vector[0].y && intersecting_Motion_Vector[0].y < g_pixelSize.y)
    {
        output.color = float4(unpack_rgb(firstLayer.x), 0);
        output.dpeth_successRate_motionVector = float4(unpack_depth(firstLayer.y), 1, 0, 0);
    }
    else
    {
        bool searchFirstLayer = false;
        float nNumOfSucess = 0;
        float3 colorSum = float3(0.0f, 0.0f, 0.0f);
        for (int time = 0; time < N_SAMPLINGTIME; time++)
        {
            randomUV = float2(Random(float2(uv.x + time * 0.0002468f, uv.y + time * 0.0003456f)), Random(float2(uv.x + time * 0.0001357f, uv.y + time * 0.0004567f)));
            float targetDepth = ZFAR;
            float3 targetColor = float3(-1.f, 0.f, 0.f);
            for (int j = 0; j < nNumintersecting_Motion_Vector; j++)
            {
                for (int layer = 0; layer < N_LAYER; layer++)
                {
                    float2 inverseMotionVector = uv - intersecting_Motion_Vector[j];

                    float clipped_ratio = MOTIONVECTOR_MAX;
                    float ratio = 1.f;
                     
                    if (inverseMotionVector.x < 0.f)
                    {
                        if (clipped_ratio > uv.x)
                            clipped_ratio = abs(uv.x / intersecting_Motion_Vector[j].x);
                    }
                    else if (inverseMotionVector.x > 1.f)
                    {
                        if (clipped_ratio > 1.f - uv.x)
                            clipped_ratio = (1.f - uv.x) / abs(intersecting_Motion_Vector[j].x);
                    }

                    if (inverseMotionVector.y < 0.f)
                    {
                        if (clipped_ratio > uv.y)
                            clipped_ratio = abs(uv.y / intersecting_Motion_Vector[j].y);
                    }
                    else if (inverseMotionVector.y > 1.f)
                    {
                        if (clipped_ratio > 1.f - uv.y)
                            clipped_ratio = (1.f - uv.y) / abs(intersecting_Motion_Vector[j].y);
                    }

                    if (clipped_ratio != MOTIONVECTOR_MAX)
                        ratio = clipped_ratio;

                    float2 Motion_Vector_Sample_Pos = uv - intersecting_Motion_Vector[j] * ratio * (time + randomUV) / N_SAMPLINGTIME;


                    uint4 pos_Sample_Color_Depth_MotionVector = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(Motion_Vector_Sample_Pos), layer, 0));
                    float3 pos_Sample_MotionVector = float3(unpack_MotionVector_XY(pos_Sample_Color_Depth_MotionVector.z), unpack_MotionVector_Z(pos_Sample_Color_Depth_MotionVector.w));

                    if (distance(intersecting_Motion_Vector[j], pos_Sample_MotionVector.xy) < g_pixelSize.y)
                    {
                        pos_Sample_MotionVector.z *= (float) (time + randomUV) / N_SAMPLINGTIME;
                        float pos_Sample_Depth = unpack_depth(pos_Sample_Color_Depth_MotionVector.y);

                        if (pos_Sample_MotionVector.z + pos_Sample_Depth < targetDepth)
                        {
                            if (layer == 0)
                                searchFirstLayer = true;
                            targetColor = unpack_rgb(pos_Sample_Color_Depth_MotionVector.x);
                            targetDepth = pos_Sample_MotionVector.z + pos_Sample_Depth;
                            break;
                        }
                    }
                }
            }

            if (targetColor.x >= 0.f)
            {
                colorSum += targetColor;
                sumDepth += targetDepth;
                nNumOfSucess++;
            }
        }

        float successRate = 0;
        float3 resColor = unpack_rgb(firstLayer.x);

        if (nNumOfSucess > 0)
        {
            resColor = colorSum / nNumOfSucess;
            successRate = nNumOfSucess / N_SAMPLINGTIME;
        }
        if (!searchFirstLayer)
        {
            successRate = 0;
        }
        output.color = float4(resColor, 1);
        output.dpeth_successRate_motionVector = float4(sumDepth / nNumOfSucess, successRate * successRate, unpack_MotionVector_XY(firstLayer.z));
    }

    return output;
}

motionBlurPSOutput MakeMotionBlurPS(FSQVSOut input) : SV_TARGET
{
    return findLineAndGetColor(input.uv);
}

technique11 MakeMotionBlur
{
    pass P0
    {
        SetRasterizerState(NoCullNoMSAA_RS);
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(CompileShader(ps_5_0, MakeMotionBlurPS()));
    }
}

//------------------------------------------------------------- Denoising based on pixel reliability pass

float4 denoising(float2 uv)
{
    float sum_w = 0, depthCmp = 0.99;
    float3 sumColor = float3(0.0f, 0.0f, 0.0f);
    uint4 firstLayer = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(uv), 0, 0));
    float3 curPixelColor = unpack_rgb(firstLayer.x).xyz;

    [unroll(25)]
    for (int j = 0; j < 25; j++)
    {
        float2 aroundUV = uv + offsets[j];
        float3 RGBtmp = g_MotionBlurColorMap.Sample(g_samLinear, aroundUV).xyz;
        float4 aroundDepth_SuccessRate_motionVector = g_MotionBlurDepthWeightMap.Sample(g_samLinear, aroundUV);
        float c_w = abs(1 - dot(RGBtmp, curPixelColor));
        float d_w = max((1 - abs(unpack_depth(firstLayer.y) - aroundDepth_SuccessRate_motionVector.x) / ZFAR), 0);
        if (d_w < depthCmp)
            d_w = 0.3;

        //float weight = gaussianKernel[j] * aroundDepth_SuccessRate_motionVector.y * c_w;
        float weight = gaussianKernel[j] * aroundDepth_SuccessRate_motionVector.y * d_w * c_w;
        sumColor += RGBtmp * weight;
        sum_w += weight;
        //if (needDenoging == false && aroundDepth_SuccessRate_motionVector.z != 0.f && aroundDepth_SuccessRate_motionVector.w != 0.f) needDenoging = true;
    }
    float3 resColor = sumColor / sum_w;

    return float4(resColor, 1);
}

float4 DenoisingPS(FSQVSOut input) : SV_TARGET
{
    float4 color;
    //uint4 pos_Sample_Color_Depth_MotionVector = g_sorted_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(input.uv), 2, 0));
    //uint4 pos_Sample_Color_Depth_MotionVector = g_denoised_Color_Depth_MotionVectorMap.Load(int4(GetScreenUV(input.uv), 1, 0));
    //float4 pos_Sample_Color_Depth_MotionVector = g_sorted_Color_Depth_MotionVectorMap.Load(int4(input.position.xy , 0, 0));
    //color = float4(unpack_rgb(pos_Sample_Color_Depth_MotionVector.x), 0.f);
    //color = float4(unpack_depth(pos_Sample_Color_Depth_MotionVector.x),0,0, 0.f);

    //color = float4(abs(unpack_MotionVector_XY(pos_Sample_Color_Depth_MotionVector.z)), abs(unpack_MotionVector_Z(pos_Sample_Color_Depth_MotionVector.w)), 0.f);
    //color = float4(color.x * color.x, color.y* color.y, color.z * color.z, 0.f);
    //color = float4(unpack_depth(pos_Sample_Color_Depth_MotionVector.y / ZFAR), unpack_depth(pos_Sample_Color_Depth_MotionVector.y / ZFAR), unpack_depth(pos_Sample_Color_Depth_MotionVector.y / ZFAR), 0);
    //color = float4(unpack_depth(pos_Sample_Color_Depth_MotionVector.x) / ZFAR, unpack_depth(pos_Sample_Color_Depth_MotionVector.x) / ZFAR, unpack_depth(pos_Sample_Color_Depth_MotionVector.x) / ZFAR,0);

    //color = float4(g_motionVectorMipMap.SampleLevel(g_samPoint, input.uv, 0).xy, 0, 0);
    //color = g_firstLayerColorMap.SampleLevel(g_samLinear, input.uv, 0);
    //color = g_motionVectorMap.SampleLevel(g_samLinear, input.uv, 0);
    //color = g_depthMap.SampleLevel(g_samLinear, input.uv, 0);

    //color = g_MotionBlurColorMap.Sample(g_samLinear, input.uv);
    //color = float4(g_MotionBlurColorMap.Sample(g_samLinear, input.uv).w / 100, g_MotionBlurColorMap.Sample(g_samLinear, input.uv).w / 100, g_MotionBlurColorMap.Sample(g_samLinear, input.uv).w / 100, 1);

    color = denoising(input.uv);

    //color = float4(denoising(input.uv).y, denoising(input.uv).y, denoising(input.uv).y ,0);

    //float depth = g_MotionBlurDepthWeightMap.Sample(g_samLinear, input.uv).y;
    //color = float4(depth / ZFAR, depth/ ZFAR, depth / ZFAR, 0);

    //float successRate = g_MotionBlurColorMap.Sample(g_samLinear, input.uv).w;
    //color = float4(successRate, successRate, successRate, 0);

    //color = float4(color.w, color.w, color.w, 1);
    //color = float4(input.uv, 0, 1);

    //if (input.uv.x > 0.04f && input.uv.x < 0.05f)  color = float4(1, 0, 0, 0);

    return color;
}

technique11 DenoisingWithReliability
{
    pass P0
    {
        SetRasterizerState(NoCullNoMSAA_RS);
        SetVertexShader(CompileShader(vs_5_0, FullScreenQuadVS()));
        SetPixelShader(CompileShader(ps_5_0, DenoisingPS()));
    }
}


```

















# 출력
---



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/f413b79c-1d3d-409b-ab47-4cbc05604146" alt width=800>
<em></em>
</center>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7a94cc11-8098-4a39-af97-a7324ec6d06b" alt width=700>
<em></em>
</center>


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ea21f298-a33c-4abd-a10b-9d445e15c599" alt width=700>
<em></em>
</center>
