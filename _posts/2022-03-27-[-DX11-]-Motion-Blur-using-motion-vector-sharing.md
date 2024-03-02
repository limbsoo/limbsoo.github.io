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

모션 블러를 accumulation으로 구현하는 방법은 실제와 거의 비슷한 모션 블러 효과를 주지만, 많은 렌더링 횟수를 필요로 한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/544d21f9-6471-47f8-9c2e-15a5e6e6a990" alt width=300>
<em>Motion blur</em>
</center>

<br>

이를 해결하기 위해 모션 벡터를 활용, 렌더링 횟수를 줄인다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/c85c034c-c06e-4e4d-a6f2-3999f4bd9e2a" alt width=700>
<em>1st pass – Make Texture</em>
</center>

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0fda0bf5-4704-4235-b19d-6b7fdaa33781" alt width=700>
<em>2nd  Pass – Draw Motion Blur in Full Screen Quad</em>
</center>

<br>
<br>

# 구현
---


이동하는 물체의 모션 블러를 생성 시, 각 프레임에서 물체가 차지하고 있는 픽셀 위치가 변화한다. 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ce0e51d9-74f6-4efd-81c6-c1c42927d0cb" alt width=600>
<em>Motion blur</em>
</center>

<br>
이를 통해 물체의 움직임(현재 위치 – 이전 위치), 모션 벡터를 계산하고 이를 활용하여 물체의 움직임을 예측한다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0685941d-3ecc-4347-b7a2-bd0f5d85d5ea" alt width=600>
<em>Motion blur</em>
</center>

<br>

Mipmap sampling시 현재 픽셀을 중심으로 일정 영역의 평균 값을 샘플링하는 것을 활용, 모션 블러 생성 범위를 설정하기 위해 mipmap 설정된 rendertargetview를 사용한다.

```c++


bool RenderTexture::InitializeBackBufferWithMipMap_Array(ID3D11Device* device, int nNumArr)
{
	HRESULT result;

	D3D11_TEXTURE2D_DESC textureDesc; /// Setup the render target texture description.
	ZeroMemory(&textureDesc, sizeof(textureDesc)); /// Initialize the render target texture description.
	textureDesc.Width = SCREENWIDTH;
	textureDesc.Height = SCREENHEIGHT;
	textureDesc.MipLevels = 0;
	textureDesc.ArraySize = nNumArr;
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

	ppRTVs = new ID3D11RenderTargetView * [nNumArr];

	ppSRVs = new ID3D11ShaderResourceView * [nNumArr];

	for (int i = 0; i < nNumArr; ++i)
	{
		layerViewDesc.Texture2DArray.FirstArraySlice = i;

		result = device->CreateRenderTargetView(m_renderTargetTexture, &layerViewDesc, &ppRTVs[i]);
		if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerTexture2DArrayRenderTargetView'", L"Error", MB_OK | MB_ICONERROR); }
	}

	result = device->CreateShaderResourceView(m_renderTargetTexture, NULL, &ppSRVs[0]);
	if (FAILED(result)) { MessageBox(NULL, L"Error in generating 'LayerTexture2DArrayShaderResourceView'", L"Error", MB_OK | MB_ICONERROR); }

	D3D11_TEXTURE2D_DESC textureDesc_; /// Setup the render target texture description.
	ZeroMemory(&textureDesc_, sizeof(textureDesc_)); /// Initialize the render target texture description.
	textureDesc_.Width = SCREENWIDTH;
	textureDesc_.Height = SCREENHEIGHT;
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

```


<br>


현재 픽셀을 중심의 평균과 분산으로 표준편차를 계산하고, 이를 이용해 Search region 계산한다.

- σ(표준편차)= √(E_x [(x)^2 ]-(E_x [x])^2 ) = √("모션 벡터 제곱의 평균" - "모션 벡터 평균 " 의 제곱)

- min(E_x [x] - 2σ)< search region < max (E_x [x] + 2σ)

