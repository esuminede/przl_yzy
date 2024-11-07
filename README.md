
//PÜRÜZLÜ YÜZEY ÜRETİMİ
//CPP KODU 
//--------------------------------------------------------------------------------------
//	DIRECTX 11 BUMP MAPPING ve PARALLAX MAPPING UYGULAMASI (2024-2025 GUZ DONEMI)
//
//  Kodlar DirectX 11'de yazıldığı için DirectX June 2010 SDK kurulmalıdır :
//  https://www.microsoft.com/en-us/download/details.aspx?id=6812
//--------------------------------------------------------------------------------------

#include <windows.h>
#include <d3d11.h>
#include <d3dx11.h>
#include <d3dcompiler.h>
#include <xnamath.h>
#include "resource.h"

//--------------------------------------------------------------------------------------
// Structures
//--------------------------------------------------------------------------------------
struct SimpleVertex
{
    XMFLOAT3 Pos;  
    XMFLOAT2 Tex; 
	XMFLOAT3 Nrm;  
	XMFLOAT3 Tan;  
	XMFLOAT3 Bin;  
};

struct ConstantBuffer
{
    XMMATRIX mWorld;
	XMMATRIX WorldViewProjection;
	XMFLOAT4 LightPosition;
	XMFLOAT4 EyePosition;
	float    fHeightMapScale;
	int      nMaxSamples;
    //KOD EKLENDI
    int key;
}cb;

//--------------------------------------------------------------------------------------
// Global Variables
//--------------------------------------------------------------------------------------
HINSTANCE                           g_hInst = NULL;
HWND                                g_hWnd = NULL;
D3D_DRIVER_TYPE                     g_driverType = D3D_DRIVER_TYPE_NULL;
D3D_FEATURE_LEVEL                   g_featureLevel = D3D_FEATURE_LEVEL_11_0;
ID3D11Device*                       g_pd3dDevice = NULL;
ID3D11DeviceContext*                g_pImmediateContext = NULL;
IDXGISwapChain*                     g_pSwapChain = NULL;
ID3D11RenderTargetView*             g_pRenderTargetView = NULL;
ID3D11Texture2D*                    g_pDepthStencil = NULL;
ID3D11DepthStencilView*             g_pDepthStencilView = NULL;
ID3D11VertexShader*                 g_pVertexShader = NULL;
ID3D11PixelShader*                  g_pPixelShader = NULL;
ID3D11InputLayout*                  g_pVertexLayout = NULL;
ID3D11Buffer*                       g_pVertexBuffer = NULL;
ID3D11Buffer*                       g_pIndexBuffer = NULL;
ID3D11Buffer*                       g_pConstantBuffer = NULL; 
ID3D11ShaderResourceView*           g_pTextureRV = NULL;
ID3D11ShaderResourceView*           g_pHeightTexRV = NULL;
ID3D11ShaderResourceView*           g_pColorTexRV = NULL;
ID3D11SamplerState*                 g_pSamplerLinear = NULL;
XMMATRIX                            g_World;
XMMATRIX                            g_View;
XMMATRIX                            g_Projection;
XMVECTOR							Eye;

//--------------------------------------------------------------------------------------
// Forward declarations
//--------------------------------------------------------------------------------------
HRESULT InitWindow( HINSTANCE hInstance, int nCmdShow );
HRESULT InitDevice();
void CleanupDevice();
LRESULT CALLBACK    WndProc( HWND, UINT, WPARAM, LPARAM );
void Render();

//--------------------------------------------------------------------------------------
// Entry point to the program. Initializes everything and goes into a message processing 
// loop. Idle time is used to render the scene.
//--------------------------------------------------------------------------------------
int WINAPI wWinMain( HINSTANCE hInstance, HINSTANCE hPrevInstance, LPWSTR lpCmdLine, int nCmdShow )
{
    UNREFERENCED_PARAMETER( hPrevInstance );
    UNREFERENCED_PARAMETER( lpCmdLine );

    if( FAILED( InitWindow( hInstance, nCmdShow ) ) )
        return 0;

    if( FAILED( InitDevice() ) )
    {
        CleanupDevice();
        return 0;
    }

    // Main message loop
    MSG msg = {0};
    while( WM_QUIT != msg.message )
    {
        if( PeekMessage( &msg, NULL, 0, 0, PM_REMOVE ) )
        {
            TranslateMessage( &msg );
            DispatchMessage( &msg );
        }
        else
        {
            Render();
        }
    }

    CleanupDevice();

    return ( int )msg.wParam;
}


