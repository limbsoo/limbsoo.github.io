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
# 목표
---

아래 과정을 통해 모션 블러 이미지를 생성한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d0530b70-a4a0-40f1-9c4f-435c43f1af93" alt width=700>
<em></em>
</center>





# 구현
---

기존 Efficient motion blur through motion vector sharing의 모션 블러 생성 방법에서 여러 깊이 별 레이어를 추가하여 모션블러 이미지를 생성한다.


```c++


void DepthPeeling(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext)
{

    //-----------------------------------------------------------------------------
    // 	   1PASS:	Clear Texture
    //-----------------------------------------------------------------------------

    g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("ClearTexture");
    g_pDepthPeelingTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);

    pd3dImmediateContext->OMSetRenderTargets(N_LAYER, g_pColorMap_RenderingClass->ppRTVs, NULL);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    pd3dImmediateContext->OMSetRenderTargets(N_LAYER, g_pMotionVectorMap_RenderingClass->ppRTVs, NULL);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    pd3dImmediateContext->OMSetRenderTargets(1, g_pMotionVectorSquareMap_RenderingClass->ppRTVs, NULL);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);


    pd3dImmediateContext->OMSetRenderTargets(N_LAYER, g_pNormalMap_RenderingClass->ppRTVs, NULL);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    //-----------------------------------------------------------------------------
    // 	   2PASS:	DepthPeeling Render
    //-----------------------------------------------------------------------------

    //g_pSkybox->BeforeFrameRender(g_Camera.GetViewMatrix(), g_Camera.GetProjMatrix(), 10.0f, 0.0f, 0.0f, 0.f);
    //g_pSkybox->Render(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    ID3D11ShaderResourceView* DepthBuffer_SRVs[] =
    {
        g_pDepthBuffer_RenderingClass[0]->GetShaderResourceView(),
        g_pDepthBuffer_RenderingClass[1]->GetShaderResourceView()
    };

    ID3D11DepthStencilView* DepthBuffer_DSVs[] =
    {
        g_pDepthBuffer_RenderingClass[0]->GetDepthStencilView(),
        g_pDepthBuffer_RenderingClass[1]->GetDepthStencilView()
    };

    pd3dImmediateContext->ClearDepthStencilView(DepthBuffer_DSVs[1], D3D11_CLEAR_DEPTH, 0.0, 0);

    for (int layer = 0; layer < N_LAYER; layer++)
    {
        int currId = layer % 2;
        int prevId = 1 - currId;

        if (layer == 0)
        {
            ID3D11RenderTargetView* Layers_RTVs[] =
            {
                g_pColorMap_RenderingClass->ppRTVs[layer],
                g_pMotionVectorMap_RenderingClass->ppRTVs[layer],
                g_pMotionVectorSquareMap_RenderingClass->ppRTVs[layer],
                g_pNormalMap_RenderingClass ->ppRTVs[layer]
            };

            pd3dImmediateContext->ClearDepthStencilView(DepthBuffer_DSVs[currId], D3D11_CLEAR_DEPTH, 1.0, 0);
            //pd3dImmediateContext->OMSetRenderTargets(3, Layers_RTVs, NULL);

            pd3dImmediateContext->OMSetRenderTargets(4, Layers_RTVs, NULL);

            //g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("DepthPeelingRendering");
            g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("DepthPeelingRenderingWithSkybox");
            g_pDepthPeelingTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
            g_effectSRV_depthBuffer->SetResource(DepthBuffer_SRVs[prevId]);
            //pd3dImmediateContext->OMSetRenderTargets(3, Layers_RTVs, DepthBuffer_DSVs[currId]);

            pd3dImmediateContext->OMSetRenderTargets(4, Layers_RTVs, DepthBuffer_DSVs[currId]);

            boxDrawing = true;
            renderPerObjects(pd3dImmediateContext, g_pDepthPeelingTech, g_pDepthPeelingEffect);

        }

        else
        {
            ID3D11RenderTargetView* Layers_RTVs[] =
            {
                g_pColorMap_RenderingClass->ppRTVs[layer],
                g_pMotionVectorMap_RenderingClass->ppRTVs[layer],
                g_pNormalMap_RenderingClass->ppRTVs[layer]
            };

            pd3dImmediateContext->ClearDepthStencilView(DepthBuffer_DSVs[currId], D3D11_CLEAR_DEPTH, 1.0, 0);
            //pd3dImmediateContext->OMSetRenderTargets(2, Layers_RTVs, NULL);
            pd3dImmediateContext->OMSetRenderTargets(3, Layers_RTVs, NULL);

            g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("DepthPeelingRendering");
            g_pDepthPeelingTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
            g_effectSRV_depthBuffer->SetResource(DepthBuffer_SRVs[prevId]);

            //pd3dImmediateContext->OMSetRenderTargets(2, Layers_RTVs, DepthBuffer_DSVs[currId]);
            pd3dImmediateContext->OMSetRenderTargets(3, Layers_RTVs, DepthBuffer_DSVs[currId]);
            renderPerObjects(pd3dImmediateContext, g_pDepthPeelingTech, g_pDepthPeelingEffect);
        }
    }

    pd3dImmediateContext->GenerateMips(g_pMotionVectorMap_RenderingClass->ppSRVs[0]);
    pd3dImmediateContext->GenerateMips(g_pMotionVectorSquareMap_RenderingClass->ppSRVs[0]);

    //---------------------------------------------------------------------------------
    //PASS 3 Make MotionBlur on FullSceenQuad
    //---------------------------------------------------------------------------------

    ID3D11RenderTargetView* Layers_RTVs = g_pPrintMotionBlurClass->GetRenderTargetView();

    pd3dImmediateContext->OMSetRenderTargets(1, &Layers_RTVs, NULL);
    g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("MotionBlur");
    g_pDepthPeelingTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    //---------------------------------------------------------------------------------
    //PASS 4 Denoising
    //---------------------------------------------------------------------------------

    ID3D11RenderTargetView* pRTV = DXUTGetD3D11RenderTargetView();
    pd3dImmediateContext->OMSetRenderTargets(1, &pRTV, NULL);
    g_pDepthPeelingTech = g_pDepthPeelingEffect->GetTechniqueByName("Denoising");
    g_pDepthPeelingTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);
    g_pFSQ->RenderFSQ(pd3dImmediateContext, g_pLayout, g_pDepthPeelingTech, techDesc, g_pDepthPeelingEffect);

    static int frame = 0;
    CString fileName = "motion blur";
    if (frame >= 1 && frame <= 100) renderTagetToImageFile(pd3dDevice, pd3dImmediateContext, pRTV, frame, fileName);
    frame++;
}


void DepthPeeling_SetResouce()
{
    g_effectSRV_colorMap              = g_pDepthPeelingEffect->GetVariableByName("g_colorMap")->AsShaderResource();
    g_effectSRV_motionVectorMap       = g_pDepthPeelingEffect->GetVariableByName("g_motionVectorMap")->AsShaderResource();
    g_effectSRV_motionVectorSquareMap = g_pDepthPeelingEffect->GetVariableByName("g_motionVectorSquareMap")->AsShaderResource();
    g_effectSRV_depthBuffer           = g_pDepthPeelingEffect->GetVariableByName("g_depthBuffer")->AsShaderResource();
    g_effectSRV_motionBlur            = g_pDepthPeelingEffect->GetVariableByName("g_motionBlur")->AsShaderResource();
    g_effectSRV_denoising             = g_pDepthPeelingEffect->GetVariableByName("g_denosing")->AsShaderResource();

    g_effectSRV_colorMap->SetResource(g_pColorMap_RenderingClass->ppSRVs[0]);
    g_effectSRV_motionVectorMap->SetResource(g_pMotionVectorMap_RenderingClass->ppSRVs[0]);
    g_effectSRV_motionVectorSquareMap->SetResource(g_pMotionVectorSquareMap_RenderingClass->ppSRVs[0]);
    g_effectSRV_motionBlur->SetResource(g_pPrintMotionBlurClass->GetShaderResourceView());
    g_effectSRV_denoising->SetResource(g_pDenoising_RenderingClass->GetShaderResourceView());


    g_effectSRV_normalMap = g_pDepthPeelingEffect->GetVariableByName("g_normalMap")->AsShaderResource();
    g_effectSRV_normalMap->SetResource(g_pNormalMap_RenderingClass->ppSRVs[0]);
}

void renderPerObjects(ID3D11DeviceContext* pd3dImmediateContext, ID3DX11EffectTechnique* pTech, ID3DX11Effect* pEffect)
{

    if (boxDrawing)
    {
        g_pSkybox->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);
        boxDrawing = false;
    }

    g_pModel_1->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);
    g_pModel_2->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);
    g_pModel_3->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);
    g_pModel_4->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);
    g_pBackgroundModel->Render(pd3dImmediateContext, g_pLayout, pTech, techDesc, pEffect);

}

```

