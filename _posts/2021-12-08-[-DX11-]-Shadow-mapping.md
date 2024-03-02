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


백버퍼에 렌더링되는 장면을 텍스처에 그린다.

```
##ShadowMap pseudo Code

#1PASS

카메라의 eye 값을 광원의 position과 일치시킨다.
광원의 위치에서의 NDC 행렬을 생성한다. 

Render()
{

	RenderToTexture //전체 장면을 먼저 텍스처로 렌더링합니다.
	{
		setRenderTarget // 렌더 대상을 백버퍼 -> 텍스쳐  
		
		ShadowShader::RenderScene
		{ // input: geometryscene(model의 모양)

			vertexShader // 물체의 정점 정보 function 출력 점 입력 점 
			{
				입력점(vector)에 행렬을 곱해 점으로 출력한다. 
			}
		
			pixelShader 
			{
				입력:(resterizer가 보관한)vertex 값
				그점의 depth를 return한다. 
			}
		}
		setRenderTarget // 렌더 대상을 텍스쳐 -> 백버퍼로 되돌림 
	}
}


#PASS 2

카메라를 원래의 eye 위치에 위치시킨다. 
카메라의 위치에서의 NDC 행렬을 생성한다. 

Render()
{
	ShadowShader::RenderScene
	{ // input: geometryscene(model의 모양)) 

		vertexShader // 물체의 정점 정보 (function)
		{
			입력점(vector)에 행렬을 곱해 점으로 출력한다.  
		}
	
		pixelShader 
		{
			입력:(resterizer가 보관한)vertex 값 (NDC좌표)
			  
		    각 픽셀마다position(벡터)으로 받은 값에 MscreenMprojectionMlightviewMview-1Mprojection-1Mscreen-1을 곱하여 light공간에서 camera 공간으로 이동하여,shadowMap에서의 좌표를 구한다. 
			RenderToTexture의 pixel의 depth 값과  RenderScene의 pixel의 depth 값과 비교하여  shadow color < lighted color일때  shadow color를 출력, else 일때  lighted color값을 출력한다. 
		}
	}
	
}

```

<br>

광원 시점에서의 depthmap을 만들기 위한 shadow shader를 정의한다.
현재 시점을 광원으로 옮겨 깊이값을 계산한다.

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
    float2 tex : TEXCOORD0;
	float3 normal : NORMAL;
};

struct PixelInputType
{
    float4 position : SV_POSITION;
    float2 tex : TEXCOORD0;
	float3 normal : NORMAL;
    float4 lightViewPosition : TEXCOORD1;
	float3 lightPos : TEXCOORD2;
};


PixelInputType ShadowVertexShader(VertexInputType input)
{
    PixelInputType output;
	float4 worldPosition;
    
	// 적절한 행렬 계산을 위해 위치 벡터를 4 단위로 변경합니다.
    input.position.w = 1.0f;

	// 월드, 뷰 및 투영 행렬에 대한 정점의 위치를 ​​계산합니다.
    output.position = mul(input.position, worldMatrix);
    output.position = mul(output.position, viewMatrix);
    output.position = mul(output.position, projectionMatrix);
    
	// 광원에 의해 보았을 때 vertice의 위치를 ​​계산합니다.
    output.lightViewPosition = mul(input.position, worldMatrix);
    output.lightViewPosition = mul(output.lightViewPosition, lightViewMatrix);
    output.lightViewPosition = mul(output.lightViewPosition, lightProjectionMatrix);

	// 픽셀 쉐이더의 텍스처 좌표를 저장한다.
    output.tex = input.tex;
    
	// 월드 행렬에 대해서만 법선 벡터를 계산합니다.
    output.normal = mul(input.normal, (float3x3)worldMatrix);
	
    // 법선 벡터를 정규화합니다.
    output.normal = normalize(output.normal);

    // 세계의 정점 위치를 계산합니다.
    worldPosition = mul(input.position, worldMatrix);

    // 빛의 위치와 세계의 정점 위치를 기반으로 빛의 위치를 ​​결정합니다.
    output.lightPos = lightPosition.xyz - worldPosition.xyz;

    // 라이트 위치 벡터를 정규화합니다.
    output.lightPos = normalize(output.lightPos);

	return output;
}

```

<br>


깊이 버퍼를 샘플링하기 위해 clamp를 사용, 일정 이상의 값이 샘플되는 것을 방지한다.
그리고 투영된 좌표를 계산하여 그림자에 속하는 픽셀인지 빛을 받는 픽셀인지 판단하고 깊이값을 비교하여 빛을 계산한다.

```c++
shadow.ps

Texture2D shaderTexture : register(t0);
Texture2D depthMapTexture : register(t1);

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
    float2 texutre : TEXCOORD0;
	float3 normal : NORMAL;
    float4 lightViewPosition : TEXCOORD1;
	float3 lightPos : TEXCOORD2;
};