//--------------------------------------------------------------------------------------
// InitWindow
//--------------------------------------------------------------------------------------
HRESULT InitWindow( HINSTANCE hInstance, int nCmdShow )
{
    // Register class
    WNDCLASSEX wcex;
    wcex.cbSize = sizeof( WNDCLASSEX );
    wcex.style = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc = WndProc;
    wcex.cbClsExtra = 0;
    wcex.cbWndExtra = 0;
    wcex.hInstance = hInstance;
    wcex.hIcon = LoadIcon( hInstance, ( LPCTSTR )IDI_TUTORIAL1 );
    wcex.hCursor = LoadCursor( NULL, IDC_ARROW );
    wcex.hbrBackground = ( HBRUSH )( COLOR_WINDOW + 1 );
    wcex.lpszMenuName = NULL;
    wcex.lpszClassName = L"ParallaxWindowClass";
    wcex.hIconSm = LoadIcon( wcex.hInstance, ( LPCTSTR )IDI_TUTORIAL1 );
    if( !RegisterClassEx( &wcex ) ) return E_FAIL;

    g_hInst = hInstance;
    RECT rc = { 0, 0, 1280, 720 };
    AdjustWindowRect( &rc, WS_OVERLAPPEDWINDOW, FALSE );
	g_hWnd = CreateWindow(L"ParallaxWindowClass", L"DX11 BUMP/PARALLAX MAPPING UYGULAMASI (2020-21 GUZ)    'Q','W','E' : DOKU DEGISTIRME    0' : Bump yok Parallax yok,  '1' : Bump yok Parallax var,  '2' : Bump var Parallax yok,  '3' : Bump var Parallax var", WS_OVERLAPPEDWINDOW,
                           CW_USEDEFAULT, CW_USEDEFAULT, rc.right - rc.left, rc.bottom - rc.top, NULL, NULL, hInstance,
                           NULL );
    if( !g_hWnd ) return E_FAIL;

    ShowWindow( g_hWnd, nCmdShow );

    return S_OK;
}

//--------------------------------------------------------------------------------------
// Clean up the objects
//--------------------------------------------------------------------------------------
void CleanupDevice()
{
    if( g_pImmediateContext )		g_pImmediateContext->ClearState();
    if( g_pSamplerLinear )			g_pSamplerLinear->Release();
    if( g_pTextureRV )				g_pTextureRV->Release();
    if( g_pConstantBuffer )	g_pConstantBuffer->Release();
    if( g_pVertexBuffer )			g_pVertexBuffer->Release();
    if( g_pIndexBuffer )			g_pIndexBuffer->Release();
    if( g_pVertexLayout )			g_pVertexLayout->Release();
    if( g_pVertexShader )			g_pVertexShader->Release();
    if( g_pPixelShader )			g_pPixelShader->Release();
    if( g_pDepthStencil )			g_pDepthStencil->Release();
    if( g_pDepthStencilView )		g_pDepthStencilView->Release();
    if( g_pRenderTargetView )		g_pRenderTargetView->Release();
    if( g_pSwapChain )				g_pSwapChain->Release();
    if( g_pImmediateContext )		g_pImmediateContext->Release();
    if( g_pd3dDevice )				g_pd3dDevice->Release();
}

//--------------------------------------------------------------------------------------
// Helper for compiling shaders with D3DX11
//--------------------------------------------------------------------------------------
HRESULT CompileShaderFromFile( WCHAR* szFileName, LPCSTR szEntryPoint, LPCSTR szShaderModel, ID3DBlob** ppBlobOut )
{
    HRESULT hr = S_OK;

    DWORD dwShaderFlags = D3DCOMPILE_ENABLE_STRICTNESS;
#if defined( DEBUG ) || defined( _DEBUG )
    // Set the D3DCOMPILE_DEBUG flag to embed debug information in the shaders.
    // Setting this flag improves the shader debugging experience, but still allows 
    // the shaders to be optimized and to run exactly the way they will run in 
    // the release configuration of this program.
    dwShaderFlags |= D3DCOMPILE_DEBUG;
#endif

    ID3DBlob* pErrorBlob;
    hr = D3DX11CompileFromFile( szFileName, NULL, NULL, szEntryPoint, szShaderModel, 
        dwShaderFlags, 0, NULL, ppBlobOut, &pErrorBlob, NULL );
    if( FAILED(hr) )
    {
        if( pErrorBlob != NULL )
            OutputDebugStringA( (char*)pErrorBlob->GetBufferPointer() );
        if( pErrorBlob ) pErrorBlob->Release();
        return hr;
    }
    if( pErrorBlob ) pErrorBlob->Release();

    return S_OK;
}

