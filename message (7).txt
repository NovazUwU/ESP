#include <iostream>
#include <vector>
#include <string>
#include "windows.h"
#include "tlhelp32.h"
#include "imgui.h"   
#include "imgui_impl_dx11.h"
#include "imgui_impl_win32.h"
#include "dwmapi.h"
#include "d3d11.h"
#include "DirectXMath.h"

#pragma comment(lib, "d3d11.lib")

using namespace DirectX;

#define string std::string 
#define vector std::vector  

extern IMGUI_IMPL_API LRESULT ImGui_ImplWin32_WndProcHandler(HWND hWnd, UINT Msg, WPARAM wParam, LPARAM lParam);

struct Vector2 {
    float x; float y; 

    Vector2() : x(0), y(0) {}
    Vector2(float x, float y) : x(x), y(y) {}
};

struct Vector3 {
    float x; float y; float z;

    Vector3() : x(0), y(0), z(0) {}
    Vector3(float x, float y, float z) : x(x), y(y), z(z) {}
};

struct PVector3 {
    uintptr_t x; uintptr_t y; uintptr_t z;

    PVector3() : x(0), y(0), z(0) {}
    PVector3(uintptr_t x, uintptr_t y, uintptr_t z) : x(x), y(y), z(z) {} 
};

HANDLE Handle;
DWORD ProcessId;
uintptr_t BaseAddress;
uintptr_t ModuleSize;
uintptr_t DataModel;
uintptr_t Camera;
Vector2 Viewport;
int TopBar;
XMMATRIX Matrix;
HWND RobloxWindow;
int R15 = 0;

template <typename T>
T Read(uintptr_t Address) {
    T Response;
    ReadProcessMemory(Handle, (LPCVOID)Address, (LPVOID)&Response, sizeof(T), 0);   
    return Response;
}

template <typename T>
BOOL Write(uintptr_t Address, T Value) {
    BOOL Response = WriteProcessMemory(Handle, (LPVOID)Address, (LPVOID)&Value, sizeof(T), 0);  
    return Response;
}

BOOL Read(uintptr_t Address, uintptr_t Buffer, SIZE_T Size) {
    SIZE_T BytesRead;
    ReadProcessMemory(Handle, (LPCVOID)Address, (LPVOID)Buffer, Size, &BytesRead);  
    return BytesRead == Size;
}

string ReadString(uintptr_t Address) {
    char Name[100];
    ReadProcessMemory(Handle, (LPCVOID)Address, (LPVOID)&Name, sizeof(Name), 0);
    return string(Name);
}

SIZE_T Query(LPCVOID Address, PMEMORY_BASIC_INFORMATION Buffer, SIZE_T Size) {
    return VirtualQueryEx(Handle, Address, Buffer, Size);
}
 
LRESULT CALLBACK WinProc(HWND hWnd, UINT Msg, WPARAM wParam, LPARAM lParam) {
    if (ImGui_ImplWin32_WndProcHandler(hWnd, Msg, wParam, lParam)) {
        return 0L;
    }

    switch (Msg) {
        case WM_DESTROY: {
            PostQuitMessage(0);
            return 0;
        }
    }

    return DefWindowProcW(hWnd, Msg, wParam, lParam); 
}

DWORD GetRobloxProcessId() {
    HANDLE Snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    if (Snapshot == INVALID_HANDLE_VALUE) {
        return 0;
    }
    
    PROCESSENTRY32 Process; 
    Process.dwSize = sizeof(PROCESSENTRY32);

    BOOL Status = Process32First(Snapshot, &Process); 

    if (!Status) {
        CloseHandle(Snapshot);
        return 0;
    }

    while (Status) {
        WCHAR* Name = Process.szExeFile;
        if (wcscmp(Name, L"RobloxPlayerBeta.exe") == 0) {
            TopBar = 22;
            RobloxWindow = FindWindowA(0, "Roblox");
        } else if (wcscmp(Name, L"Windows10Universal.exe") == 0) {
            TopBar = 30;
            RobloxWindow = FindWindowA(0, "Roblox"); 
        } else if (wcscmp(Name, L"RobloxStudioBeta.exe") == 0) { 
            TopBar = 175;
            RobloxWindow = FindWindowA(0, "RobloxStudio"); 
        }
        if (TopBar) {
            CloseHandle(Snapshot);
            return Process.th32ProcessID;
        }
        Status = Process32Next(Snapshot, &Process);   
    }

    CloseHandle(Snapshot);

    return 0;
}