float4 ShadowPixelShader(PixelInputType input) : SV_TARGET
{
	float bias;
    float4 color;
	float2 projectTexCoord;
	float depthValue;
	float lightDepthValue;
    float lightIntensity;
	float4 textureColor;


	// 부동 소수점 정밀도 문제를 해결할 바이어스 값을 설정합니다.
	bias = 0.001f;

	// 모든 픽셀에 대해 기본 출력 색상을 주변 광원 값으로 설정합니다.
    color = ambientColor;

	// 투영 된 텍스처 좌표를 계산합니다.
	projectTexCoord.x =  input.lightViewPosition.x / input.lightViewPosition.w / 2.0f + 0.5f;
	projectTexCoord.y = -input.lightViewPosition.y / input.lightViewPosition.w / 2.0f + 0.5f;

	// 투영 된 좌표가 0에서 1 범위에 있는지 결정합니다. 그렇다면이 픽셀은 빛의 관점에 있습니다.
	if((saturate(projectTexCoord.x) == projectTexCoord.x) && (saturate(projectTexCoord.y) == projectTexCoord.y))
	{
		// 투영 된 텍스처 좌표 위치에서 샘플러를 사용하여 깊이 텍스처에서 섀도우 맵 깊이 값을 샘플링합니다.
		depthValue = depthMapTexture.Sample(SampleTypeClamp, projectTexCoord).r;

		// 빛의 깊이를 계산합니다.
		lightDepthValue = input.lightViewPosition.z / input.lightViewPosition.w;

		// lightDepthValue에서 바이어스를 뺍니다.
		lightDepthValue = lightDepthValue - bias;

		// 섀도우 맵 값의 깊이와 빛의 깊이를 비교하여이 픽셀을 음영 처리할지 조명할지 결정합니다.
		// 빛이 객체 앞에 있으면 픽셀을 비추고, 그렇지 않으면 객체 (오클 루더)가 그림자를 드리 우기 때문에이 픽셀을 그림자로 그립니다.
		if(lightDepthValue < depthValue)
		{
		    // 이 픽셀의 빛의 양을 계산합니다.
			lightIntensity = saturate(dot(input.normal, input.lightPos));

		    if(lightIntensity > 0.0f)
			{
				// 확산 색과 광 강도의 양에 따라 최종 확산 색을 결정합니다.
				color += (diffuseColor * lightIntensity);

				// 최종 빛의 색상을 채웁니다.
				color = saturate(color);
			}
		}
	}

	// 이 텍스처 좌표 위치에서 샘플러를 사용하여 텍스처에서 픽셀 색상을 샘플링합니다.
	textureColor = shaderTexture.Sample(SampleTypeWrap, input.texutre);

	// 빛과 텍스처 색상을 결합합니다.
	color = color * textureColor;

    return color;
}
```

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/2d2c85c6-ede1-4cf2-ab41-dc3802698c9f" alt width="100%">
<em>깊이 맵</em>
</center>



<br>

쉐이더를 사용하기 위한 ShadowShaderClass를 정의한다.

```c++
ShadowShaderClass.h

class ShadowShaderClass
{
private:
	struct MatrixBufferType
	{
		XMMATRIX world;
		XMMATRIX view;
		XMMATRIX projection;
		XMMATRIX lightView;
		XMMATRIX lightProjection;
	};

	struct LightBufferType
	{
		XMFLOAT4 ambientColor;
		XMFLOAT4 diffuseColor;
	};

	struct LightBufferType2
	{
		XMFLOAT3 lightPosition;
		float padding;
	};
public:
	ShadowShaderClass();
	ShadowShaderClass(const ShadowShaderClass&);
	~ShadowShaderClass();

	bool Initialize(ID3D11Device*, HWND);
	void Shutdown();
	bool Render(ID3D11DeviceContext*, int, XMMATRIX, XMMATRIX, XMMATRIX, XMMATRIX, XMMATRIX, ID3D11ShaderResourceView*,
		ID3D11ShaderResourceView*, XMFLOAT3, XMFLOAT4, XMFLOAT4);

private:
	bool InitializeShader(ID3D11Device*, HWND, const WCHAR*, const WCHAR*);
	void ShutdownShader();
	void OutputShaderErrorMessage(ID3D10Blob*, HWND, const WCHAR*);

	bool SetShaderParameters(ID3D11DeviceContext*, XMMATRIX, XMMATRIX, XMMATRIX, XMMATRIX, XMMATRIX, ID3D11ShaderResourceView*,
		ID3D11ShaderResourceView*, XMFLOAT3, XMFLOAT4, XMFLOAT4);
	void RenderShader(ID3D11DeviceContext*, int);

private:
	ID3D11VertexShader* m_vertexShader = nullptr;
	ID3D11PixelShader* m_pixelShader = nullptr;
	ID3D11InputLayout* m_layout = nullptr;
	ID3D11SamplerState* m_sampleStateWrap = nullptr;
	ID3D11SamplerState* m_sampleStateClamp = nullptr;
	ID3D11Buffer* m_matrixBuffer = nullptr;
	ID3D11Buffer* m_lightBuffer = nullptr;
	ID3D11Buffer* m_lightBuffer2 = nullptr;
};

```

<br>

쉐이더를 초기화하고 렌더링에 사용

```c++
ShadowShaderClass.cpp

