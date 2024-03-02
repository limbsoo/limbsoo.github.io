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


앞서 구현한 render texutre를 사용하여 environment mapping을 구현한다.

```c++

shadow.vs
cbuffer MatrixBuffer
{
    matrix worldMatrix;
    matrix viewMatrix;
    matrix projectionMatrix;
    matrix lightViewMatrix;
    matrix lightProjectionMatrix;
};


cbuffer LightBuffer
{
    float3 lightPosition;
    float padding;
};


struct VertexInputType
{
    float4 position : POSITION;
    float2 texure : TEXCOORD;
    float3 normal : NORMAL;
    float4 camera : BLENDWEIGHT;
};

struct PixelInputType
{
    float4 position : SV_POSITION;
    float2 texure : TEXCOORD0;
    float3 normal : TEXCOORD3;
    float4 lightViewPosition : TEXCOORD1;
    float3 lightDir : TEXCOORD2;
    float3 rVector : TEXCOORD4;
};

PixelInputType ShadowVertexShader(VertexInputType input)
{
    PixelInputType output;
    float4 worldPosition;

    //makeRVector
    float cos;
    float4 s;
    float3 cos1;

    //Vector4 normal(m_worldVertexNormal[i], 1);
    output.normal = mul(input.normal, (float3x3)worldMatrix);

    //Vector4 postition(m_worldVertex[i], 1);
    output.position = mul(input.position, worldMatrix);

    //postition = eye - postition;
    output.position = input.camera - output.position;

    //postition = normalize(postition);
    output.position = normalize(output.position);

    //cos = normal * postition;
    //cos = output.normal * output.position;

    // 적절한 행렬 계산을 위해 위치 벡터를 4 단위로 변경합니다.
    input.position.w = 1.0f;

    // 월드, 뷰 및 투영 행렬에 대한 정점의 위치를 ??계산합니다.
    output.position = mul(input.position, worldMatrix);
    output.position = mul(output.position, viewMatrix);
    output.position = mul(output.position, projectionMatrix);

    // 광원에 의해 보았을 때 vertice의 위치를 ??계산합니다.
    output.lightViewPosition = mul(input.position, worldMatrix);
    output.lightViewPosition = mul(output.lightViewPosition, lightViewMatrix);
    output.lightViewPosition = mul(output.lightViewPosition, lightProjectionMatrix);

    // 픽셀 쉐이더의 텍스처 좌표를 저장한다.
    output.texure = input.texure;

    // 월드 행렬에 대해서만 법선 벡터를 계산합니다.
    output.normal = mul(input.normal, (float3x3)worldMatrix);

    // 법선 벡터를 정규화합니다.
    output.normal = normalize(output.normal);

    // 세계의 정점 위치를 계산합니다.
    worldPosition = mul(input.position, worldMatrix);

    // 빛의 위치와 세계의 정점 위치를 기반으로 빛의 위치를 ??결정합니다.
    //makeWorldLightingDepth
    output.lightDir = lightPosition.xyz - worldPosition.xyz;

    // 라이트 위치 벡터를 정규화합니다.
    output.lightDir = normalize(output.lightDir);

    cos1.x = output.normal.x * worldPosition.x;
    cos1.y = output.normal.y * worldPosition.y;
    cos1.z = output.normal.z * worldPosition.z;

    cos = cos1.x + cos1.y + cos1.z;

    s.x = output.normal.x * cos;
    s.y = output.normal.y * cos;
    s.z = output.normal.z * cos;

    s = (s * 2) - worldPosition;

    output.rVector.x = s.x;
    output.rVector.y = s.y;
    output.rVector.z = s.z;

    return output;
}



```

<br>