//--------------------------------------------------------------------------------------
// Called every time the application receives a message
//--------------------------------------------------------------------------------------
LRESULT CALLBACK WndProc( HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam )
{
    PAINTSTRUCT ps;
    HDC hdc;

    switch( message )
    {
        case WM_PAINT:
            hdc = BeginPaint( hWnd, &ps );
            EndPaint( hWnd, &ps );
            break;

        case WM_DESTROY:
            PostQuitMessage( 0 );
            break;

		case WM_KEYDOWN:
		{
			switch( wParam )
			{
				case VK_UP:
					cb.fHeightMapScale = cb.fHeightMapScale - 0.01f;
					if ( cb.fHeightMapScale < 0.0f ) cb.fHeightMapScale = 0.0f;
					break;
				
				case VK_DOWN:
					cb.fHeightMapScale = cb.fHeightMapScale + 0.01f;
					if ( cb.fHeightMapScale > 0.4f ) cb.fHeightMapScale = 0.4f;
					break;
				
				case VK_LEFT:
					cb.nMaxSamples--;
					if ( cb.nMaxSamples < 4 ) cb.nMaxSamples = 4;
					break;
				
				case VK_RIGHT:
					cb.nMaxSamples++;
					if ( cb.nMaxSamples > 40 ) cb.nMaxSamples = 40;
					break;

				case 'Q':
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Wood\\wood_normalmap.dds", NULL, NULL, &g_pHeightTexRV, NULL );
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Wood\\wood_colormap.dds", NULL, NULL, &g_pColorTexRV, NULL );
					break;

				case 'W':
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock1\\rock_normalmap.dds", NULL, NULL, &g_pHeightTexRV, NULL );
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock1\\rock_colormap.dds", NULL, NULL, &g_pColorTexRV, NULL );
					break;

				case 'E':
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock2\\rock_normalmap.dds", NULL, NULL, &g_pHeightTexRV, NULL );
					D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock2\\rock_colormap.dds", NULL, NULL, &g_pColorTexRV, NULL );
					break;
                    //KOD EKLENDI
                case '0':
                    cb.key=0;
                    break;
                case '1':
                    cb.key=1;
                    break;
                case '2':
                    cb.key=2;
                    break;
                case '3':
                    cb.key=3;
                    break;
                    
			}
		}

        default:
            return DefWindowProc( hWnd, message, wParam, lParam );
    }

    return 0;
}