...

bool ShadowShaderClass::Initialize(ID3D11Device* device, HWND hwnd)
{
	// 정점 및 픽셀 쉐이더를 초기화합니다.
	return InitializeShader(device, hwnd, L"../DX11_Shadowmap/hlsl/shadow_vs.hlsl", L"../DX11_Shadowmap/hlsl/shadow_ps.hlsl");
}


bool ShadowShaderClass::Render(ID3D11DeviceContext* deviceContext, int indexCount, XMMATRIX worldMatrix, XMMATRIX viewMatrix,
	XMMATRIX projectionMatrix, XMMATRIX lightViewMatrix, XMMATRIX lightProjectionMatrix,
	ID3D11ShaderResourceView* texture, ID3D11ShaderResourceView* depthMapTexture, XMFLOAT3 lightPosition,
	XMFLOAT4 ambientColor, XMFLOAT4 diffuseColor)
{
	// 렌더링에 사용할 셰이더 매개 변수를 설정합니다.
	if (!SetShaderParameters(deviceContext, worldMatrix, viewMatrix, projectionMatrix, lightViewMatrix, lightProjectionMatrix, texture,
		depthMapTexture, lightPosition, ambientColor, diffuseColor))
	{
		return false;
	}

	// 설정된 버퍼를 셰이더로 렌더링한다.
	RenderShader(deviceContext, indexCount);

	return true;
}


