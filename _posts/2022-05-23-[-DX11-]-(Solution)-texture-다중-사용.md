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
---
# 문제
___

현재 framework는 하나의 모델에 하나의 texture를 적용, 렌더링한다. 그러나 대부분의 3D 모델은 여러 texture로 이루어져 있어, 여러 texture 모델을 불러오기 위해 코드를 변경하였다.



```c++

void LoadModels(ID3D11Device* pd3dDevice)
{
    g_pModel_1 = new Animation();
    g_pModel_1->LoadMesh("./models/Sphere/sphere.x", pd3dDevice);
}

HRESULT LoadTextures(ID3D11Device* pd3dDevice, ID3D11DeviceContext* pd3dImmediateContext)
{
    HRESULT hr = S_OK;
    V_RETURN(DXUTCreateShaderResourceViewFromFile(pd3dDevice, L"./models/Sphere/billiard13.jpg", &g_pTextures_1));
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
}

void SetEffect(XMMATRIX WorldViewProjection, ID3D11ShaderResourceView* Textures, XMMATRIX previousWorldViewProjection)
{
	XMFLOAT4X4 tmp4x4;
	XMStoreFloat4x4(&tmp4x4, WorldViewProjection);
	g_WVP->SetMatrix(reinterpret_cast<float*>(&tmp4x4));

	XMStoreFloat4x4(&tmp4x4, previousWorldViewProjection);
	g_PreWVP->SetMatrix(reinterpret_cast<float*>(&tmp4x4));

	g_Texture->SetResource(Textures);
}

void RenderObject(Animation* Model, ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* Layout, ID3DX11EffectTechnique* pTech)
{
	constexpr UINT stride = sizeof(MyVertex);
	constexpr UINT offset = NULL;
	ID3D11Buffer* pVertexBuffer = Model->GetVB();
	ID3D11Buffer* pIndexBuffer = Model->GetIB();
	ID3D11Buffer* pBuffer[1] = { pVertexBuffer };
	pd3dImmediateContext->IASetVertexBuffers(0, 1, pBuffer, &stride, &offset);
	pd3dImmediateContext->IASetIndexBuffer(pIndexBuffer, DXGI_FORMAT_R32_UINT, 0);
	pd3dImmediateContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
	pd3dImmediateContext->IASetInputLayout(Layout);

	pTech->GetDesc(&techDesc);

	for (UINT p = 0; p < techDesc.Passes; p++)
	{
		pTech->GetPassByIndex(0)->Apply(0, pd3dImmediateContext);

		vector<MeshEntry> Mesh = Model->GetMesh();
		const UINT MeshCount = Mesh.size();
		for (UINT index = 0; index < MeshCount; index++)
		{
			pd3dImmediateContext->DrawIndexed(Mesh[index].NumIndices, Mesh[index].BaseIndex, Mesh[index].BaseVertex);
		}
	}
}



```



# 해결
___


모델을 불러올 때, 각 텍스처 정보를 읽어 저장, 이를 모델에 적용하고 셰이더에 정보를 전달한다.