//--------------------------------------------------------------------------------------
// Device and swap chain
//--------------------------------------------------------------------------------------
HRESULT InitDevice()
{
    HRESULT hr = S_OK;

    RECT rc;
    GetClientRect( g_hWnd, &rc );
    UINT width = rc.right - rc.left;
    UINT height = rc.bottom - rc.top;

    UINT createDeviceFlags = 0;
#ifdef _DEBUG
    createDeviceFlags |= D3D11_CREATE_DEVICE_DEBUG;
#endif

    D3D_DRIVER_TYPE driverTypes[] =
    {
        D3D_DRIVER_TYPE_HARDWARE,
        D3D_DRIVER_TYPE_WARP,
        D3D_DRIVER_TYPE_REFERENCE,
    };
    UINT numDriverTypes = ARRAYSIZE( driverTypes );

    D3D_FEATURE_LEVEL featureLevels[] =
    {
        D3D_FEATURE_LEVEL_11_0,
        D3D_FEATURE_LEVEL_10_1,
        D3D_FEATURE_LEVEL_10_0,
    };
    UINT numFeatureLevels = ARRAYSIZE( featureLevels );

    DXGI_SWAP_CHAIN_DESC sd;
    ZeroMemory( &sd, sizeof( sd ) );
    sd.BufferCount = 1;
    sd.BufferDesc.Width = width;
    sd.BufferDesc.Height = height;
    sd.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.BufferDesc.RefreshRate.Numerator = 60;
    sd.BufferDesc.RefreshRate.Denominator = 1;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.OutputWindow = g_hWnd;
    sd.SampleDesc.Count = 1;
    sd.SampleDesc.Quality = 0;
    sd.Windowed = TRUE;

    for( UINT driverTypeIndex = 0; driverTypeIndex < numDriverTypes; driverTypeIndex++ )
    {
        g_driverType = driverTypes[driverTypeIndex];
        hr = D3D11CreateDeviceAndSwapChain( NULL, g_driverType, NULL, createDeviceFlags, featureLevels, numFeatureLevels,
                                            D3D11_SDK_VERSION, &sd, &g_pSwapChain, &g_pd3dDevice, &g_featureLevel, &g_pImmediateContext );
        if( SUCCEEDED( hr ) ) break;
    }
    if( FAILED( hr ) ) return hr;

    ID3D11Texture2D* pBackBuffer = NULL;
    hr = g_pSwapChain->GetBuffer( 0, __uuidof( ID3D11Texture2D ), ( LPVOID* )&pBackBuffer );
    if( FAILED( hr ) ) return hr;

    hr = g_pd3dDevice->CreateRenderTargetView( pBackBuffer, NULL, &g_pRenderTargetView );
    pBackBuffer->Release();
    if( FAILED( hr ) ) return hr;

    D3D11_TEXTURE2D_DESC descDepth;
    ZeroMemory( &descDepth, sizeof(descDepth) );
    descDepth.Width = width;
    descDepth.Height = height;
    descDepth.MipLevels = 1;
    descDepth.ArraySize = 1;
    descDepth.Format = DXGI_FORMAT_D24_UNORM_S8_UINT;
    descDepth.SampleDesc.Count = 1;
    descDepth.SampleDesc.Quality = 0;
    descDepth.Usage = D3D11_USAGE_DEFAULT;
    descDepth.BindFlags = D3D11_BIND_DEPTH_STENCIL;
    descDepth.CPUAccessFlags = 0;
    descDepth.MiscFlags = 0;
    hr = g_pd3dDevice->CreateTexture2D( &descDepth, NULL, &g_pDepthStencil );
    if( FAILED( hr ) ) return hr;

    D3D11_DEPTH_STENCIL_VIEW_DESC descDSV;
    ZeroMemory( &descDSV, sizeof(descDSV) );
    descDSV.Format = descDepth.Format;
    descDSV.ViewDimension = D3D11_DSV_DIMENSION_TEXTURE2D;
    descDSV.Texture2D.MipSlice = 0;
    hr = g_pd3dDevice->CreateDepthStencilView( g_pDepthStencil, &descDSV, &g_pDepthStencilView );
    if( FAILED( hr ) ) return hr;

    g_pImmediateContext->OMSetRenderTargets( 1, &g_pRenderTargetView, g_pDepthStencilView );

    // Setup the viewport
    D3D11_VIEWPORT vp;
    vp.Width = (FLOAT)width;
    vp.Height = (FLOAT)height;
    vp.MinDepth = 0.0f;
    vp.MaxDepth = 1.0f;
    vp.TopLeftX = 0;
    vp.TopLeftY = 0;
    g_pImmediateContext->RSSetViewports( 1, &vp );

    // Compile the vertex shader
    ID3DBlob* pVSBlob = NULL;
    hr = CompileShaderFromFile( L"ParallaxMapping11.fx", "VS", "vs_4_0", &pVSBlob );
    if( FAILED( hr ) )
    {
        MessageBox( NULL,
                    L"The FX file cannot be compiled.  Please run this executable from the directory that contains the FX file.", L"Error", MB_OK );
        return hr;
    }

    hr = g_pd3dDevice->CreateVertexShader( pVSBlob->GetBufferPointer(), pVSBlob->GetBufferSize(), NULL, &g_pVertexShader );
    if( FAILED( hr ) )
    {    
        pVSBlob->Release();
        return hr;
    }

    // Define the input layout
    D3D11_INPUT_ELEMENT_DESC layout[] =
    {
        { "POSITION",	0, DXGI_FORMAT_R32G32B32_FLOAT,	0,  0, D3D11_INPUT_PER_VERTEX_DATA, 0 },  
        { "TEXCOORD",	0, DXGI_FORMAT_R32G32_FLOAT,	0, 12, D3D11_INPUT_PER_VERTEX_DATA, 0 }, 
		{ "NORMAL",		0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 20, D3D11_INPUT_PER_VERTEX_DATA, 0 }, 
		{ "TANGENT",	0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 32, D3D11_INPUT_PER_VERTEX_DATA, 0 }, 
		{ "BINORMAL",	0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 44, D3D11_INPUT_PER_VERTEX_DATA, 0 }, 
    };
	UINT numElements = ARRAYSIZE( layout );

    hr = g_pd3dDevice->CreateInputLayout( layout, numElements, pVSBlob->GetBufferPointer(),
                                          pVSBlob->GetBufferSize(), &g_pVertexLayout );
    pVSBlob->Release();
    if( FAILED( hr ) ) return hr;

    // Set the input layout
    g_pImmediateContext->IASetInputLayout( g_pVertexLayout );

    // Compile the pixel shader
    ID3DBlob* pPSBlob = NULL;
    hr = CompileShaderFromFile( L"ParallaxMapping11.fx", "PS", "ps_4_0", &pPSBlob );
    if( FAILED( hr ) )
    {
        MessageBox( NULL,
                    L"The FX file cannot be compiled.  Please run this executable from the directory that contains the FX file.", L"Error", MB_OK );
        return hr;
    }

    hr = g_pd3dDevice->CreatePixelShader( pPSBlob->GetBufferPointer(), pPSBlob->GetBufferSize(), NULL, &g_pPixelShader );
    pPSBlob->Release();
    if( FAILED( hr ) ) return hr;

	SimpleVertex vertices[] =
	{
		{ XMFLOAT3(  1.0f,  0.0f,  1.0f ),	XMFLOAT2( 1.0f, 0.0f ), XMFLOAT3( 0.0f, 1.0f, 0.0f),	XMFLOAT3( 1.0f, 0.0f, 0.0f),	XMFLOAT3( 0.0f, 0.0f, -1.0f) },
		{ XMFLOAT3(  1.0f,  0.0f, -1.0f ),	XMFLOAT2( 1.0f, 1.0f ), XMFLOAT3( 0.0f, 1.0f, 0.0f),	XMFLOAT3( 1.0f, 0.0f, 0.0f),	XMFLOAT3( 0.0f, 0.0f, -1.0f) },
		{ XMFLOAT3( -1.0f,  0.0f, -1.0f ),	XMFLOAT2( 0.0f, 1.0f ), XMFLOAT3( 0.0f, 1.0f, 0.0f),	XMFLOAT3( 1.0f, 0.0f, 0.0f),	XMFLOAT3( 0.0f, 0.0f, -1.0f) },
		{ XMFLOAT3( -1.0f,  0.0f,  1.0f ),	XMFLOAT2( 0.0f, 0.0f ), XMFLOAT3( 0.0f, 1.0f, 0.0f),	XMFLOAT3( 1.0f, 0.0f, 0.0f),	XMFLOAT3( 0.0f, 0.0f, -1.0f) },
	};

    D3D11_BUFFER_DESC bd;
    ZeroMemory( &bd, sizeof(bd) );
    bd.Usage = D3D11_USAGE_DEFAULT;
    bd.ByteWidth = sizeof( SimpleVertex ) * 4;
    bd.BindFlags = D3D11_BIND_VERTEX_BUFFER;
    bd.CPUAccessFlags = 0;
    D3D11_SUBRESOURCE_DATA InitData;
    ZeroMemory( &InitData, sizeof(InitData) );
    InitData.pSysMem = vertices;
    hr = g_pd3dDevice->CreateBuffer( &bd, &InitData, &g_pVertexBuffer );
    if( FAILED( hr ) ) return hr;

    // Set vertex buffer
    UINT stride = sizeof( SimpleVertex );
    UINT offset = 0;
    g_pImmediateContext->IASetVertexBuffers( 0, 1, &g_pVertexBuffer, &stride, &offset );

    WORD indices[] =
    {
        0,1,2,
        0,2,3,
    };

    bd.Usage = D3D11_USAGE_DEFAULT;
    bd.ByteWidth = sizeof( WORD ) * 6;
    bd.BindFlags = D3D11_BIND_INDEX_BUFFER;
    bd.CPUAccessFlags = 0;
    InitData.pSysMem = indices;
    hr = g_pd3dDevice->CreateBuffer( &bd, &InitData, &g_pIndexBuffer );
    if( FAILED( hr ) ) return hr;

    // Set index buffer
    g_pImmediateContext->IASetIndexBuffer( g_pIndexBuffer, DXGI_FORMAT_R16_UINT, 0 );

    // Set primitive topology
    g_pImmediateContext->IASetPrimitiveTopology( D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST );

    bd.Usage = D3D11_USAGE_DEFAULT;
    bd.BindFlags = D3D11_BIND_CONSTANT_BUFFER;
    bd.CPUAccessFlags = 0;
    bd.ByteWidth = sizeof(ConstantBuffer);
    hr = g_pd3dDevice->CreateBuffer( &bd, NULL, &g_pConstantBuffer );
    if( FAILED( hr ) ) return hr;

	// Load the rock1 texture
	hr = D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock1\\rock_normalmap.dds", NULL, NULL, &g_pHeightTexRV, NULL );
	hr = D3DX11CreateShaderResourceViewFromFile( g_pd3dDevice, L"Textures\\Rock1\\rock_colormap.dds", NULL, NULL, &g_pColorTexRV, NULL );
	if( FAILED( hr ) ) return hr;

    D3D11_SAMPLER_DESC sampDesc;
    ZeroMemory( &sampDesc, sizeof(sampDesc) );
    sampDesc.Filter = D3D11_FILTER_MIN_MAG_MIP_LINEAR;
    sampDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
    sampDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
    sampDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
    sampDesc.ComparisonFunc = D3D11_COMPARISON_NEVER;
    sampDesc.MinLOD = 0;
    sampDesc.MaxLOD = D3D11_FLOAT32_MAX;
    hr = g_pd3dDevice->CreateSamplerState( &sampDesc, &g_pSamplerLinear );
    if( FAILED( hr ) ) return hr;

    // Initialize the world matrices
    g_World = XMMatrixIdentity();

    // Initialize the view matrix
    Eye = XMVectorSet( 0.0f, 2.0f, -2.0f, 0.0f );
    XMVECTOR At = XMVectorSet( 0.0f, 0.0f, 0.0f, 0.0f );
    XMVECTOR Up = XMVectorSet( 0.0f, 1.0f, 0.0f, 0.0f );
    g_View = XMMatrixLookAtLH( Eye, At, Up );

    // Initialize the projection matrix
    g_Projection = XMMatrixPerspectiveFovLH( XM_PIDIV4, width / (FLOAT)height, 0.01f, 100.0f );

	// Initialize the constant buffer
	cb.fHeightMapScale	= 0.1F;
	cb.nMaxSamples		= 20;

    return S_OK;
}