bool ShadowShaderClass::InitializeShader(ID3D11Device* device, HWND hwnd, const WCHAR* vsFilename, const WCHAR* psFilename)
{
	HRESULT result;
	ID3D10Blob* errorMessage = nullptr;

	// 버텍스 쉐이더 코드를 컴파일한다.
	ID3D10Blob* vertexShaderBuffer = nullptr;
	result = D3DCompileFromFile(vsFilename, NULL, NULL, "ShadowVertexShader", "vs_5_0", D3D10_SHADER_ENABLE_STRICTNESS, 0,
		&vertexShaderBuffer, &errorMessage);
	if (FAILED(result))
	{
		// 셰이더 컴파일 실패시 오류메시지를 출력합니다.
		if (errorMessage)
		{
			OutputShaderErrorMessage(errorMessage, hwnd, vsFilename);
		}
		// 컴파일 오류가 아니라면 셰이더 파일을 찾을 수 없는 경우입니다.
		else
		{
			MessageBox(hwnd, vsFilename, L"Missing Shader File", MB_OK);
		}

		return false;
	}

	// 픽셀 쉐이더 코드를 컴파일한다.
	ID3D10Blob* pixelShaderBuffer = nullptr;
	result = D3DCompileFromFile(psFilename, NULL, NULL, "ShadowPixelShader", "ps_5_0", D3D10_SHADER_ENABLE_STRICTNESS, 0,
		&pixelShaderBuffer, &errorMessage);
	if (FAILED(result))
	{
		// 셰이더 컴파일 실패시 오류메시지를 출력합니다.
		if (errorMessage)
		{
			OutputShaderErrorMessage(errorMessage, hwnd, psFilename);
		}
		// 컴파일 오류가 아니라면 셰이더 파일을 찾을 수 없는 경우입니다.
		else
		{
			MessageBox(hwnd, psFilename, L"Missing Shader File", MB_OK);
		}

		return false;
	}

	// 버퍼로부터 정점 셰이더를 생성한다.
	result = device->CreateVertexShader(vertexShaderBuffer->GetBufferPointer(), vertexShaderBuffer->GetBufferSize(), NULL, &m_vertexShader);
	if (FAILED(result))
	{
		return false;
	}

	// 버퍼에서 픽셀 쉐이더를 생성합니다.
	result = device->CreatePixelShader(pixelShaderBuffer->GetBufferPointer(), pixelShaderBuffer->GetBufferSize(), NULL, &m_pixelShader);
	if (FAILED(result))
	{
		return false;
	}

	// 정점 입력 레이아웃 구조체를 설정합니다.
	// 이 설정은 ModelClass와 셰이더의 VertexType 구조와 일치해야합니다.
	D3D11_INPUT_ELEMENT_DESC polygonLayout[3];
	polygonLayout[0].SemanticName = "POSITION";
	polygonLayout[0].SemanticIndex = 0;
	polygonLayout[0].Format = DXGI_FORMAT_R32G32B32_FLOAT;
	polygonLayout[0].InputSlot = 0;
	polygonLayout[0].AlignedByteOffset = 0;
	polygonLayout[0].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	polygonLayout[0].InstanceDataStepRate = 0;

	polygonLayout[1].SemanticName = "TEXCOORD";
	polygonLayout[1].SemanticIndex = 0;
	polygonLayout[1].Format = DXGI_FORMAT_R32G32_FLOAT;
	polygonLayout[1].InputSlot = 0;
	polygonLayout[1].AlignedByteOffset = D3D11_APPEND_ALIGNED_ELEMENT;
	polygonLayout[1].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	polygonLayout[1].InstanceDataStepRate = 0;

	polygonLayout[2].SemanticName = "NORMAL";
	polygonLayout[2].SemanticIndex = 0;
	polygonLayout[2].Format = DXGI_FORMAT_R32G32B32_FLOAT;
	polygonLayout[2].InputSlot = 0;
	polygonLayout[2].AlignedByteOffset = D3D11_APPEND_ALIGNED_ELEMENT;
	polygonLayout[2].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	polygonLayout[2].InstanceDataStepRate = 0;

	// 레이아웃의 요소 수를 가져옵니다.
	UINT numElements = sizeof(polygonLayout) / sizeof(polygonLayout[0]);

	// 정점 입력 레이아웃을 만듭니다.
	result = device->CreateInputLayout(polygonLayout, numElements, vertexShaderBuffer->GetBufferPointer(),
		vertexShaderBuffer->GetBufferSize(), &m_layout);
	if (FAILED(result))
	{
		return false;
	}

	// 더 이상 사용되지 않는 정점 셰이더 퍼버와 픽셀 셰이더 버퍼를 해제합니다.
	vertexShaderBuffer->Release();
	vertexShaderBuffer = 0;

	pixelShaderBuffer->Release();
	pixelShaderBuffer = 0;

	// 텍스처 샘플러 상태 구조체를 생성 및 설정합니다.
	D3D11_SAMPLER_DESC samplerDesc;
	samplerDesc.Filter = D3D11_FILTER_MIN_MAG_MIP_LINEAR;
	samplerDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
	samplerDesc.MipLODBias = 0.0f;
	samplerDesc.MaxAnisotropy = 1;
	samplerDesc.ComparisonFunc = D3D11_COMPARISON_ALWAYS;
	samplerDesc.BorderColor[0] = 0;
	samplerDesc.BorderColor[1] = 0;
	samplerDesc.BorderColor[2] = 0;
	samplerDesc.BorderColor[3] = 0;
	samplerDesc.MinLOD = 0;
	samplerDesc.MaxLOD = D3D11_FLOAT32_MAX;

	// 텍스처 샘플러 상태를 만듭니다.
	result = device->CreateSamplerState(&samplerDesc, &m_sampleStateWrap);
	if (FAILED(result))
	{
		return false;
	}

	// 클램프 텍스처 샘플러 상태 설명을 만듭니다.
	samplerDesc.AddressU = D3D11_TEXTURE_ADDRESS_CLAMP;
	samplerDesc.AddressV = D3D11_TEXTURE_ADDRESS_CLAMP;
	samplerDesc.AddressW = D3D11_TEXTURE_ADDRESS_CLAMP;

	// 텍스처 샘플러 상태를 만듭니다.
	result = device->CreateSamplerState(&samplerDesc, &m_sampleStateClamp);
	if (FAILED(result))
	{
		return false;
	}

	D3D11_BUFFER_DESC matrixBufferDesc;
	matrixBufferDesc.Usage = D3D11_USAGE_DYNAMIC;
	matrixBufferDesc.ByteWidth = sizeof(MatrixBufferType);
	matrixBufferDesc.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
	matrixBufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
	matrixBufferDesc.MiscFlags = 0;
	matrixBufferDesc.StructureByteStride = 0;

	// 상수 버퍼 포인터를 만들어 이 클래스에서 정점 셰이더 상수 버퍼에 접근할 수 있게 합니다.
	result = device->CreateBuffer(&matrixBufferDesc, NULL, &m_matrixBuffer);
	if (FAILED(result))
	{
		return false;
	}

	// 픽셀 쉐이더에있는 광원 동적 상수 버퍼의 설명을 설정합니다.
	// D3D11_BIND_CONSTANT_BUFFER를 사용하면 ByteWidth가 항상 16의 배수 여야하며 그렇지 않으면 CreateBuffer가 실패합니다.
	D3D11_BUFFER_DESC lightBufferDesc;
	lightBufferDesc.Usage = D3D11_USAGE_DYNAMIC;
	lightBufferDesc.ByteWidth = sizeof(LightBufferType);
	lightBufferDesc.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
	lightBufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
	lightBufferDesc.MiscFlags = 0;
	lightBufferDesc.StructureByteStride = 0;

	// 이 클래스 내에서 정점 셰이더 상수 버퍼에 액세스 할 수 있도록 상수 버퍼 포인터를 만듭니다.
	result = device->CreateBuffer(&lightBufferDesc, NULL, &m_lightBuffer);
	if (FAILED(result))
	{
		return false;
	}

	// 버텍스 쉐이더에있는 광원 동적 상수 버퍼의 설명을 설정합니다.
	D3D11_BUFFER_DESC lightBufferDesc2;
	lightBufferDesc2.Usage = D3D11_USAGE_DYNAMIC;
	lightBufferDesc2.ByteWidth = sizeof(LightBufferType2);
	lightBufferDesc2.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
	lightBufferDesc2.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
	lightBufferDesc2.MiscFlags = 0;
	lightBufferDesc2.StructureByteStride = 0;

	// 이 클래스 내에서 정점 셰이더 상수 버퍼에 액세스 할 수 있도록 상수 버퍼 포인터를 만듭니다.
	result = device->CreateBuffer(&lightBufferDesc2, NULL, &m_lightBuffer2);
	if (FAILED(result))
	{
		return false;
	}

	return true;
}