<br>

기존 샘플 과정은 아래를 따른다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/756624ed-7559-4c5b-ad85-c580d955d539" alt width="100%">
<em></em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0bfc1145-1b84-4687-a778-0e645785c3e6" alt width="100%">
<em></em>
</center>

<br>


그러나 이러한 샘플과정은 아래 그림과 같이 물체의 색상이 소실이 발생한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/8a1325dc-22fa-4a32-8fa9-72da38d97182" alt width=700>
<em></em>
</center>

<br>

이는 아래 그림과 같이 색상 샘플에 실패하면서 발생하는 현상이다. 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/27b88244-fef8-40c2-8c6a-715a083a1251" alt width="80%">
<em></em>
</center>


<br>

따라서, 이러한 색상 샘플에 실패한 픽셀 위치에서 다시 샘플을 다음 레이어에서 시도한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ac53a233-08b0-485f-b327-d9b06a6cd140" alt width=700>
<em></em>
</center>


<br>

이를 위해 여러 레이어를 Depth Peeling Rendering으로 나눈다.Depth Peeling은 기존에 렌더링 된 이미지의 픽셀을 peeling하고 다시 렌더링하는 방법으로, 렌더링 이미지를 z-buffer를 이용한 여러 개의 layer로 나눠 뒷면의 모션 벡터 정보를 보존하는 방법이다.