int Initialize() {
    ProcessId = GetRobloxProcessId();

    if (!ProcessId) {
        std::cout << "Failed To Get PID\n";
        return 0;
    }

    if (!RobloxWindow) {
        std::cout << "Failed To Find Window\n";
        return 0;
    }

    Handle = OpenProcess(PROCESS_ALL_ACCESS, 0, ProcessId);

    if (!Handle) {
        std::cout << "Failed To Get Handle\n";
        return 0;
    }

    return 1;
}

string GetName(uintptr_t Instance) {
    uintptr_t Name = Read<uintptr_t>(Instance + 0x48);
    uintptr_t Check = Read<uintptr_t>(Name + 0x18);

    if (Check == 0x1F || Check == 0x2F) { Name = Read<uintptr_t>(Name); } 
    
    return ReadString(Name);
}


string GetClass(uintptr_t Instance) {
    if (!Instance) return "";
    uintptr_t Pointer = Read<uintptr_t>(Instance + 0x18);
    if (!Pointer) return "";
    uintptr_t ClassName = Read<uintptr_t>(Pointer + 0x8);
    if (!ClassName) return "";
    uintptr_t Check = Read<uintptr_t>(ClassName + 0x18);

    if (Check == 0x1F || Check == 0x2F) { ClassName = Read<uintptr_t>(ClassName); }

    return ReadString(ClassName);
}

vector<uintptr_t> GetChildren(uintptr_t Instance) {
    vector<uintptr_t> Children;

    uintptr_t Pointer = Read<uintptr_t>(Instance + 0x50);
    if (!Pointer) return Children;

    uintptr_t ChildrenStart = Read<uintptr_t>(Pointer);
    uintptr_t ChildrenEnd = Read<uintptr_t>(Pointer + 0x8);
    if (!ChildrenStart || !ChildrenEnd) return Children;
    
    while (ChildrenStart < ChildrenEnd) {
        uintptr_t Child = Read<uintptr_t>(ChildrenStart);
        if (Child) {
            Children.push_back(Child);
        }
        ChildrenStart += 0x10;
    }
    return Children;
}

uintptr_t FindFirstChild(uintptr_t Instance, string Child) {
    for (uintptr_t v : GetChildren(Instance)) { if (GetName(v) == Child) { return v; } }
    return 0;
}

uintptr_t FindFirstChildOfClass(uintptr_t Instance, string Class) {
    for (uintptr_t v : GetChildren(Instance)) { if (GetClass(v) == Class) { return v; } }
    return 0;
}

PVector3 GetPosition(uintptr_t Instance) {
    if (GetClass(Instance) == "Camera") {
        return PVector3(Instance + 0x128, Instance + 0x12C, Instance + 0x130);
    } else {
        uintptr_t Primitive = Read<uintptr_t>(Instance + 0x148);
        return PVector3(Primitive + 0x13C, Primitive + 0x140, Primitive + 0x144);
    }
}

PVector3 GetSize(uintptr_t Instance) {
    uintptr_t Primitive = Read<uintptr_t>(Instance + 0x148);
    return PVector3(Primitive + 0x27C, Primitive + 0x280, Primitive + 0x284);
}

PVector3 GetRightVector(uintptr_t Instance) {
    if (GetClass(Instance) == "Camera") {
        return PVector3(Instance + 0x104, Instance + 0x110, Instance + 0x11C);
    } else {
        uintptr_t Primitive = Read<uintptr_t>(Instance + 0x148);
        return PVector3(Primitive + 0x160, Primitive + 0x124, Primitive + 0x130);
    }
}