※ mipmap level에 따라 모션 블러 생성 범위가 정해지므로, 적절한 mipmap level 설정이 필요하다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/43f24b3d-f11e-46cb-951b-287c30c57107" alt width="100%">
<em>Motion blur</em>
</center>

<br>


```

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


```


<br>


그리고 원하는 샘플 횟수를 지정, 샘플 위치에 모션 벡터가 있을 때 해당 색상을 가져온다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/756624ed-7559-4c5b-ad85-c580d955d539" alt width="100%">
<em></em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/0bfc1145-1b84-4687-a778-0e645785c3e6" alt width="100%">
<em></em>
</center>

<br>

```

float Get_Random(float2 uv)//
{
	return frac(sin(dot(uv, float2(12.9898f, 78.233f) * 2.0f)) * 43758.5453f);
}
float2 rand_2_10(in float2 uv) {
	float noiseX = (frac(sin(dot(uv, float2(12.9898, 78.233) * 2.0)) * 43758.5453));
	float noiseY = sqrt(1 - noiseX * noiseX);
	return float2(noiseX, noiseY);
}


float2 currentPixelLine(float2 uv, float2 lineStart, float2 lineMotionVector, Texture2D motionVectorMap, Texture2D depthMap) {
	float2 failedLine = float2(-2.0f, -2.0f);
	float2 tempPosition = uv;
	for (int i = 0; i < maxSearchCount; i++) {
		uv = tempPosition - lineStart;//원점 방향으로 이동된 uv
		float changeduvX = uv.x * lineMotionVector.x + uv.y * lineMotionVector.y;//x 축으로 회전
		float changeduvY = uv.x * (-lineMotionVector.y) + uv.y * lineMotionVector.x;
		float rangeOfX = (lineMotionVector.x * lineMotionVector.x) + (lineMotionVector.y * lineMotionVector.y) + thresholdX; // a2 + b2 + threshold
		float rangeOfY = thresholdY;
		float currentPixelDepth = depthMap.SampleLevel(g_samPoint, lineStart, 0).x;
		float currentPixelMotionVectorZ = motionVectorMap.SampleLevel(g_samPoint, lineStart, 0).z;
		float Y = currentPixelDepth + (changeduvX * currentPixelMotionVectorZ / rangeOfX);
		float Z = currentPixelDepth / Y;
		if (changeduvX <= rangeOfX && changeduvX >= -thresholdX && changeduvY <= rangeOfY * Z && changeduvY >= -thresholdY * Z) {
			return lineMotionVector;
		}
		lineStart = tempPosition - lineMotionVector * (1.0f / randomPickNum);
		lineMotionVector = motionVectorMap.SampleLevel(g_samPoint, lineStart, 0).xy;
	}
	//완전실패, 최악의 경우
	return failedLine;
}


float4 findLineAndGetColor(float2 uv, Texture2D motionVectorMap, Texture2D depthMap, Texture2D colorMap)
{
	//return float3(1, 1, uv.x);

	float4 resultColor = float4(0.0f, 0.0f, 0.0f, 0.0f);
	float2 UV_1 = float2(0.0f, 0.0f);
	float2 UV_2 = float2(0.0f, 0.0f);
	float randomx = 0.0f;
	float randomy = 0.0f;
	float2 warpStart = float2(0.0f, 0.0f);
	int lineIndexPre = 0;
	float2 linePre[10];
	int lineIndexCur = 0;
	float2 lineCur[10];

	float2 totalLine[20];
	float2 zero = float2(0.0f, 0.0f);
	float2 texelSize = float2(10.0f / gWidth, 10.0 / gHeight);
	float threshold = distance(zero, texelSize);
	float temp = uv.x;

	int minify = 2;// mipmap레벨을 얼마나 줄일것인가 ?
	float highestMipmapLevel = 11.0f;// mipmap 레벨 최대 11
	float mipmapLevelT0 = 11.0f;
	float mipmapLevelT1 = 11.0f;


	float4 searchBoundary = motionBoundary(motionVectorMap, g_motionVector3DSquareMap, uv, 8.0f);
	int failCount = 0;
	for (int j = 0; j < randomPickNum; j++)
	{
		bool isFind = false;
		bool isFindCur = false;
		UV_1 = float2(uv.x + j * 0.0001357f, uv.y + j * 0.0004567f);
		UV_2 = float2(uv.x + j * 0.0002468f, uv.y + j * 0.0003456f);
		randomx = Get_Random(UV_1); //random -> return 0~0.999999
		randomy = Get_Random(UV_2);

		warpStart = float2(searchBoundary.x + randomx * abs(searchBoundary.z - searchBoundary.x),
			searchBoundary.y + randomy * abs(searchBoundary.w - searchBoundary.y));//motion boundary 내의 random한 점

		float2 lineMotionVector = motionVectorMap.SampleLevel(g_samPoint, warpStart, 0).xy;
		float2 foundLine = currentPixelLine(uv, warpStart, lineMotionVector, motionVectorMap, depthMap);
		if (lengthSquare(foundLine) + eps < 8.0f) 
		{
			for (int searchingIndex = 0; searchingIndex < lineIndexPre; searchingIndex++) 
			{
				if (abs(foundLine.x - linePre[searchingIndex].x) < eps && abs(foundLine.y - linePre[searchingIndex].y) < eps) 
				{
					isFind = true;
					break;
				}
			}
			if (!isFind) 
			{
				// 샘플수 저장
				linePre[lineIndexPre] = foundLine;
				lineIndexPre++;
			}
		}
	}
	

	float currentPixelDepth = 0.0f;
	float prevPixelDepth = 0.0f;
	int nNumOfSucess = 0;
	float3 sumColor = float3(0.0f, 0.0f, 0.0f);
	float3 sumDepth = float3(0.0f, 0.0f, 0.0f);
	float targetDepth;
	float2 randomNumber = rand_2_10(float2(uv.x, uv.y));

	float3 targetMotionVector;
	float3 targetStaticColor;
	float targetStaticDepth;

	targetMotionVector = motionVectorMap.SampleLevel(g_samPoint, uv, 0).xyz;
	targetStaticColor = colorMap.SampleLevel(g_samPoint, uv, 0).xyz;
	targetStaticDepth = depthMap.SampleLevel(g_samPoint, uv, 0).x;

	for (int i = 0; i < 8; i++) 
	{
		if (!(targetMotionVector.x == 0.0f && targetMotionVector.y == 0.0f))
		{
			targetMotionVector = g_motionVector3DMap.SampleLevel(g_samPoint, uv, 0).xyz;
			targetStaticColor = float4(g_Ref_Primary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
			targetStaticDepth = g_DepthMap.SampleLevel(g_samPoint, uv, 0).x;
		}
	}

	if (MAXSUBFRAME > 8)
	{
		if (!(targetMotionVector.x == 0.0f && targetMotionVector.y == 0.0f))
		{
			targetMotionVector = g_motionVector3DMap.SampleLevel(g_samPoint, uv, 0).xyz;
			targetStaticColor = float4(g_Ref_Secondary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
			targetStaticDepth = g_DepthMap.SampleLevel(g_samPoint, uv, 0).x;
		}
	}
	
	if (MAXSUBFRAME > 16)
	{
		if (!(targetMotionVector.x == 0.0f && targetMotionVector.y == 0.0f))
		{
			targetMotionVector = g_motionVector3DMap.SampleLevel(g_samPoint, uv, 0).xyz;
			targetStaticColor = float4(g_Ref_Tertiary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
			targetStaticDepth = g_DepthMap.SampleLevel(g_samPoint, uv, 0).x;
		}
	}

	if (MAXSUBFRAME > 24)
	{
		if (!(targetMotionVector.x == 0.0f && targetMotionVector.y == 0.0f))
		{
			targetMotionVector = g_motionVector3DMap.SampleLevel(g_samPoint, uv, 0).xyz;
			targetStaticColor = float4(g_Ref_Quaternary_colorMap.Load(int4(IN.tex * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
			targetStaticDepth = g_DepthMap.SampleLevel(g_samPoint, uv, 0).x;
		}
	}

	for (int k = 0; k < samplingTime; k++) //시간에서 탐색
	{
		randomNumber = rand_2_10(randomNumber);
		float3 targetColor = float3(-1.0f, 0.0f, 0.0f);
		targetDepth = 1000.0f;

		for (int p = 0; p < lineIndexPre; p++) 
		{
			float2 prevPixelPos = uv - linePre[p] * (k+ randomNumber) / samplingTime;
			float3 prevPixelMotionVector = motionVectorMap.SampleLevel(g_samPoint, prevPixelPos, 0).xyz;
			prevPixelDepth = depthMap.SampleLevel(g_samPoint, prevPixelPos, 0).x;

			if (distance(linePre[p], prevPixelMotionVector.xy) < 0.0025f) 
			{
				prevPixelMotionVector.z *= (float)(k+ randomNumber) / samplingTime;

				if (prevPixelDepth + prevPixelMotionVector.z <= targetDepth) 
				{
					targetDepth = prevPixelDepth + prevPixelMotionVector.z;
					targetColor = float4(g_Ref_Primary_colorMap.Load(int4(prevPixelPos * float2(SCREENWIDTH, SCREENHEIGHT), i, 0)).xyz, 0);
				}
			}
		}

		//배경하고 같이 안나오는거, 대신 없으면 한쪽 안나옴
		if (targetMotionVector.x == 0.0f && targetMotionVector.y == 0.0f && targetStaticDepth < targetDepth) 
		{
			targetDepth = targetStaticDepth;
			targetColor = targetStaticColor;
		}
		if (targetColor.r >= 0.0f) 
		{
			sumColor += targetColor;
			sumDepth += float3(targetDepth, targetDepth, targetDepth);
			nNumOfSucess++;
		}
	}
	
	float3 bluredColor = float3(0, 0, 0);
	if (nNumOfSucess > 0)
		bluredColor = sumColor / nNumOfSucess;
	float weightOfSuccessRate = (float)nNumOfSucess / samplingTime;
	return float4(bluredColor, weightOfSuccessRate);
}

```