//--------------------------------------------------------------------------------------
// Render a frame
//--------------------------------------------------------------------------------------
void Render()
{
    // Update our time
    static float t = 0.0f;
    if( g_driverType == D3D_DRIVER_TYPE_REFERENCE )
    {
        t += ( float )XM_PI * 0.0125f;
    }
    else
    {
        static DWORD dwTimeStart = 0;
        DWORD dwTimeCur = GetTickCount();
        if( dwTimeStart == 0 ) dwTimeStart = dwTimeCur;
        t = ( dwTimeCur - dwTimeStart ) / 1000.0f;
    }

    // Clear the back buffer
    float ClearColor[4] = { 0.0f, 0.125f, 0.3f, 1.0f }; // red, green, blue, alpha
    g_pImmediateContext->ClearRenderTargetView( g_pRenderTargetView, ClearColor );

    // Clear the depth buffer to 1.0 (max depth)
    g_pImmediateContext->ClearDepthStencilView( g_pDepthStencilView, D3D11_CLEAR_DEPTH, 1.0f, 0 );

    g_World = XMMatrixRotationY( t * 0.5F );
	//g_World = XMMatrixIdentity();
	
	// Update the constant buffer
	XMMATRIX g_WorldView = g_World * g_View;  
    cb.mWorld = XMMatrixTranspose( g_World );
	cb.WorldViewProjection = XMMatrixTranspose( g_WorldView * g_Projection );
	cb.LightPosition = XMFLOAT4(0,2,1,0);
	XMStoreFloat4( &cb.EyePosition, Eye );
    g_pImmediateContext->UpdateSubresource( g_pConstantBuffer, 0, NULL, &cb, 0, 0 );

    // Render the parallax mapped plane
    g_pImmediateContext->VSSetShader( g_pVertexShader, NULL, 0 );
    g_pImmediateContext->VSSetConstantBuffers( 0, 1, &g_pConstantBuffer );
    g_pImmediateContext->PSSetShader( g_pPixelShader, NULL, 0 );
    g_pImmediateContext->PSSetConstantBuffers( 0, 1, &g_pConstantBuffer );
	g_pImmediateContext->PSSetSamplers( 0, 1, &g_pSamplerLinear );
    g_pImmediateContext->PSSetShaderResources( 0, 1, &g_pColorTexRV );
	g_pImmediateContext->PSSetShaderResources( 1, 1, &g_pHeightTexRV );
    g_pImmediateContext->DrawIndexed( 6, 0, 0 );

    // Present our back buffer to our front buffer
    g_pSwapChain->Present( 0, 0 );
}