```c++
shadow.ps


//셰이더 모델 5(SM5.0) 리소스 구문은 register 키워드를 사용하여 리소스에 대한 중요한 정보를 HLSL 컴파일러에 릴레이합니다. 

//메모리에 쉐이더 리소스를 주겠다
Texture2D shaderTexture : register(t0);
Texture2D depthMapTexture : register(t1);
Texture2D sphericalTexture : register(t2);

//s ? 샘플러
SamplerState SampleTypeClamp : register(s0);
SamplerState SampleTypeWrap  : register(s1);


cbuffer LightBuffer
{
	float4 ambientColor;
	float4 diffuseColor;
};


struct PixelInputType
{
	float4 position : SV_POSITION;
	float2 texure : TEXCOORD0;
	float3 normal : NORMAL0;
	float4 lightViewPosition : TEXCOORD1;
	float3 lightDir : TEXCOORD2;

	float3 rVector : NORMAL1;
};

float4 ShadowPixelShader(PixelInputType input) : SV_TARGET
{
	float bias;
	float4 sphereColor, color;
	float2 projectedTexureCoord;
	float depthValue;
	float lightDepthValue;
    float lightIntensity;
	float4 textureColor;

	float4 sphericalTextureColor;


	//이 텍스처 좌표 위치에서 샘플러를 사용하여 텍스처에서 픽셀 색상을 샘플링합니다.
	textureColor = shaderTexture.Sample(SampleTypeWrap, input.texure);
	input.rVector.y *= -1;

	float scalaN;
	scalaN = sqrt(pow(input.rVector.x, 2) + pow(input.rVector.y, 2) + pow(input.rVector.z + 1, 2));

	sphericalTextureColor.x = (input.rVector.x / scalaN + 1) / 2;
	sphericalTextureColor.y = (input.rVector.y / scalaN + 1) / 2;
	sphereColor = sphericalTexture.Sample(SampleTypeWrap, sphericalTextureColor);

	// 부동 소수점 정밀도 문제를 해결할 바이어스 값을 설정합니다.
	bias = 0.001f;

	//모든 픽셀에 대해 기본 출력 색상을 주변 광원 값으로 설정합니다.
    color = ambientColor;

	//투영 된 텍스처 좌표를 계산합니다.
	// tu와 tv좌표가 -1과 1사이의 값을 가지게 되기 때문에 2를 더하고 0.5를 곱하여 0과 1사이의 범위로 맞춰줍니다.
	projectedTexureCoord.x = input.lightViewPosition.x / input.lightViewPosition.w / 2.0f + 0.5f;
	projectedTexureCoord.y = -input.lightViewPosition.y / input.lightViewPosition.w / 2.0f + 0.5f;

	 //투영 된 좌표가 0에서 1 범위에 있는지 결정합니다. 그렇다면이 픽셀은 빛의 관점에 있습니다.
	if((saturate(projectedTexureCoord.x) == projectedTexureCoord.x) && (saturate(projectedTexureCoord.y) == projectedTexureCoord.y))
	{

		depthValue = depthMapTexture.Sample(SampleTypeClamp, projectedTexureCoord).r;

		// 빛의 깊이를 계산합니다.
		lightDepthValue = input.lightViewPosition.z / input.lightViewPosition.w;

		lightDepthValue = lightDepthValue - bias;

		if(lightDepthValue < depthValue)
		{

			lightIntensity = saturate(dot(input.normal, input.lightDir));

		    if(lightIntensity > 0.0f)
			{
				color += (diffuseColor * lightIntensity* textureColor)+ sphereColor - 0.2f;
				// 최종 빛의 색상을 채웁니다.
				color = saturate(color);
			}
		}
	}

    return sphereColor;
}


```

<br>
<br>
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/53431841-f77c-4503-bc44-80c6d6eeee5f" alt width=500>
<em>return float4(rVector.xy,0,1)</em>
</center>

<br>
<br>

# 출력
---

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7830be2f-c048-4b22-a897-18cd4379d093" alt width=500>
<em>출력 이미지</em>
</center>

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/b7f570c6-85f0-4db5-a29f-8048f34a245b" alt width=500>
<em>빛 강도 조절</em>
</center>
<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/cb963146-274c-44c9-b851-be4b2cfb0cf6" alt width=500>
<em>쉐도우 맵 기반</em>
</center>

<br>
<br>
<br>

출처: www.rastertek.com/tutdx11win10.html