PVector3 GetUpVector(uintptr_t Instance) {
    if (GetClass(Instance) == "Camera") {
        return PVector3(Instance + 0x108, Instance + 0x114, Instance + 0x120);
    } else {
        uintptr_t Primitive = Read<uintptr_t>(Instance + 0x148);
        return PVector3(Primitive + 0x164, Primitive + 0x128, Primitive + 0x134);
    }
}

PVector3 GetLookVector(uintptr_t Instance) {
    if (GetClass(Instance) == "Camera") {
        return PVector3(Instance + 0x10C, Instance + 0x118, Instance + 0x124);
    } else {
        uintptr_t Primitive = Read<uintptr_t>(Instance + 0x148);
        return PVector3(Primitive + 0x130, Primitive + 0x174, Primitive + 0x138);
    }
}

Vector3 ReadVector(PVector3 Vec) { return Vector3(Read<float>(Vec.x), Read<float>(Vec.y), Read<float>(Vec.z));}

Vector3 Multiply(Vector3 Vector, float Amount) { return Vector3(Vector.x * Amount, Vector.y * Amount, Vector.z * Amount); }
Vector3 Add(Vector3 Vector1, Vector3 Vector2) { return Vector3(Vector1.x + Vector2.x, Vector1.y + Vector2.y, Vector1.z + Vector2.z); }
Vector3 Add(Vector3 Vector1, float Amount) { return Vector3(Vector1.x + Amount, Vector1.y + Amount, Vector1.z + Amount); }
Vector3 Sub(Vector3 Vector1, Vector3 Vector2) { return Vector3(Vector1.x - Vector2.x, Vector1.y - Vector2.y, Vector1.z - Vector2.z); }
Vector3 Sub(Vector3 Vector1, float Amount) { return Vector3(Vector1.x - Amount, Vector1.y - Amount, Vector1.z - Amount); }

float rad(float Degrees) { return Degrees * (3.141592653989 / 180); }
float Index(int Row, int Collum) { return Matrix.r[Row].m128_f32[Collum]; }

bool WorldToScreen(Vector3 Position, ImVec2& Screen) { 
    Vector3 P;
    Vector3 Point(Index(0, 3), Index(1, 3), Index(2, 3));
    P = Vector3(Index(0, 0), Index(1, 0), Index(2, 0));
    Point = Add(Point, Multiply(P, Position.x));
    P = Vector3(Index(0, 1), Index(1, 1), Index(2, 1));
    Point = Add(Point, Multiply(P, Position.y));
    P = Vector3(Index(0, 2), Index(1, 2), Index(2, 2));
    Point = Add(Point, Multiply(P, Position.z));

    if (Point.z > -0.1) return false;

    float AspectRatio = Viewport.x / Viewport.y;
    float Height = tan(rad(70) / 2);
    float Width = AspectRatio * Height;

    float x = (Point.x / Point.z) / -Width;
    float y = (Point.y / Point.z) / Height;

    Screen.x = Viewport.x * (0.5 + 0.5 * x);
    Screen.y = Viewport.y * (0.5 + 0.5 * y);

    return true;
}

ImDrawList* DrawList;
void DrawLine(ImVec2 From, ImVec2 To) { DrawList->AddLine(From, To, 0xFF0000FF, 1.0f); }

void UpdateViewportSize() {
    Viewport.x = Read<float>(Camera + 0x2CC);
    Viewport.y = Read<float>(Camera + 0x2D0);
}

void UpdateViewMatrix() {
    uintptr_t Pointer = Read<uintptr_t>(DataModel + 0x118);
    Pointer = Read<uintptr_t>(Pointer + 0x8);

    uintptr_t RenderView = Read<uintptr_t>(Pointer + 0x28);
    uintptr_t VisualEngine = Read<uintptr_t>(RenderView + 0x10);

    Read(VisualEngine + 0xBC, (uintptr_t)&Matrix, 64);
}