...

bool ShadowShaderClass::SetShaderParameters(ID3D11DeviceContext* deviceContext, XMMATRIX worldMatrix, XMMATRIX viewMatrix,
	XMMATRIX projectionMatrix, XMMATRIX lightViewMatrix, XMMATRIX lightProjectionMatrix,
	ID3D11ShaderResourceView* texture, ID3D11ShaderResourceView* depthMapTexture, XMFLOAT3 lightPosition,
	XMFLOAT4 ambientColor, XMFLOAT4 diffuseColor)
{
	// 행렬을 transpose하여 셰이더에서 사용할 수 있게 합니다
	worldMatrix = XMMatrixTranspose(worldMatrix);
	viewMatrix = XMMatrixTranspose(viewMatrix);
	projectionMatrix = XMMatrixTranspose(projectionMatrix);
	lightViewMatrix = XMMatrixTranspose(lightViewMatrix);
	lightProjectionMatrix = XMMatrixTranspose(lightProjectionMatrix);

	// 상수 버퍼의 내용을 쓸 수 있도록 잠급니다.
	D3D11_MAPPED_SUBRESOURCE mappedResource;
	if (FAILED(deviceContext->Map(m_matrixBuffer, 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedResource)))
	{
		return false;
	}

	// 상수 버퍼의 데이터에 대한 포인터를 가져옵니다.
	MatrixBufferType* dataPtr = (MatrixBufferType*)mappedResource.pData;

	// 상수 버퍼에 행렬을 복사합니다.
	dataPtr->world = worldMatrix;
	dataPtr->view = viewMatrix;
	dataPtr->projection = projectionMatrix;
	dataPtr->lightView = lightViewMatrix;
	dataPtr->lightProjection = lightProjectionMatrix;

	// 상수 버퍼의 잠금을 풉니다.
	deviceContext->Unmap(m_matrixBuffer, 0);

	// 정점 셰이더에서의 상수 버퍼의 위치를 설정합니다.
	unsigned int bufferNumber = 0;

	// 마지막으로 정점 셰이더의 상수 버퍼를 바뀐 값으로 바꿉니다.
	deviceContext->VSSetConstantBuffers(bufferNumber, 1, &m_matrixBuffer);

	// 픽셀 셰이더에서 셰이더 텍스처 리소스를 설정합니다.
	deviceContext->PSSetShaderResources(0, 1, &texture);
	deviceContext->PSSetShaderResources(1, 1, &depthMapTexture);

	// light constant buffer를 잠글 수 있도록 기록한다.
	if (FAILED(deviceContext->Map(m_lightBuffer, 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedResource)))
	{
		return false;
	}

	// 상수 버퍼의 데이터에 대한 포인터를 가져옵니다.
	LightBufferType* dataPtr2 = (LightBufferType*)mappedResource.pData;

	// 조명 변수를 상수 버퍼에 복사합니다.
	dataPtr2->ambientColor = ambientColor;
	dataPtr2->diffuseColor = diffuseColor;

	// 상수 버퍼의 잠금을 해제합니다.
	deviceContext->Unmap(m_lightBuffer, 0);

	// 픽셀 쉐이더에서 광원 상수 버퍼의 위치를 ??설정합니다.
	bufferNumber = 0;

	// 마지막으로 업데이트 된 값으로 픽셀 쉐이더에서 광원 상수 버퍼를 설정합니다.
	deviceContext->PSSetConstantBuffers(bufferNumber, 1, &m_lightBuffer);

	// 쓸 수 있도록 두 번째 빛 상수 버퍼를 잠급니다.
	if (FAILED(deviceContext->Map(m_lightBuffer2, 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedResource)))
	{
		return false;
	}

	// 상수 버퍼의 데이터에 대한 포인터를 가져옵니다.
	LightBufferType2* dataPtr3 = (LightBufferType2*)mappedResource.pData;

	// 조명 변수를 상수 버퍼에 복사합니다.
	dataPtr3->lightPosition = lightPosition;
	dataPtr3->padding = 0.0f;

	// 상수 버퍼의 잠금을 해제합니다.
	deviceContext->Unmap(m_lightBuffer2, 0);

	// 버텍스 쉐이더에서 라이트 상수 버퍼의 위치를 ??설정합니다.
	bufferNumber = 1;

	// 마지막으로 업데이트 된 값으로 픽셀 쉐이더에서 광원 상수 버퍼를 설정합니다.
	deviceContext->VSSetConstantBuffers(bufferNumber, 1, &m_lightBuffer2);

	return true;
}


void ShadowShaderClass::RenderShader(ID3D11DeviceContext* deviceContext, int indexCount)
{
	// 정점 입력 레이아웃을 설정합니다.
	deviceContext->IASetInputLayout(m_layout);

	// 삼각형을 그릴 정점 셰이더와 픽셀 셰이더를 설정합니다.
	deviceContext->VSSetShader(m_vertexShader, NULL, 0);
	deviceContext->PSSetShader(m_pixelShader, NULL, 0);

	// 픽셀 쉐이더에서 샘플러 상태를 설정합니다.
	deviceContext->PSSetSamplers(0, 1, &m_sampleStateClamp);
	deviceContext->PSSetSamplers(1, 1, &m_sampleStateWrap);

	// 삼각형을 그립니다.
	deviceContext->DrawIndexed(indexCount, 0, 0);
}

