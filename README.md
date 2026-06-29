// ============ AIMLOCK_COLOR_ANDROID.CPP ============
// Biên dịch: ndk-build hoặc CMake với toolchain android-ndk
#include <jni.h>
#include <string>
#include <android/log.h>
#include <dlfcn.h>
#include <substrate.h>   // Hoặc Dobby

#define LOG_TAG "AimLockColor"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

uintptr_t gameBase = 0;
bool aimEnabled = true;
bool colorEnabled = true;

// Offsets giả định (cần tìm bằng IDA)
#define OFFSET_WEAPON_COLOR 0x00ABCDEF  // Địa chỉ màu súng
#define OFFSET_SET_VIEW    0x00FEDCBA

// Hàm thay đổi màu súng
void SetWeaponColor(uint32_t color) {
    if (gameBase == 0) return;
    uintptr_t addr = gameBase + OFFSET_WEAPON_COLOR;
    // Ghi giá trị màu (ARGB) vào bộ nhớ
    // Cần dùng mprotect để thay đổi quyền ghi
    *(uint32_t*)addr = color;
}

// Hook aimlock
typedef void (*SetViewAngles_t)(float, float);
SetViewAngles_t oSetViewAngles = nullptr;

void hkSetViewAngles(float pitch, float yaw) {
    if (aimEnabled) {
        // Logic aimlock
        // Đổi màu súng sang đỏ khi khóa
        SetWeaponColor(0xFFFF0000); // Đỏ
    } else {
        SetWeaponColor(0xFFFFFFFF); // Trắng (gốc)
    }
    oSetViewAngles(pitch, yaw);
}

// Khởi tạo
void Init() {
    void* handle = dlopen("libil2cpp.so", RTLD_LAZY);
    if (handle) {
        gameBase = (uintptr_t)handle;
        LOGI("Game base: 0x%lx", gameBase);
        oSetViewAngles = (SetViewAngles_t)(gameBase + OFFSET_SET_VIEW);
        MSHookFunction((void*)oSetViewAngles, (void*)hkSetViewAngles, NULL);
        LOGI("Hook thanh cong!");
    }
}

extern "C" JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
    LOGI("Module loaded");
    Init();
    return JNI_VERSION_1_6;
}
// ============ AIMLOCK_COLOR.CPP ============
// Biên dịch: cl /EHsc /std:c++17 aimlock_color.cpp d3d9.lib detours.lib user32.lib
#include <windows.h>
#include <d3d9.h>
#include <detours.h>
#include <cmath>
#include <vector>

#pragma comment(lib, "d3d9.lib")
#pragma comment(lib, "detours.lib")

// Cấu trúc Vector3
struct Vector3 { float x, y, z; };

// Biến toàn cục
IDirect3DDevice9* g_pDevice = NULL;
float g_aimFov = 30.0f;
bool g_aimEnabled = true;
bool g_showColor = true;
DWORD g_crosshairColor = D3DCOLOR_ARGB(255, 0, 255, 0); // Màu xanh lá

// Hàm vẽ crosshair màu (gọi trong EndScene)
void DrawColorCrosshair(IDirect3DDevice9* pDevice) {
    if (!g_showColor) return;
    D3DVIEWPORT9 vp;
    pDevice->GetViewport(&vp);
    int cx = vp.Width / 2;
    int cy = vp.Height / 2;

    // Vẽ 4 đường thẳng tạo crosshair màu
    // (Dùng vertex buffer đơn giản – demo)
    // Thực tế cần tạo ID3DXLine hoặc vẽ sprite
}

// Hook EndScene để vẽ crosshair
typedef HRESULT(WINAPI* EndScene_t)(IDirect3DDevice9*);
EndScene_t oEndScene = NULL;

HRESULT WINAPI hkEndScene(IDirect3DDevice9* pDevice) {
    if (g_showColor) {
        DrawColorCrosshair(pDevice);
    }
    return oEndScene(pDevice);
}

// Hook SetViewAngles cho aimlock (giả định)
typedef void(*SetViewAngles_t)(float, float);
SetViewAngles_t oSetViewAngles = NULL;

void WINAPI hkSetViewAngles(float pitch, float yaw) {
    if (g_aimEnabled) {
        // Logic chọn mục tiêu gần nhất và điều chỉnh góc
        // (code tương tự như đã cung cấp ở tin nhắn trước)
        // Đổi màu crosshair sang đỏ khi khóa
        g_crosshairColor = D3DCOLOR_ARGB(255, 255, 0, 0);
    } else {
        g_crosshairColor = D3DCOLOR_ARGB(255, 0, 255, 0);
    }
    oSetViewAngles(pitch, yaw);
}

// Hàm khởi tạo hook
void InitHooks() {
    // Hook EndScene (cần tìm địa chỉ vtable của d3d9)
    // Hook SetViewAngles (cần pattern scan)
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID) {
    if (reason == DLL_PROCESS_ATTACH) {
        DisableThreadLibraryCalls(hModule);
        AllocConsole();
        freopen("CONOUT$", "w", stdout);
        printf("[AimLock] Khoi tao thanh cong!\n");
        InitHooks();
    }
    return TRUE;
}