void UpdateDataModel() {
    MEMORY_BASIC_INFORMATION MemoryInfo;
    SIZE_T ReadA;

    for (LPCVOID Address = 0; Address < (LPCVOID)(0x7FFFFFFFFFF); Address = (LPCVOID)((uintptr_t)MemoryInfo.BaseAddress + MemoryInfo.RegionSize)) {
        ReadA = Query(Address, &MemoryInfo, sizeof(MemoryInfo));
        if (!(ReadA == sizeof(MemoryInfo))) break;

        DWORD Protection = MemoryInfo.Protect; 
        if (Protection & PAGE_NOACCESS || Protection & PAGE_GUARD) {
            std::cout << "Skipped Protected Page: " << Address << "\n";
            continue;
        }

        PUCHAR Buffer = (PUCHAR)malloc(MemoryInfo.RegionSize); 
        size_t ReadB;
        if (Buffer) {
            if (Address && Read((uintptr_t)Address, (uintptr_t)Buffer, MemoryInfo.RegionSize)) {  
                for (int i = 0; i < MemoryInfo.RegionSize; i += 8) {
                    uintptr_t Value;
                    memcpy(&Value, Buffer + i, 8);

                    if (Value && GetClass(Value) == "DataModel" && GetClass(Read<uintptr_t>(Value + 0x300)) == "Workspace" && FindFirstChildOfClass(Value, "ReplicatedFirst")) { 
                        DataModel = Value;
                    }
                }
            }
            free(Buffer);
            if (DataModel) break;
        }
    }
}