//FX KODU

//-----------------------------------------------------------------------------------------
// This effect file implements the baseline Parallax Occlusion Mapping 
// as described in the chapter "A Closer Look At Parallax Occlusion Mapping" by Jason Zink.
// http://www.gamedev.net/page/resources/_/technical/graphics-programming-and-theory/a-closer-look-at-parallax-occlusion-mapping-r3262
//-----------------------------------------------------------------------------------------

Texture2D ColorMap : register(t0);
Texture2D NormalHeightMap : register(t1);

SamplerState samLinear : register(s0);

cbuffer ConstantBuffer : register(b0)
{
    matrix World;
    matrix WorldViewProjection;
    float4 LightPosition;
    float4 EyePosition;
    float fHeightMapScale;
    int nMaxSamples;
    int key;
};

float nMinSamples = 4;

struct vertex
{
    float3 position : POSITION;
    float2 texcoord : TEXCOORD0;
    float3 normal : NORMAL;
    float3 tangent : TANGENT;
    float3 binormal : BINORMAL;
};

struct fragment
{
    float4 position : SV_Position;
    float2 texcoord : TEXCOORD0;
    float3 eye : TEXCOORD1;
    float3 normal : TEXCOORD2;
    float3 light : TEXCOORD3;
};

struct pixel
{
    float4 color : SV_Target0;
};