```

<br>

LightClass에 텍스처로 shadowmap을 그리기 위해 투영 정보를 전달

```c++
LightClass.cpp
...

void LightClass::GenerateViewMatrix()
{
	// 위쪽을 가리키는 벡터를 설정합니다.
	XMFLOAT3 up = XMFLOAT3(0.0f, 1.0f, 0.0f);

	XMVECTOR upVector = XMLoadFloat3(&up);
	XMVECTOR positionVector = XMLoadFloat3(&m_position);
	XMVECTOR lookAtVector = XMLoadFloat3(&m_lookAt);

	// 세 벡터로부터 뷰 행렬을 만듭니다.
	m_viewMatrix = XMMatrixLookAtLH(positionVector, lookAtVector, upVector);
	//XMVECTOR upVector = XMLoadFloat3(&up);

}


void LightClass::GenerateProjectionMatrix(float screenDepth, float screenNear)
{
	// 정사각형 광원에 대한 시야 및 화면 비율을 설정합니다.
	float fieldOfView = (float)XM_PI / 2.0f;
	float screenAspect = 1.0f;

	// 빛의 투영 행렬을 만듭니다.
	m_projectionMatrix = XMMatrixPerspectiveFovLH(fieldOfView, screenAspect, screenNear, screenDepth);
}

```

<br>

GraphicsClass에서 해당 객체를 생성하고 렌더링한다.

```c++

GraphicsClass.cpp

...

bool GraphicsClass::Initialize(int screenWidth, int screenHeight, HWND hwnd)
{
	// Direct3D 객체 생성
	m_D3D = new D3DClass;
	if (!m_D3D)
	{
		return false;
	}

	// Direct3D 객체 초기화
	bool result = m_D3D->Initialize(screenWidth, screenHeight, VSYNC_ENABLED, hwnd, FULL_SCREEN, SCREEN_DEPTH, SCREEN_NEAR);
	if (!result)
	{
		MessageBox(hwnd, L"Could not initialize Direct3D.", L"Error", MB_OK);
		return false;
	}

	// m_Camera 객체 생성
	m_Camera = new CameraClass;
	if (!m_Camera)
	{
		return false;
	}

	// 카메라 포지션을 설정한다
	m_Camera->SetPosition(XMFLOAT3(0.0f, 0.0f, -10.0f));

	// 큐브 모델 오브젝트를 생성합니다.
	m_CubeModel = new ModelClass;
	if(!m_CubeModel)
	{
		return false;
	}

	// 큐브 모델 오브젝트를 초기화 합니다.
	result = m_CubeModel->Initialize(m_D3D->GetDevice(), "../Dx11Demo_48/data/cube.txt", L"../Dx11Demo_48/data/wall01.dds");
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the cube model object.", L"Error", MB_OK);
		return false;
	}

	// 큐브 모델의 위치를 ​​설정 합니다.
	m_CubeModel->SetPosition(-2.0f, 2.0f, 0.0f);

	// 구형 모델 객체를 만듭니다.
	m_SphereModel = new ModelClass;
	if(!m_SphereModel)
	{
		return false;
	}

	// 구형 모델 객체를 초기화합니다.
	result = m_SphereModel->Initialize(m_D3D->GetDevice(), "../Dx11Demo_48/data/sphere.txt", L"../Dx11Demo_48/data/ice.dds");
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the sphere model object.", L"Error", MB_OK);
		return false;
	}

	// 구형 모델의 위치를 ​​설정합니다.
	m_SphereModel->SetPosition(2.0f, 2.0f, 0.0f);

	// 지면 모델 객체를 만듭니다.
	m_GroundModel = new ModelClass;
	if(!m_GroundModel)
	{
		return false;
	}

	// 지면 모델 객체를 초기화합니다.
	result = m_GroundModel->Initialize(m_D3D->GetDevice(), "../Dx11Demo_48/data/plane01.txt", L"../Dx11Demo_48/data/metal001.dds");
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the ground model object.", L"Error", MB_OK);
		return false;
	}

	// 지면 모델의 위치를 ​​설정합니다.
	m_GroundModel->SetPosition(0.0f, 1.0f, 0.0f);

	// light 객체를 만듭니다.
	m_Light = new LightClass;
	if(!m_Light)
	{
		return false;
	}

	// 조명 객체를 초기화합니다.
	m_Light->SetAmbientColor(0.15f, 0.15f, 0.15f, 1.0f);
	m_Light->SetDiffuseColor(1.0f, 1.0f, 1.0f, 1.0f);
	m_Light->GenerateOrthoMatrix(20.0f, SHADOWMAP_DEPTH, SHADOWMAP_NEAR);

	// 렌더링을 텍스처 오브젝트에 생성한다.
	m_RenderTexture = new RenderTextureClass;
	if(!m_RenderTexture)
	{
		return false;
	}

	// 렌더링을 텍스처 오브젝트를 초기화한다.
	result = m_RenderTexture->Initialize(m_D3D->GetDevice(), SHADOWMAP_WIDTH, SHADOWMAP_HEIGHT, SHADOWMAP_DEPTH, SHADOWMAP_NEAR);
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the render to texture object.", L"Error", MB_OK);
		return false;
	}

	// 깊이 셰이더 개체를 만듭니다.
	m_DepthShader = new DepthShaderClass;
	if(!m_DepthShader)
	{
		return false;
	}

	// 깊이 셰이더 개체를 초기화합니다.
	result = m_DepthShader->Initialize(m_D3D->GetDevice(), hwnd);
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the depth shader object.", L"Error", MB_OK);
		return false;
	}

	// 그림자 셰이더 개체를 만듭니다.
	m_ShadowShader = new ShadowShaderClass;
	if(!m_ShadowShader)
	{
		return false;
	}

	// 그림자 쉐이더 객체를 초기화합니다.
	result = m_ShadowShader->Initialize(m_D3D->GetDevice(), hwnd);
	if(!result)
	{
		MessageBox(hwnd, L"Could not initialize the shadow shader object.", L"Error", MB_OK);
		return false;
	}

	return true;
}