<br>

Current pixel와 PreviousLayerDepthMap$[current pixel]$의 Depth를 비교하는 Rendering 방법으로, Current pixel의 depth값이 PreviousLayerDepthMap$[current pixel]$값보다 작거나 같으면 Previous layer에서 이미 Rendering한 Polygon이므로 Discard, Current pixel의 depth값이 Previous layer depth값보다 크다면 Rendering한다.

```
if (depth <= PreviousLayerDepthMap[current pixel])
   discard
else
   rendering
```

<br>



```



struct VS_INPUT
{
   float4 Position    : POSITION; 
   float3 Normal      : NORMAL;
   float2 TextureUV   : TEXCOORD0;
};

struct VS_OUTPUT
{
   float4 Position     : SV_POSITION;
   float4 prePosition  : POSITION0;
   float4 curPosition  : POSITION1;
   float3 Normal       : NORMAL;
   float2 TexCoord     : TEXCOORD0;  

   float4 WorldPos : POSITION2;
};

struct RenderOutput_PS
{
   float4 color;
   float4 motionVector3D;
   float4 motionVector3DSquare;
   float3 outNormal;
};

VS_OUTPUT DepthPeelingRendering_VS(VS_INPUT InputVS)
{
   VS_OUTPUT OutputVS = (VS_OUTPUT)0;

   OutputVS.Position = mul(InputVS.Position, g_previousWorldViewProjection);
   OutputVS.prePosition = OutputVS.Position;
   OutputVS.curPosition = mul(InputVS.Position, g_worldViewProjection);
   OutputVS.TexCoord = InputVS.TextureUV;

   //OutputVS.Normal = InputVS.Normal;
   //float3 n = normalize(mul(float4(InputVS.Normal, 0.f), g_world)).xyz;
   //float3 n = mul(float4(InputVS.Normal, 0.f), g_world).xyz;
   //OutputVS.Normal = normalize(mul(float4(InputVS.Normal, 0.f), g_world));
   //n = normalize(n);
   //OutputVS.Normal = float3(n.x, n.y, n.z);
   //OutputVS.Normal = n.xyz;

   OutputVS.WorldPos = mul(InputVS.Position, g_world);
   OutputVS.Normal = mul(float4(InputVS.Normal, 0.f), g_world).xyz;
   OutputVS.Normal = normalize(OutputVS.Normal);
   //OutputVS.Normal = mul(InputVS.Normal,g_world);
   //OutputVS.Normal = mul(InputVS.Normal, (float3x3)g_world);



   return OutputVS;
}

RenderOutput_PS DepthPeelingRendering_PS(VS_OUTPUT InputPS) : SV_TARGET
{
   float3 currentPos = homogeneous2uv(InputPS.curPosition);
   float3 previousPos = homogeneous2uv(InputPS.prePosition);
   float3 motionVector = currentPos - previousPos;
   float3 motionVectorSquare = float3(motionVector.x * motionVector.x, motionVector.y * motionVector.y, motionVector.z * motionVector.z);
   float4 color = g_modelTexture.Sample(g_samPoint, InputPS.TexCoord , 0);
   float depth = Linearize(InputPS.Position.z, 0.1f, 1000.0f);

   RenderOutput_PS outputPS;
   outputPS.color = float4(color.xyz, depth);
   outputPS.motionVector3D = float4(motionVector, 1.0f);
   outputPS.motionVector3DSquare = float4(motionVectorSquare, 1.0f);

   outputPS.outNormal = InputPS.Normal;


   float3 ambientLightColor = float3(0.1f, 0.1f, 0.1f);
   float ambientLightStrength = 0.01f;
   float3 dynamicLightPosition = float3(15.f, -6.f, 16.f);
   float dynamicLightStrength = 0.9f;
   float3 dynamicLightColor = float3(1.f, 1.f, 1.f);

   float3 ambientLight = ambientLightColor * ambientLightStrength;
   float3 appliedLight = ambientLight;
   float3 vectorToLight = normalize(dynamicLightPosition - InputPS.WorldPos.xyz).xyz;
   float3 diffuseLightIntensity = max(dot(vectorToLight, outputPS.outNormal), 0);
   float3 diffuseLight = diffuseLightIntensity * dynamicLightStrength * dynamicLightColor;
   appliedLight += diffuseLight;
   float3 finalColor = color.xyz * appliedLight;
   outputPS.color = float4(finalColor, depth);


   float zFront = g_depthBuffer.Load(int3(InputPS.Position.xy, 0)).r;
   if (InputPS.Position.z <= zFront) discard;

   return outputPS;
}

technique11 DepthPeelingRendering
{
   pass P0
   {
      SetVertexShader(CompileShader(vs_5_0, DepthPeelingRendering_VS()));
      SetGeometryShader(NULL);
      SetPixelShader(CompileShader(ps_5_0, DepthPeelingRendering_PS()));

      //SetRasterizerState(rasterizerState_New);

      SetRasterizerState(BackCullNoMSAA_RS);
      SetDepthStencilState(DepthNoStencil_DS, 0x00000000);
      SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
   }
}


technique11 DepthPeelingRenderingWithSkybox
{
    pass P0
    {
        SetVertexShader(CompileShader(vs_5_0, DepthPeelingRendering_VS()));
        SetGeometryShader(NULL);
        SetPixelShader(CompileShader(ps_5_0, DepthPeelingRendering_PS()));

        //SetRasterizerState(rasterizerState_New);

        SetRasterizerState(NoCullNoMSAA_RS);
        SetDepthStencilState(DepthNoStencil_DS, 0x00000000);
        SetBlendState(NoMRT_BS, float4(1.0f, 1.0f, 1.0f, 1.0f), 0xffffffff);
    }
}


//---------------------------------------------------------------------------------
//PASS 3 Make MotionBlur on FullSceenQuad
//---------------------------------------------------------------------------------

float Get_Random(float2 uv)//
{
   return frac(sin(dot(uv, float2(12.9898f, 78.233f) * 2.0f)) * 43758.5453f);
}

float2 rand_2_10(in float2 uv) {
   float noiseX = (frac(sin(dot(uv, float2(12.9898, 78.233) * 2.0)) * 43758.5453));
   float noiseY = sqrt(1 - noiseX * noiseX);
   return float2(noiseX, noiseY);
}


float4 MotionBoundary(float2 uv, float mipmapLevelForGaussian)
{
   const int constant =3;

   float2 mean = g_motionVectorMap.SampleLevel(g_samLinear, float3(uv, 0), mipmapLevelForGaussian).xy;
   float2 squaMean = g_motionVectorSquareMap.SampleLevel(g_samLinear, float3(uv, 0), mipmapLevelForGaussian).xy;

   //float2 mean = g_motionVectorMap.SampleLevel(g_samPoint, float3(uv, 0), mipmapLevelForGaussian).xy;
   //float2 squaMean = g_motionVectorSquareMap.SampleLevel(g_samPoint, float3(uv, 0), mipmapLevelForGaussian).xy;

   float2 standardDeviation = sqrt(squaMean - mean * mean);//제곱의 평균 -평균의 제곱 : 표준편차 제곱 =>분산
   float2 Max = mean + standardDeviation * constant;
   float2 Min = mean - standardDeviation * constant;

   float2 leftTopCorner = max(uv - Max, float2(0.0f, 0.0f));
   float2 rightBottomCorner = min(uv - Min, float2(1.f, 1.0f));

   return float4 (leftTopCorner, rightBottomCorner);
}


bool IsIntersect(float2 uv, float2 lineStart, float2 lineMotionVector)
{

   bool Intersected = false;

   static const float eps = 0.5f;

   static const float threshold_X = 0.5f / SCREENWIDTH;
   static const float threshold_Y = 0.5f / SCREENHEIGHT;

   if (distance(uv.x, lineStart.x) >= threshold_X && distance(uv.x, lineStart.x) <= -threshold_X &&
      distance(uv.y, lineStart.y) >= threshold_Y && distance(uv.y, lineStart.y) <= -threshold_Y)
   {
      Intersected = true;
   }

   else if (lineMotionVector.x <= eps && lineMotionVector.x >= -eps && lineMotionVector.y <= eps && lineMotionVector.y >= -eps)
   {
      Intersected = true;
   }

   else
   {

      static const float2 pixelSize = float2(1 / SCREENWIDTH, 1 / SCREENHEIGHT);
      static const float epsilon = pixelSize.y * 7.f;

      float2 RS_Start = lineStart;
      float2 RS_Vector = lineMotionVector;
      float2 RS_Point = uv;

      float segmentLengthSqr = (RS_Vector.x * RS_Vector.x) + (RS_Vector.y * RS_Vector.y);
      float r = ((RS_Point.x - RS_Start.x) * RS_Vector.x + (RS_Point.y - RS_Start.y) * RS_Vector.y) / segmentLengthSqr;

      if (r > 0 && r < 1)
      {
         float sl = ((RS_Start.y - RS_Point.y) * RS_Vector.x - (RS_Start.x - RS_Point.x) * (RS_Vector.y)) / sqrt(segmentLengthSqr);

         if (-epsilon <= sl && sl <= epsilon)
         {
            Intersected = true;
         }

      }
   }


   return Intersected;


}

bool intersect(float2 target, float2 vec, float2 start, float epsilon) {

    bool intersecting = false;

    float2 startToTarget = target - start;
    float segmentLengthSqr = vec.x * vec.x + vec.y * vec.y;
    float r = (startToTarget.x * vec.x + startToTarget.y * vec.y) / segmentLengthSqr;

    if (r > 0 && r < 1) {
        float2 targetToStart = start - target;
        float sl = (targetToStart.y * vec.x - targetToStart.x * vec.y) / sqrt(segmentLengthSqr);

        if (-epsilon <= sl && sl <= epsilon) intersecting = true;
    }

    return intersecting;
}

struct FullScreenQuadVertexOut 
{
   float4 position : SV_POSITION;
   float2 uv : TEXCOORD0;
};

FullScreenQuadVertexOut FullScreenQuadVS(uint vertexID : SV_VertexID)
{
   FullScreenQuadVertexOut OutputVS;

   OutputVS.uv = float2((vertexID << 1) & 2, vertexID & 2);
   OutputVS.position = float4(OutputVS.uv * float2(2.0f, -2.0f) + float2(-1.0f, 1.0f), 0.0f, 1.0f);

   return OutputVS;
}



float4 FullScreenQuadPS(FullScreenQuadVertexOut In) : SV_TARGET
{
   //static const float2 g_pixelSize = float2(1.f / SCREENWIDTH, 1.f / SCREENHEIGHT);
   static const float2 pixelSize = float2(1 / SCREENWIDTH, 1 / SCREENHEIGHT);
   static const float epsilon = pixelSize.y * EPSILON;

   float2 intersectedMotionVector[N_LAYER * N_RANDOMPICK];
   int intersectingCount = 1;
   intersectedMotionVector[0] = g_motionVectorMap.Load(int4(In.uv * float2(SCREENWIDTH, SCREENHEIGHT), 0, 0)).xy;

   float2 randomUV;
   float2 StartingPoint = float2(0.0f, 0.0f);
   float2 StartingPointMotionVector;

   float4 searchBoundary = MotionBoundary(In.uv, MIPMAP_LEVEL);

   for (int i = 0; i < N_RANDOMPICK; i++)
   {

      for (int perLayer = 0; perLayer < N_LAYER; perLayer++)
      {
         bool isFind = false;

         randomUV = float2(Get_Random(float2(In.uv.x + i * 0.0002468f, In.uv.y + i * 0.0003456f)),
                    Get_Random(float2(In.uv.x + i * 0.0001357f, In.uv.y + i * 0.0004567f)));

         StartingPoint = float2(searchBoundary.x + randomUV.x * abs(searchBoundary.z - searchBoundary.x),
                         searchBoundary.y + randomUV.y * abs(searchBoundary.w - searchBoundary.y));

         StartingPointMotionVector = g_motionVectorMap.Load(int4(StartingPoint * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).xy;

         for (float j = 0; j < N_SEARCH; j++)
         {
            //float2 randomNumber = rand_2_10(In.uv);

            //if (intersect(In.uv, StartingPointMotionVector, StartingPoint, epsilon))
            if (IsIntersect(In.uv, StartingPoint, StartingPointMotionVector))
            {
               isFind = true;
               break;
            }

            else
            {
               //StartingPoint = In.uv - (StartingPointMotionVector * (j + randomNumber / (N_RANDOMPICK)));
               StartingPoint = In.uv - StartingPointMotionVector * j / N_RANDOMPICK;
               StartingPointMotionVector = g_motionVectorMap.Load(int4(StartingPoint * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).xy;
            }
         }

         if (isFind)
         {
            for (int k = 0; k < intersectingCount; k++)
            {

               //if (distance(StartingPointMotionVector, intersectedMotionVector[k]) < pixelSize.y) break;
               if (abs(StartingPointMotionVector.x - intersectedMotionVector[k].x) < pixelSize.x &&
                  abs(StartingPointMotionVector.y - intersectedMotionVector[k].y) < pixelSize.y)
               {
                  break;
               }
            }

            if (k == intersectingCount)
            {
               intersectedMotionVector[k] = StartingPointMotionVector;
               intersectingCount++;
            }

         }

      }
   }

   float currentPixelDepth = 0.0f;
   float BackwardSampled_Depth = 0.0f;
   float2 BackwardSampled_PixelPosition;
   float3 BackwardSampled_MotionVector;
   int SucessCount = 0;
   float3 sumColor = float3(0.0f, 0.0f, 0.0f);
   float3 sumDepth = float3(0.0f, 0.0f, 0.0f);

   float4 finalColor = float4(0, 0, 0, 0);

   //if (intersectingCount == 1 && -pixelSize.x < intersectedMotionVector[0].x && intersectedMotionVector[0].x < pixelSize.x && -pixelSize.y < intersectedMotionVector[0].y && intersectedMotionVector[0].y < pixelSize.y) {
   //    float4 NonMotionBlurColor = g_colorMap.Load(int4(In.uv * float2(SCREENWIDTH, SCREENHEIGHT), 0, 0));
   //    //output.dpeth_successRate_motionVector = float4(unpack_depth(firstLayer.y), 1, 0, 0);
   //    finalColor = float4(NonMotionBlurColor.xyz, 0);
   //}

   //else
   {
       for (float samplingTime_idx = 0; samplingTime_idx < N_SAMPLINGTIME; samplingTime_idx++)
       {
           float3 targetColor = float3(-1.0f, 0.0f, 0.0f);
           float targetDepth = 1000.f;

           randomUV = float2(Get_Random(float2(In.uv.x + samplingTime_idx * 0.0002468f, In.uv.y + samplingTime_idx * 0.0003456f)),
                             Get_Random(float2(In.uv.x + samplingTime_idx * 0.0001357f, In.uv.y + samplingTime_idx * 0.0004567f)));

           //int perLayer = 0;
           for (int perLayer = 0; perLayer < N_LAYER; perLayer++)
           {
               //모션벡터갯수
               for (int k = 0; k < intersectingCount; k++)
               {
                   BackwardSampled_PixelPosition = In.uv - intersectedMotionVector[k] * (samplingTime_idx + randomUV) / N_SAMPLINGTIME;
                   //BackwardSampled_PixelPosition = In.uv - (intersectedMotionVector[k] * (8.f / N_SAMPLINGTIME));
                   BackwardSampled_MotionVector = g_motionVectorMap.Load(int4(BackwardSampled_PixelPosition * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).xyz;
                   BackwardSampled_Depth = g_colorMap.Load(int4(BackwardSampled_PixelPosition * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).w;

                   if (distance(intersectedMotionVector[k], BackwardSampled_MotionVector.xy) < 0.005f)
                   {
                       BackwardSampled_MotionVector.z *= (float)(samplingTime_idx + randomUV) / N_SAMPLINGTIME;
                       //BackwardSampled_MotionVector.z *= float(12.f / SAMPLINGTIME);

                       if (BackwardSampled_Depth + BackwardSampled_MotionVector.z < targetDepth)
                       {
                           targetColor = g_colorMap.Load(int4(BackwardSampled_PixelPosition * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).xyz;
                           //targetColor = g_motionVectorMap.Load(int4(BackwardSampled_PixelPosition * float2(SCREENWIDTH, SCREENHEIGHT), perLayer, 0)).xyz;

                           targetDepth = BackwardSampled_Depth + BackwardSampled_MotionVector.z;

                       }

                   }

               }
           }


           if (targetColor.r >= 0.0f)
           {
               sumColor += targetColor;
               SucessCount++;
           }

       }

       float3 bluredColor = float3(0, 0, 0);

       if (SucessCount > 0)
       {
           bluredColor = sumColor / SucessCount;

       }

       float SuccessRate = (float)SucessCount / N_SAMPLINGTIME;
       finalColor = float4(bluredColor, SuccessRate);
   }



   return finalColor;
}


```