<br>



랜덤한 위치에서의 샘플로 인해, 일부분에서 모션블러 효과가 적용되지 않는 경우가 발생한다. 이는 샘플 횟수를 늘리면 해결할 수 있으나, 성능 저하가 발생하므로 이를 디노이징을 통해 해결한다.

```

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

   sum /= cum_w;
   return float4(sum, 1.0f);
}


```

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/134ab9c8-8910-42e4-aa2b-4e19546a0557" alt width="100%">
<em>Denoising</em>
</center>

<br>

이를 적용한 코드

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

렌더링을 진행, 행렬을 업데이트하여 물체의 위치를 변화시키고 다시 렌더링을 반복한다. 이러한 작업이 끝난 후 셰이더를 호출한다.

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

    for (int layer = 0; layer < N_LAYER; layer++)
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
        for (int layer = 0; layer < N_LAYER; layer++)
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
        for (int layer = 0; layer < N_LAYER; layer++)
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
        for (int layer = 0; layer < N_LAYER; layer++)
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



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/70c4144b-5b4e-449c-9673-8be76e17b2e7" alt width=500>
<em></em>
</center>

<br>


그리고, 이를 테스트 환경(CPU( i9-10900K) / Ram (128GB) / GPU (RTX 3080 10GB) / Direct11/해상도 (1920 * 1080))에서 accumulation과 비교했을 때, 아래와 같은 결과를 얻을 수 있다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/62f82a9a-da9a-4707-b4b9-2b8fa002bf37" alt width=500>
<em></em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/dffb909c-05af-41ec-a400-384eb5ad362f" alt width=500>
<em></em>
</center>


<br>
<br>
<br>
