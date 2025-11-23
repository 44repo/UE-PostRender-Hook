# Hooking UGameViewportClient::PostRender In UE Games

## Step 1: Finding the Function in IDA Pro

First, we open IDA Pro and use the string search feature to locate our target function.

![String Search](https://i.ibb.co/0VMwxWdS/1.png)

You can use any of these strings to find `UGameViewportClient::PostRender`:
- `PausedMessage`
- `LoadingMessage`
- `SavingMessage`
- `ConnectingMessage`
- `PrecachingMessage`
- `Waiting to connect...`
- `GameViewportClient`
- `PRECACHING`

For this example, I'll use **"LoadingMessage"**.

![Search Result](https://i.ibb.co/0pyRk1Tb/2.png)

After searching, we can see the function that uses this string:

![Function Found](https://i.ibb.co/VYNDf5KF/3.png)

---

## Step 2: Understanding the Decompiled Code

Let's break down what this function does:
```cpp
void __fastcall sub_142A04E00(__int64 *a1, __int64 a2)
```

### Parameters:
- **`a1`** = `this` pointer (points to `UGameViewportClient` object)
- **`a2`** = Canvas pointer (used for drawing on screen)

### What the function does:
```cpp
if ( !*((_BYTE *)a1 + 136) )
```
First, it checks if rendering is enabled (byte at offset 136 must be false/0).
```cpp
v4 = 0;
if ( (a1[1] & 0x10) == 0 )
    v4 = a1[4];
switch ( *(_BYTE *)(v4 + 2216) )
```
Then it gets a pointer to the game world/engine state and checks a byte at offset 2216. This byte is an **enum** that tells us the current game state.

The function uses a switch statement to display different messages based on game state (PAUSED, LOADING, SAVING, CONNECTING, PRECACHING, etc.). Each case does basically the same thing:

1. Gets the vtable pointer: `v18 = *a1`
2. Gets localized text for the message (like "LOADING", "PAUSED", etc.)
3. Calls a drawing function to display it on screen
4. Handles reference counting for the text objects

### Reference Counting:
```cpp
if ( v8 && _InterlockedExchangeAdd(v8 + 2, 0xFFFFFFFF) == 1 )
{
    (**(void (__fastcall ***)(volatile signed __int32 *))v8)(v8);
    if ( _InterlockedExchangeAdd(v8 + 3, 0xFFFFFFFF) == 1 )
        (*(void (__fastcall **)(volatile signed __int32 *, __int64))(*(_QWORD *)v8 + 8LL))(v8, 1);
}
```
This handles **reference counting** for the localized text objects. It decrements the reference count and deletes the object if no one is using it anymore.

---

## Step 3: Calculating the VTable Index

Looking at this critical line:
```cpp
(*(void (__fastcall **)(__int64 *, __int64, __int64))(v18 + 0x328))(a1, a2, v20);
```

Where:
- **`v18 = *a1`** = Gets the vtable pointer from the object
- **`v18 + 0x328`** = Goes 0x328 bytes forward from the vtable start

### Why subtract 0x10?

When you dereference `a1` to get `v18`, you're getting the **actual runtime vtable pointer**. However, there's a difference between:
1. **Runtime vtable pointer** (what `v18` contains)
2. **IDA's vtable display** (what you see in the vtable viewer)

In C++ (especially MSVC), the vtable pointer points to the **first virtual function**, but there's **RTTI metadata** (0x10 bytes) **before** this pointer that IDA includes in its display.
```
Memory Layout:
[RTTI info - 0x10 bytes]  <- IDA vtable viewer starts here
[First virtual function]   <- v18 points HERE (runtime)
[Second virtual function]
...
```

### The Calculation:

So we see **0x328**, we subtract **0x10**, the result is **0x318**.  
We divide **0x318** by **8** (pointer size on x64), the result is **0x63**.
```
IDA_vtable_index = (runtime_offset - 0x10) / 8
                 = (0x328 - 0x10) / 8
                 = 0x318 / 8
                 = 0x63 (99 in decimal)
```

**So PostRender index = 0x63**

---

## Step 4: Hooking the Function

Now let's move to the code part to see how we can hook it.

### Setting Up the Hook

First, we define our hook function and store the original:
```cpp
typedef void(__thiscall* PostRenderOriginal)(SDK::UGameViewportClient*, SDK::UCanvas*);
PostRenderOriginal oPostRender = nullptr;

void hkPostRender(SDK::UGameViewportClient* Viewport, SDK::UCanvas* Canvas)
{
    // Call the original function first
    if (oPostRender)
        oPostRender(Viewport, Canvas);
    
    // Safety check
    if (!Canvas) return;
    
    // Find the font we want to use
    SDK::UFont* font = SDK::UObject::FindObject<SDK::UFont>("Font Roboto.Roboto");
    if (!font) return;
    
    // Draw our custom text on screen
    Canvas->K2_DrawText(
        font,
        SDK::FString(L"github.com/44repo\n https://t.me/iam_44"),
        { 10.f, 10.f },           // Position (x, y)
        { 1.f, 1.f },             // Scale
        { 0.f, 1.f, 0.f, 1.f },   // Color (green)
        0.f,                       // Rotation
        { 0.f, 0.f, 0.f, 1.f },   // Shadow color
        { 1.f, 1.f },             // Shadow offset
        false, false, true,        // Flags
        { 0.f, 0.f, 0.f, 1.f }    // Outline color
    );
}
```

### Installing the VTable Hook
```cpp
// Get the vtable from the viewportClient object
void** vtable = *(void***)viewportClient;

// Save the original function
oPostRender = (PostRenderOriginal)vtable[0x63];  // postrenderindex = 0x63

// Change memory protection to allow writing
DWORD oldProtect;
VirtualProtect(&vtable[0x63], sizeof(void*), PAGE_EXECUTE_READWRITE, &oldProtect);

// Replace the function pointer with our hook
vtable[0x63] = &hkPostRender;

// Restore original memory protection
VirtualProtect(&vtable[0x63], sizeof(void*), oldProtect, &oldProtect);
```

### What This Does:

1. **`void** vtable = *(void***)viewportClient;`**  
   Gets the vtable pointer from the viewportClient object (dereferences it three times)

2. **`oPostRender = (PostRenderOriginal)vtable[0x63];`**  
   Saves the original PostRender function pointer so we can call it later

3. **`VirtualProtect(..., PAGE_EXECUTE_READWRITE, ...);`**  
   Changes memory permissions so we can modify the vtable (it's normally read-only)

4. **`vtable[0x63] = &hkPostRender;`**  
   Replaces the original function pointer with our hook function

5. **`VirtualProtect(..., oldProtect, ...);`**  
   Restores the original memory permissions

I've already set this up as you can see in this screenshot:

![Implementation](https://i.ibb.co/B5LBY2D5/Screenshot-1.png)

---

## Results

After hooking PostRender, our custom text is drawn on screen every frame!

![Results](https://i.ibb.co/35NBctLg/results.png)