//-----------------------------------------------------------------------------
// Vertex Shader
//-----------------------------------------------------------------------------
fragment VS(vertex IN)
{
    fragment OUT;

	// Calculate the world space position of the vertex and view points.
    float3 P = mul(float4(IN.position, 1), World).xyz;
    float3 N = IN.normal;
    float3 E = P - EyePosition.xyz;
    float3 L = LightPosition.xyz - P;

	// The per-vertex tangent, binormal, normal form the tangent to object 
	// space rotation matrix.  Multiply by the world matrix to form tangent
	// to world space rotation matrix.  Then transpose the matrix to form
	// the inverse tangent to world space, otherwise called the world to
	// tangent space rotation matrix.
    float3x3 tangentToWorldSpace;

    tangentToWorldSpace[0] = mul(normalize(IN.tangent), World);
    tangentToWorldSpace[1] = mul(normalize(IN.binormal), World);
    tangentToWorldSpace[2] = mul(normalize(IN.normal), World);
	
    float3x3 worldToTangentSpace = transpose(tangentToWorldSpace);

	// Output the projected vertex position for rasterization and
	// pass through the texture coordinates.
    OUT.position = mul(float4(IN.position, 1), WorldViewProjection);
    OUT.texcoord = IN.texcoord;

	// Output the tangent space normal, eye, and light vectors.
    OUT.eye = mul(E, worldToTangentSpace);
    OUT.normal = mul(N, worldToTangentSpace);
    OUT.light = mul(L, worldToTangentSpace);

    return OUT;
}