<br>


```
//---------------------------------------------------------------------------------
//PASS 4 Denoising
//---------------------------------------------------------------------------------

static const float Gaussiankernel[25] =
{
   0.00390625f,   0.015625f,   0.0234375f,   0.015625f,  0.00390625f,
   0.015625f,   0.0625f,   0.09375f,   0.0625f,   0.015625f,
   0.0234375f,   0.09375f,   0.140625f,   0.09375f,   0.0234375f,
   0.015625f,   0.0625f,   0.09375f,   0.0625f,   0.015625f,
   0.00390625f,   0.015625f,   0.0234375f,   0.015625f,   0.00390625f,
};

float2 g_pixelSize = float2(1.f / SCREENWIDTH, 1.f / SCREENHEIGHT);

static const float2 offsets[25] =
{
    float2(-2.0f, -2.0f) * g_pixelSize, float2(-1.0f, -2.0f) * g_pixelSize, float2(0.0f, -2.0f) * g_pixelSize, float2(1.0f, -2.0f) * g_pixelSize, float2(2.0f, -2.0f) * g_pixelSize,
    float2(-2.0f, -1.0f) * g_pixelSize, float2(-1.0f, -1.0f) * g_pixelSize, float2(0.0f, -1.0f) * g_pixelSize, float2(1.0f, -1.0f) * g_pixelSize, float2(2.0f, -1.0f) * g_pixelSize,
    float2(-2.0f, 0.0f) * g_pixelSize, float2(-1.0f, 0.0f) * g_pixelSize,  float2(0.0f, 0.0f) * g_pixelSize,  float2(1.0f, 0.0f) * g_pixelSize,  float2(2.0f, 0.0f) * g_pixelSize,
    float2(-2.0f, 1.0f) * g_pixelSize, float2(-1.0f, 1.0f) * g_pixelSize,  float2(0.0f, 1.0f) * g_pixelSize,  float2(1.0f, 1.0f) * g_pixelSize,  float2(2.0f, 1.0f) * g_pixelSize,
    float2(-2.0f, 2.0f) * g_pixelSize, float2(-1.0f, 2.0f) * g_pixelSize,  float2(0.0f, 2.0f) * g_pixelSize,  float2(1.0f, 2.0f) * g_pixelSize,  float2(2.0f, 2.0f) * g_pixelSize,
};

float getLuminance(float3 RGB)
{
   return log10((RGB.r * 0.3f + RGB.g * 0.59f + RGB.b * 0.11f) + 1.0f) / log10(2.0f);
}

float4 doEdgeAvoidATrousWavelet_improved(float2 uv)
{
   bool needDenoging = false;

   float cum_w = 0.0f;
   float2 UV, step = float2(1.0f / SCREENWIDTH, 1.0f / SCREENHEIGHT);
   float3 RGBtmp, sum = float3(0.0f, 0.0f, 0.0f);

   //if (g_motionBlur.Sample(g_samPoint, uv).w == 0)
   //{
   //    sum = g_colorMap.Load(int4(uv * float2(SCREENWIDTH, SCREENHEIGHT), 0, 0)).xyz;
   //    cum_w = 1;
   //}

   //else
   {
       [unroll(25)] for (int j = 0; j < 25; j++)
       {
           UV = uv + offsets[j];

           RGBtmp = g_motionBlur.Sample(g_samPoint, UV).xyz;
           float searchSuccessRatetmp = g_motionBlur.Sample(g_samPoint, UV).w;

           float l_w = Gaussiankernel[j] * searchSuccessRatetmp;

           float weight = l_w;
           sum += RGBtmp * weight;
           cum_w += weight;
       }
   }

   sum /= cum_w;

   sum = 0.1f;

   return float4(sum, 1.0f);
}

struct FullScreenQuadDenoisingVertexOut
{
   float4 position : SV_POSITION;
   float2 uv : TEXCOORD0;
};

FullScreenQuadDenoisingVertexOut FullScreenQuadDenoisingVS(uint vertexID : SV_VertexID)
{
   FullScreenQuadDenoisingVertexOut OutputVS;

   OutputVS.uv = float2((vertexID << 1) & 2, vertexID & 2);
   OutputVS.position = float4(OutputVS.uv * float2(2.0f, -2.0f) + float2(-1.0f, 1.0f), 0.0f, 1.0f);

   return OutputVS;
}

float4 FullScreenQuadDenoisingPS(FullScreenQuadDenoisingVertexOut In) : SV_TARGET
{

   float4 color = doEdgeAvoidATrousWavelet_improved(In.uv);// 디노이징 한거
   return color;
}

technique11 Denoising
{
   pass P0
   {
      //SetVertexShader(CompileShader(vs_5_0, FullScreenQuadDenoisingVS()));
      SetVertexShader(CompileShader(vs_5_0, FSQVS()));

      SetPixelShader(CompileShader(ps_5_0, FullScreenQuadDenoisingPS()));
      SetRasterizerState(rasterizerState_New);
   }

}



```






# 출력
---


이를 통해 depthpeeling을 통한 모션 블러 이미지를 생성하였다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/6100828a-5a33-4a03-8d17-9b6a5c7bf238" alt width="100%">
<em>모션블러 효과 비교</em>
</center>

<br>

그리고 이를 일반적인 accumulation 빙법의 모션블러 이미지와 비교했을 때 같은 샘플 횟수일 때 비슷한 품질의 이미지를 생성한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/403056ff-3db0-4b8e-8729-70c8fa7a8914" alt width=700>
<em></em>
</center>