INT APIENTRY WinMain(HINSTANCE HInstance, HINSTANCE, LPSTR, INT Show) {
    AllocConsole();  
    FILE* f; 
    freopen_s(&f, "CONOUT$", "w", stdout); 

    if (!Initialize()) {
        std::cout << "Failed to initialize\n";
        Sleep(5000);
        return 0;
    }

    std::cout << "Scanning For DataModel\n";

    UpdateDataModel();
    if (!DataModel) {
        std::cout << "Failed To Find DataModel\n";
        Sleep(5000);
        return 0;
    }

    uintptr_t Workspace = Read<uintptr_t>(DataModel + 0x300);
    if (!(GetClass(Workspace) == "Workspace")) {
        std::cout << "Failed To Find Workspace\n";
        Sleep(5000);
        return 1;
    }

    uintptr_t Players = FindFirstChildOfClass(DataModel, "Players");
    if (!(GetClass(Players) == "Players")) {
        std::cout << "Failed To Find Players\n";
        Sleep(5000);
        return 1;
    }

    uintptr_t LocalPlayer = Read<uintptr_t>(Players + 0x238);
    if (!LocalPlayer) {
        for (int i = 0x1; i < 0x1000; i++) {
            uintptr_t Result = Read<uintptr_t>(Players + i);
            if (GetClass(Result) == "Player") {
                LocalPlayer = Result;
                break;
            }
        }
    }
    
    if (!LocalPlayer) {
        std::cout << "Failed To Find LocalPlayer\n";
        Sleep(5000);
        return 1;
    }

    string Name = GetName(LocalPlayer); 

    WNDCLASSEX WindowsClass;
    ZeroMemory(&WindowsClass, sizeof(WindowsClass));
    WindowsClass.cbSize = sizeof(WindowsClass);
    WindowsClass.style = CS_HREDRAW | CS_VREDRAW;
    WindowsClass.lpfnWndProc = WinProc;
    WindowsClass.cbClsExtra = 0;
    WindowsClass.cbWndExtra = 0;
    WindowsClass.hInstance = HInstance;
    WindowsClass.lpszMenuName = L"ESP";
    WindowsClass.lpszClassName = L"ESPUwU";

    DXGI_SWAP_CHAIN_DESC SwapDescriptor;
    ZeroMemory(&SwapDescriptor, sizeof(SwapDescriptor));
    SwapDescriptor.BufferCount = 2;
    SwapDescriptor.BufferDesc.Width = 0;
    SwapDescriptor.BufferDesc.Height = 0;
    SwapDescriptor.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    SwapDescriptor.BufferDesc.RefreshRate.Numerator = 500;
    SwapDescriptor.BufferDesc.RefreshRate.Denominator = 1;
    SwapDescriptor.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;
    SwapDescriptor.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    SwapDescriptor.SampleDesc.Count = 1; 
    SwapDescriptor.SampleDesc.Quality = 0; 
    SwapDescriptor.Windowed = TRUE; 
    SwapDescriptor.SwapEffect = DXGI_SWAP_EFFECT_DISCARD; 

    if (!RegisterClassExW(&WindowsClass)) {
        printf("Registration Failed: %lu\n", GetLastError());
        Sleep(5000);
        return 0;
    }
    
    HWND Window = CreateWindowExW(WS_EX_TOPMOST | WS_EX_TRANSPARENT | WS_EX_LAYERED, WindowsClass.lpszClassName, L"ESP", WS_POPUP, 0, TopBar, 1920, 1080, NULL, NULL, WindowsClass.hInstance, NULL);

    if (!Window) {
        printf("Failed To Create Window: %lu\n", GetLastError());
        Sleep(5000);
        return 0;
    }

    SwapDescriptor.OutputWindow = Window; 

    SetLayeredWindowAttributes(Window, RGB(0, 0, 0), BYTE(255), LWA_ALPHA);

    MARGINS Margins = { 1920, 1080, 0, TopBar };
    DwmExtendFrameIntoClientArea(Window, &Margins);
    
    ID3D11Device* D3DDevice; 
    ID3D11DeviceContext* D3DDeviceContext; 
    IDXGISwapChain* D3DSwapChain; 
    ID3D11RenderTargetView* D3DRenderTargetView;
    D3D_FEATURE_LEVEL D3DLevel; 

    HRESULT Result = D3D11CreateDeviceAndSwapChain(NULL, D3D_DRIVER_TYPE_HARDWARE,NULL, NULL, NULL, NULL, D3D11_SDK_VERSION, &SwapDescriptor, &D3DSwapChain, &D3DDevice, &D3DLevel, &D3DDeviceContext);
    
    if (FAILED(Result)) { 
        printf("Failed To Create Device: 0x%08X", Result);
        Sleep(5000);
        return 0;
    }
    
    ID3D11Texture2D* D3DBuffer;
    D3DSwapChain->GetBuffer(0, IID_PPV_ARGS(&D3DBuffer));
    
    if (D3DBuffer) {
        D3DDevice->CreateRenderTargetView(D3DBuffer, nullptr, &D3DRenderTargetView);
        D3DBuffer->Release();
    } else {
        printf("Failed To Get Buffer");
        Sleep(1000);
        return 0;
    }

    ShowWindow(Window, Show);
    UpdateWindow(Window);

    ImGui::CreateContext();
    ImGui::StyleColorsDark();

    ImGui_ImplWin32_Init(Window);
    ImGui_ImplDX11_Init(D3DDevice, D3DDeviceContext);

    constexpr float ClearColor[4]{ 0, 0, 0, 0 };

    bool Running = true;
    while (Running) {
        MSG Msg;
        while (PeekMessage(&Msg, NULL, 0, 0, PM_REMOVE)) {
            TranslateMessage(&Msg);
            DispatchMessage(&Msg);

            if (Msg.message == WM_QUIT) {
                Running = false;
            }
        }

        if (!Running) break;

        ImGui_ImplDX11_NewFrame();
        ImGui_ImplWin32_NewFrame();

        ImGui::NewFrame();

        DrawList = ImGui::GetBackgroundDrawList();

        Camera = FindFirstChildOfClass(Workspace, "Camera");
        UpdateViewMatrix();
        UpdateViewportSize();

        RECT Rect;
        if (DwmGetWindowAttribute(RobloxWindow, DWMWA_EXTENDED_FRAME_BOUNDS, &Rect, sizeof(RECT)) == S_OK) {
            if (Rect.top < 0) { Rect.top = 0; }
            SetWindowPos(Window, NULL, Rect.left, Rect.top + TopBar, Viewport.x, Viewport.y, SWP_NOSIZE | SWP_NOZORDER); 
        }

        for (uintptr_t v : GetChildren(Players)) { 
            if (GetClass(v) == "Player" && GetName(v) != Name) {
                uintptr_t Character = Read<uintptr_t>(v + 0x188);  
                if (!Character) continue;

                uintptr_t Root = FindFirstChild(Character, "HumanoidRootPart");
                if (!Root) continue;

                Vector3 Position = ReadVector(GetPosition(Root));
                Vector3 RightVector = ReadVector(GetRightVector(Root));
                Vector3 UpVector = ReadVector(GetUpVector(Root));
                Vector3 LookVector = ReadVector(GetLookVector(Root));
                LookVector.z = -LookVector.z; // LookVector Z Is Inverted In Memory
                LookVector.y = -LookVector.y; // Inverted

                Vector3 Look = Multiply(LookVector, 1.5);
                Vector3 Up = Multiply(UpVector, 3.5);
                Vector3 Right = Multiply(RightVector, 2.5);

                Vector3 BaseTR = Add(Add(Position, Right), Vector3(Up.x, Up.y - 0.5, Up.z)); 
                Vector3 TopRightFront = Add(BaseTR, Look); 
                Vector3 TopRightBack = Sub(BaseTR, Look); 

                Vector3 BaseTL = Add(Sub(Position, Right), Vector3(Up.x, Up.y - 0.5, Up.z));
                Vector3 TopLeftFront = Add(BaseTL, Look); 
                Vector3 TopLeftBack = Sub(BaseTL, Look); 
                 
                Vector3 BaseBR = Sub(Add(Position, Right), Up); 
                Vector3 BottomRightFront = Add(BaseBR, Look); 
                Vector3 BottomRightBack = Sub(BaseBR, Look);
                 
                Vector3 BaseBL = Sub(Sub(Position, Right), Up);  
                Vector3 BottomLeftFront = Add(BaseBL, Look); 
                Vector3 BottomLeftBack = Sub(BaseBL, Look);

                ImVec2 TRF, TLF, BRF, BLF, TRB, TLB, BRB, BLB; 

                if (!WorldToScreen(TopRightFront, TRF) || !WorldToScreen(TopLeftFront, TLF) || !WorldToScreen(BottomRightFront, BRF) || !WorldToScreen(BottomLeftFront, BLF)) continue; 
                if (!WorldToScreen(TopRightBack, TRB) || !WorldToScreen(TopLeftBack, TLB) || !WorldToScreen(BottomRightBack, BRB) || !WorldToScreen(BottomLeftBack, BLB)) continue; 

                DrawLine(TRF, TLF); DrawLine(TRF, BRF); DrawLine(TLF, BLF); DrawLine(BRF, BLF);
                DrawLine(TRB, TLB); DrawLine(TRB, BRB); DrawLine(TLB, BLB); DrawLine(BRB, BLB);
                DrawLine(TLB, TLF); DrawLine(TRB, TRF); DrawLine(BLB, BLF); DrawLine(BRB, BRF);
            }
        }

        ImGui::Render();

        D3DDeviceContext->OMSetRenderTargets(1, &D3DRenderTargetView, NULL); 
        D3DDeviceContext->ClearRenderTargetView(D3DRenderTargetView, ClearColor); 
    
        ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());

        D3DSwapChain->Present(0, 0);
    }

    ImGui_ImplDX11_Shutdown();
    ImGui_ImplWin32_Shutdown();

    ImGui::DestroyContext();

    if (D3DSwapChain) D3DSwapChain->Release();
    if (D3DDeviceContext) D3DDeviceContext->Release();
    if (D3DDevice) D3DDevice->Release();
    if (D3DRenderTargetView) D3DRenderTargetView->Release();

    DestroyWindow(Window);
    UnregisterClassW(WindowsClass.lpszClassName, WindowsClass.hInstance);

    if (f) { fclose(f); }
    FreeConsole();
}