//-----------------------------------------------------------------------------
// Pixel Shader
//-----------------------------------------------------------------------------
pixel PS(fragment IN)
{
    pixel OUT;

	// Calculate the parallax offset vector max length.
	// This is equivalent to the tangent of the angle between the
	// viewer position and the fragment location.
    float fParallaxLimit = -length(IN.eye.xy) / IN.eye.z;

	// Scale the parallax limit according to heightmap scale.
    fParallaxLimit *= fHeightMapScale;
    //KOD EKLENDI
	//parallax yok
    if (key == 0 || key == 2)
    {
        fParallaxLimit = 0;
    }
	// Calculate the parallax offset vector direction and maximum offset.
    float2 vOffsetDir = normalize(IN.eye.xy);
    float2 vMaxOffset = vOffsetDir * fParallaxLimit;
	
	// Calculate the geometric surface normal vector, the vector from
	// the viewer to the fragment, and the vector from the fragment
	// to the light.
    float3 N = normalize(IN.normal);
    float3 E = normalize(IN.eye);
    float3 L = normalize(IN.light);

	// Calculate how many samples should be taken along the view ray
	// to find the surface intersection.  This is based on the angle
	// between the surface normal and the view vector.
    int nNumSamples = (int) lerp(nMaxSamples, nMinSamples, dot(E, N));
	
	// Specify the view ray step size.  Each sample will shift the current
	// view ray by this amount.
    float fStepSize = 1.0 / (float) nNumSamples;

	// Calculate the texture coordinate partial derivatives in screen
	// space for the tex2Dgrad texture sampling instruction.
    float2 dx = ddx(IN.texcoord);
    float2 dy = ddy(IN.texcoord);

	// Initialize the starting view ray height and the texture offsets.
    float fCurrRayHeight = 1.0;
    float2 vCurrOffset = float2(0, 0);
    float2 vLastOffset = float2(0, 0);
	
    float fLastSampledHeight = 1;
    float fCurrSampledHeight = 1;

    int nCurrSample = 0;

    while (nCurrSample < nNumSamples)
    {
		// Sample the heightmap at the current texcoord offset.  The heightmap 
		// is stored in the alpha channel of the height/normal map.
        fCurrSampledHeight = NormalHeightMap.SampleGrad(samLinear, IN.texcoord + vCurrOffset, dx, dy).a;

		// Test if the view ray has intersected the surface.
        if (fCurrSampledHeight > fCurrRayHeight)
        {
			// Find the relative height delta before and after the intersection.
			// This provides a measure of how close the intersection is to 
			// the final sample location.
            float delta1 = fCurrSampledHeight - fCurrRayHeight;
            float delta2 = (fCurrRayHeight + fStepSize) - fLastSampledHeight;
            float ratio = delta1 / (delta1 + delta2);

			// Interpolate between the final two segments to 
			// find the true intersection point offset.
            vCurrOffset = (ratio) * vLastOffset + (1.0 - ratio) * vCurrOffset;
			
			// Force the exit of the while loop
            nCurrSample = nNumSamples + 1; // Donguden cikmak icin	
        }
        else
        {
			// The intersection was not found.  Now set up the loop for the next
			// iteration by incrementing the sample count,
            nCurrSample++;

			// take the next view ray height step,
            fCurrRayHeight -= fStepSize;
			
			// save the current texture coordinate offset and increment
			// to the next sample location, 
            vLastOffset = vCurrOffset;
            vCurrOffset += fStepSize * vMaxOffset;

			// and finally save the current heightmap height.
            fLastSampledHeight = fCurrSampledHeight;
        }
    }
	
	//fCurrSampledHeight = NormalHeightMap.SampleGrad( samLinear, IN.texcoord + vCurrOffset, dx, dy ).a;
	//
	//while ( fCurrSampledHeight < fCurrRayHeight )
	//{
	//		fCurrRayHeight -= fStepSize;
	//		
	//		vCurrOffset += fStepSize * vMaxOffset;			
	//		
	//		fCurrSampledHeight = NormalHeightMap.SampleGrad( samLinear, IN.texcoord + vCurrOffset, dx, dy ).a;
	//}
	
	// Calculate the final texture coordinate at the intersection point.
    float2 vFinalCoords = IN.texcoord + vCurrOffset;

	// Sample the colormap at the final intersection point.
    float4 vFinalColor = ColorMap.SampleGrad(samLinear, vFinalCoords, dx, dy);
	//kod eklendi
    float3 vFinalNormal = NormalHeightMap.SampleGrad(samLinear, vFinalCoords, dx, dy); //.a;
	//bump yok
    if (key == 0 || key == 1)
    {
        vFinalNormal = N;
    }
    else
    {
		// Expand the final normal vector from [0,1] to [-1,1] range.
        vFinalNormal = vFinalNormal * 2.0f - 1.0f;
    }
	
	// Expand the final normal vector from [0,1] to [-1,1] range.
	//vFinalNormal = vFinalNormal * 2.0f - 1.0f;

	// Shade the fragment based on light direction and normal.
    float3 vAmbient = vFinalColor.rgb * 0.2f;
    float3 vDiffuse = vFinalColor.rgb * max(0.0f, dot(L, vFinalNormal.xyz)) * 0.8f;
	
    float3 reflection = reflect(-L, vFinalNormal);
    float3 viewDirection = normalize(-E);
    float specularAngle = max(0.0f, dot(reflection, viewDirection));
    float3 vSpecular = vFinalColor.rgb * pow(specularAngle, 64.0f);
    vFinalColor.rgb = vAmbient + vDiffuse + vSpecular;

    OUT.color = float4(vFinalColor.rgb, 1.0f);
	
	/*
	//Deney Sorulari

		float3 reflection = reflect(-L, N);
		float3 viewDirection = normalize(-E);
		float specularAngle = max(0.0f, dot(reflection, viewDirection));
		float3 vSpecular = vFinalColor.rgb * pow(specularAngle, 64.0f);
		vFinalColor.rgb = vAmbient + vDiffuse + vSpecular;
		OUT.color = float4(vFinalColor.rgb, 1.0f);

	*/
	


// Define the 'GRIDLINES' preprocessor directive to draw the gridlines on the texture surface.
// This helps visualization of the simulated surface.

//#define GRIDLINES
#ifdef GRIDLINES
	float2 vGridCoords = frac( vFinalCoords * 10.0f );
	if ( ( vGridCoords.x < 0.025f ) || ( vGridCoords.x > 0.975f ) )
		OUT.color = float4( 1.0f, 0.0f, 0.0f, 1.0f );
	if ( ( vGridCoords.y < 0.025f ) || ( vGridCoords.y > 0.975f ) )
		OUT.color = float4( 0.0f, 0.0f, 1.0f, 1.0f );
#endif

    return OUT;
}