```c++


bool Model::MakeLoadModel(const string& Filename, ID3D11Device* pDevice, ID3D11DeviceContext* deviceContext)
{
    this->device = pDevice;
    this->deviceContext = deviceContext;

    this->directory = StringHelper::GetDirectoryFromPath(Filename);

    Assimp::Importer importer;

    const aiScene* pScene = importer.ReadFile(Filename,
        aiProcess_Triangulate |
        aiProcess_ConvertToLeftHanded);

    if (pScene == nullptr)
        return false;


    this->ProcessNode(pScene->mRootNode, pScene);
    return true;
}


void Model::ProcessNode(aiNode* node, const aiScene* scene)
{
    for (UINT i = 0; i < node->mNumMeshes; i++)
    {
        aiMesh* mesh = scene->mMeshes[node->mMeshes[i]];
        aiVector3D aiVertex = node->mTransformation * scene->mMeshes[node->mMeshes[i]]->mVertices[i];
        scene->mMeshes[node->mMeshes[i]]->mVertices[i] = aiVertex;

        meshes.push_back(this->ProcessMesh(mesh, scene));
    }

    for (UINT i = 0; i < node->mNumChildren; i++)
    {
        this->ProcessNode(node->mChildren[i], scene);
    }
}

Mesh Model::ProcessMesh(aiMesh* mesh, const aiScene* scene)
{
    // Data to fill
    std::vector<MakeVertex> vertices;
    std::vector<DWORD> indices;

    //Get vertices
    for (UINT i = 0; i < mesh->mNumVertices; i++)
    {
        MakeVertex vertex;
        vertex.pos.x = mesh->mVertices[i].x;
        vertex.pos.y = mesh->mVertices[i].y;
        vertex.pos.z = mesh->mVertices[i].z;

        vertex.normal.x = mesh->mNormals[i].x;
        vertex.normal.y = mesh->mNormals[i].y;
        vertex.normal.z = mesh->mNormals[i].z;

        if (mesh->mTextureCoords[0])
        {
            vertex.texCoord.x = (float)mesh->mTextureCoords[0][i].x;
            vertex.texCoord.y = (float)mesh->mTextureCoords[0][i].y;
        }

        vertices.push_back(vertex);

    }

    //Get indices
    for (UINT i = 0; i < mesh->mNumFaces; i++)
    {
        aiFace face = mesh->mFaces[i];

        for (UINT j = 0; j < face.mNumIndices; j++)
            indices.push_back(face.mIndices[j]);
    }

    std::vector<Texture> textures;
    aiMaterial* material = scene->mMaterials[mesh->mMaterialIndex];
    std::vector<Texture> diffuseTextures = LoadMaterialTextures(material, aiTextureType::aiTextureType_DIFFUSE, scene);
    textures.insert(textures.end(), diffuseTextures.begin(), diffuseTextures.end());

    return Mesh(this->device, this->deviceContext, vertices, indices, textures);
}

TextureStorageType Model::DetermineTextureStorageType(const aiScene* pScene, aiMaterial* pMat, unsigned int index, aiTextureType textureType)
{
    if (pMat->GetTextureCount(textureType) == 0)
        return TextureStorageType::None;

    aiString path;
    pMat->GetTexture(textureType, index, &path);
    std::string texturePath = path.C_Str();

    //Check if texture is an embedded indexed texture by seeing if the file path is an index #
    if (texturePath[0] == '*')
    {
        if (pScene->mTextures[0]->mHeight == 0)
        {
            return TextureStorageType::EmbeddedIndexCompressed;
        }
        else
        {
            assert("SUPPORT DOES NOT EXIST YET FOR INDEXED NON COMPRESSED TEXTURES!" && 0);
            return TextureStorageType::EmbeddedIndexNonCompressed;
        }
    }
    //Check if texture is an embedded texture but not indexed (path will be the texture's name instead of #)

    if (auto pTex = pScene->GetEmbeddedTexture(texturePath.c_str()))
    {
        if (pTex->mHeight == 0)
        {
            return TextureStorageType::EmbeddedCompressed;
        }
        else
        {
            assert("SUPPORT DOES NOT EXIST YET FOR EMBEDDED NON COMPRESSED TEXTURES!" && 0);
            return TextureStorageType::EmbeddedNonCompressed;
        }
    }

    //Lastly check if texture is a filepath by checking for period before extension name
    if (texturePath.find('.') != std::string::npos)
    {
        return TextureStorageType::Disk;
    }

    return TextureStorageType::None; // No texture exists
}

std::vector<Texture> Model::LoadMaterialTextures(aiMaterial* pMaterial, aiTextureType textureType, const aiScene* pScene)
{
    std::vector<Texture> materialTextures;
    TextureStorageType storetype = TextureStorageType::Invalid;
    unsigned int textureCount = pMaterial->GetTextureCount(textureType);

    if (textureCount == 0) //If there are no textures
    {
        storetype = TextureStorageType::None;
        aiColor3D aiColor(0.0f, 0.0f, 0.0f);
        switch (textureType)
        {
        case aiTextureType_DIFFUSE:
            pMaterial->Get(AI_MATKEY_COLOR_DIFFUSE, aiColor);
            if (aiColor.IsBlack()) //If color = black, just use grey
            {
                materialTextures.push_back(Texture(this->device, N_Color::UnloadedTextureColor, textureType));
                return materialTextures;
            }
            materialTextures.push_back(Texture(this->device, Color(aiColor.r * 255, aiColor.g * 255, aiColor.b * 255), textureType));
            return materialTextures;
        }
    }
    else
    {
        for (UINT i = 0; i < textureCount; i++)
        {
            aiString path;
            pMaterial->GetTexture(textureType, i, &path);

            //std::string texturePath = path.C_Str();
            //std::string filename = this->directory;

            string tempDirectory = this->directory;
            string tempTextureName = path.C_Str();

            tempDirectory.erase(0, 1);

            for (int i = 0; i < tempTextureName.size(); i++)
            {
                if (tempTextureName[i] == '\\')
                {
                    tempTextureName.erase(tempTextureName.begin() + i);
                    tempTextureName.insert(i, "/");
                }
            }

            if (tempTextureName.find(tempDirectory) != string::npos)
            {
                int index = tempTextureName.find(tempDirectory);
                tempTextureName.erase(tempTextureName.begin(), tempTextureName.begin() + index + tempDirectory.size());
            }


            TextureStorageType storetype = DetermineTextureStorageType(pScene, pMaterial, i, textureType);
            switch (storetype)
            {
            case TextureStorageType::Disk:
            {
                std::string textureName = this->directory + '/' + tempTextureName;

                //texturePath.find(filename);
                //std::string filename = this->directory + '\\' + path.C_Str();
                //Texture diskTexture(this->device, filename, textureType);
                Texture diskTexture(this->device, this->deviceContext, textureName, textureType);
                
                materialTextures.push_back(diskTexture);
                break;
            }
            }
        }
    }

    if (materialTextures.size() == 0)
    {
        materialTextures.push_back(Texture(this->device, N_Color::UnhandledTextureColor, aiTextureType::aiTextureType_DIFFUSE));
    }
    return materialTextures;

}

void Model::Draw(ID3DX11EffectShaderResourceVariable* m_effect_texture, ID3DX11EffectTechnique* g_pTech, D3DX11_TECHNIQUE_DESC techDesc , ID3D11DeviceContext* pd3dImmediateContext, ID3D11InputLayout* g_pLayout)
{
    for (int i = 0; i < meshes.size(); i++)
    {
        meshes[i].Draw(m_effect_texture, g_pTech, techDesc, pd3dImmediateContext, g_pLayout);
    }
}


```



여러 텍스처로 이루어진 Sponza 맵이 올바르게 렌더링된 것을 확인할 수 있었다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/e0ee63c6-4423-41f3-86ea-9b3830887454" alt width="100%">
<em></em>
</center>