...

bool GraphicsClass::Frame(float posX, float posY, float posZ, float rotX, float rotY, float rotZ, float frameTime)
{
	static float lightAngle = 270.0f;
	static float lightPosX = 9.0f;

	// 카메라 위치를 설정합니다.
	m_Camera->SetPosition(XMFLOAT3(posX, posY, posZ));
	m_Camera->SetRotation(XMFLOAT3(rotX, rotY, rotZ));

	// 각 프레임의 조명 위치를 업데이트 합니다.
	lightPosX -= 0.003f * frameTime;

	// 각 프레임의 조명 각도를 업데이트 합니다.
	lightAngle -= 0.03f * frameTime;
	if(lightAngle < 90.0f)
	{
		lightAngle = 270.0f;

		// 조명 위치도 재설정 합니다.
		lightPosX = 9.0f;
	}
	float radians = lightAngle * 0.0174532925f;

	// 빛의 방향을 업데이 트합니다.
	m_Light->SetDirection(sinf(radians), cosf(radians), 0.0f);

	// 빛의 위치와 표시를 설정합니다.
	m_Light->SetPosition(lightPosX, 8.0f, -0.1f);
	m_Light->SetLookAt(-lightPosX, 0.0f, 0.0f);
	
	// 그래픽 장면을 렌더링합니다.
	return Render();
}


bool GraphicsClass::RenderSceneToTexture()
{
	XMMATRIX worldMatrix, lightViewMatrix, lightOrthoMatrix;

	float posX = 0;
	float posY = 0;
	float posZ = 0;

	// 렌더링 대상을 렌더링에 맞게 설정합니다.
	m_RenderTexture->SetRenderTarget(m_D3D->GetDeviceContext());

	// 렌더링을 텍스처에 지웁니다.
	m_RenderTexture->ClearRenderTarget(m_D3D->GetDeviceContext(), 0.0f, 0.0f, 0.0f, 1.0f);

	// 조명의 위치에 따라 조명보기 행렬을 생성합니다.
	m_Light->GenerateViewMatrix();

	// d3d 객체에서 세계 행렬을 가져옵니다.
	m_D3D->GetWorldMatrix(worldMatrix);

	// 라이트 오브젝트에서 뷰 및 정사각형 매트릭스를 가져옵니다.
	m_Light->GetViewMatrix(lightViewMatrix);
	m_Light->GetOrthoMatrix(lightOrthoMatrix);

	// 큐브 모델에 대한 변환 행렬을 설정하십시오.
	m_CubeModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);

	// 깊이 셰이더로 큐브 모델을 렌더링합니다.
	m_CubeModel->Render(m_D3D->GetDeviceContext());
	bool result = m_DepthShader->Render(m_D3D->GetDeviceContext(), m_CubeModel->GetIndexCount(), worldMatrix, lightViewMatrix, lightOrthoMatrix);
	if(!result)
	{
		return false;
	}

	// 월드 행렬을 재설정합니다.
	m_D3D->GetWorldMatrix(worldMatrix);

	// 구형 모델에 대한 변환 행렬을 설정합니다.
	m_SphereModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);

	// 깊이 셰이더로 구형 모델을 렌더링합니다.
	m_SphereModel->Render(m_D3D->GetDeviceContext());
	result = m_DepthShader->Render(m_D3D->GetDeviceContext(), m_SphereModel->GetIndexCount(), worldMatrix, lightViewMatrix, lightOrthoMatrix);
	if(!result)
	{
		return false;
	}

	// 월드 행렬을 재설정합니다.
	m_D3D->GetWorldMatrix(worldMatrix);

	// ground 모델에 대한 변환 행렬을 설정합니다.
	m_GroundModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);

	// 깊이 셰이더로 그라운드 모델을 렌더링합니다.
	m_GroundModel->Render(m_D3D->GetDeviceContext());
	result = m_DepthShader->Render(m_D3D->GetDeviceContext(), m_GroundModel->GetIndexCount(), worldMatrix, lightViewMatrix, lightOrthoMatrix);
	if(!result)
	{
		return false;
	}

	// 렌더링 대상을 원래의 백 버퍼로 다시 설정하고 렌더링에 대한 렌더링을 더 이상 다시 설정하지 않습니다.
	m_D3D->SetBackBufferRenderTarget();

	// 뷰포트를 원본으로 다시 설정합니다.
	m_D3D->ResetViewport();

	return true;
}


bool GraphicsClass::Render()
{
	XMMATRIX worldMatrix, viewMatrix, projectionMatrix;
	XMMATRIX lightViewMatrix, lightOrthoMatrix;

	float posX = 0;
	float posY = 0;
	float posZ = 0;

	// 먼저 장면을 텍스처로 렌더링합니다.
	bool result = RenderSceneToTexture();
	if(!result)
	{
		return false;
	}

	// 장면을 시작할 버퍼를 지운다.
	m_D3D->BeginScene(0.0f, 0.0f, 0.0f, 1.0f);

	// 카메라의 위치에 따라 뷰 행렬을 생성합니다.
	m_Camera->Render();

	// 조명의 위치에 따라 조명보기 행렬을 생성합니다.
	m_Light->GenerateViewMatrix();

	// 카메라 및 d3d 객체에서 월드, 뷰 및 투영 행렬을 가져옵니다.
	m_Camera->GetViewMatrix(viewMatrix);
	m_D3D->GetWorldMatrix(worldMatrix);
	m_D3D->GetProjectionMatrix(projectionMatrix);

	// 라이트 오브젝트로부터 라이트의 뷰와 투영 행렬을 가져옵니다.
	m_Light->GetViewMatrix(lightViewMatrix);
	m_Light->GetOrthoMatrix(lightOrthoMatrix);

	// 큐브 모델에 대한 변환 행렬을 설정하십시오.
	m_CubeModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);
	
	// 큐브 모델 정점과 인덱스 버퍼를 그래픽 파이프 라인에 배치하여 그리기를 준비합니다.
	m_CubeModel->Render(m_D3D->GetDeviceContext());

	// 그림자 쉐이더를 사용하여 모델을 렌더링합니다.
	result = m_ShadowShader->Render(m_D3D->GetDeviceContext(), m_CubeModel->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix, lightViewMatrix, 
lightOrthoMatrix, m_CubeModel->GetTexture(), m_RenderTexture->GetShaderResourceView(), m_Light->GetDirection(),m_Light->GetAmbientColor(), m_Light->GetDiffuseColor());
	if(!result)
	{
		return false;
	}

	// 월드 행렬을 재설정합니다.
	m_D3D->GetWorldMatrix(worldMatrix);

	// 구형 모델에 대한 변환 행렬을 설정합니다.
	m_SphereModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);

	// 모델 버텍스와 인덱스 버퍼를 그래픽 파이프 라인에 배치하여 드로잉을 준비합니다.
	m_SphereModel->Render(m_D3D->GetDeviceContext());
	result = m_ShadowShader->Render(m_D3D->GetDeviceContext(), m_SphereModel->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix, lightViewMatrix, 
lightOrthoMatrix, m_SphereModel->GetTexture(), m_RenderTextureGetShaderResourceView(), m_Light->GetDirection(), m_Light->GetAmbientColor(), m_Light->GetDiffuseColor());
	if(!result)
	{
		return false;
	}

	// 월드 행렬을 재설정합니다.
	m_D3D->GetWorldMatrix(worldMatrix);

	// ground 모델에 대한 변환 행렬을 설정합니다.
	m_GroundModel->GetPosition(posX, posY, posZ);
	worldMatrix = XMMatrixTranslation(posX, posY, posZ);

	// 그림자 쉐이더를 사용하여 그라운드 모델을 렌더링합니다.
	m_GroundModel->Render(m_D3D->GetDeviceContext());
	result = m_ShadowShader->Render(m_D3D->GetDeviceContext(), m_GroundModel->GetIndexCount(), worldMatrix, viewMatrix, projectionMatrix, lightViewMatrix, lightOrthoMatrix, m_GroundModel->GetTexture(), m_RenderTexture->GetShaderResourceView(), m_Light->GetDirection(), m_Light->GetAmbientColor(), m_Light->GetDiffuseColor());
	if(!result)
	{
		return false;
	}

	// 렌더링 된 장면을 화면에 표시합니다.
	m_D3D->EndScene();

	return true;
}

```

<br>
<br>


# 출력
---

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/af4e9d2a-c967-42e3-95da-a4e8b5bbeca7" alt width=600>
<em>광원 이동</em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d39e395c-3c6d-4127-a092-3b5618f27781" alt width=600>
<em>출력 이미지</em>
</center>


<br>
<br>
<br>


출처: www.rastertek.com/tutdx11win10.html

