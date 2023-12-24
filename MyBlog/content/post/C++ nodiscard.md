---
title: "C++ nodiscard"
date: 2023-07-22
thumbnailImagePosition: left
thumbnailImage: c++/base/nodiscard/nd_thumb.jpg
coverImage: c++/base/nodiscard/nd_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- c++
- 2023
- July
tags:
- nodiscard
showSocial: false
---

学习一下关于C++基础知识nodiscard。


<!--more-->
# 1介绍

nodiscard是c++17引入的一种标记符，其语法一般为[[nodiscard]]或[[nodiscard("string")]](c++20引入)，含义可以理解为“不应舍弃”。nodiscard一般用于标记函数的返回值或者某个类，当使用某个弃值表达式而不是cast to void 来调用相关函数时，编译器会发出相关warning。

# 2实例

## 2.1出现警告

```c++
#include <iostream>
using namespace std;

[[nodiscard]] int calculateSum(int a, int b) {
    return a + b;
}

int main() {
    calculateSum(2, 3);  // 没有使用返回值，可能会产生警告或错误
	cout << 1 << endl;
    return 0;
}
```

如果在真实clion编译结果是没有异常，有一个警告。

{{< image classes="fancybox center fig-100" src="/c++/base/nodiscard/nd_1.png" thumbnail="/c++/base/nodiscard/nd_1.png" title="">}}

如果是在线编译器，同样也会提示一个警告

{{< image classes="fancybox center fig-100" src="/c++/base/nodiscard/nd_2.png" thumbnail="/c++/base/nodiscard/nd_2.png" title="">}}

同样也适用于枚举类

```c++
#include <iostream>
using namespace std;

[[nodiscard]] int fi()              //修饰函数返回值
{
    return 1;
}

class [[nodiscard]] C{};            //修饰类
enum class [[nodiscard]] E{e1, e2}; //修饰枚举类

C fc()
{
    return C();
}

E fe()
{
    return E::e1;
}

int main()
{
    fi();    //没有使用fi的返回值，告警：ignoring return value of 'int fi()', declared with attribute nodiscard
    fc();    //没有使用fc的返回值，告警：ignoring returned value of type 'C', declared with attribute nodiscard
    fe();    //没有使用fe的返回值，告警：ignoring returned value of type 'E', declared with attribute nodiscard
    return 0;
}
```

结果

```c++
====================[ Build | CPPProject0710 | Debug ]==========================
"D:\Clion\CLion 2022.2.1\bin\cmake\win\bin\cmake.exe" --build D:\project\CPPProject0710\cmake-build-debug --target CPPProject0710 -j 3
[1/2] Building CXX object CMakeFiles/CPPProject0710.dir/main.cpp.obj
D:/project/CPPProject0710/main.cpp: In function 'int main()':
D:/project/CPPProject0710/main.cpp:23:7: warning: ignoring return value of 'int fi()', declared with attribute 'nodiscard' [-Wunused-result]
   23 |     fi();    //ignoring return value of 'int fi()', declared with attribute nodiscard
      |     ~~^~
D:/project/CPPProject0710/main.cpp:3:19: note: declared here
    3 | [[nodiscard]] int fi()              //Modifies the return value of the function
      |                   ^~
D:/project/CPPProject0710/main.cpp:24:7: warning: ignoring returned value of type 'C', declared with attribute 'nodiscard' [-Wunused-result]
   24 |     fc();    //ignoring returned value of type 'C', declared with attribute nodiscard
      |     ~~^~
D:/project/CPPProject0710/main.cpp:11:3: note: in call to 'C fc()', declared here
   11 | C fc()
      |   ^~
D:/project/CPPProject0710/main.cpp:8:21: note: 'C' declared here
    8 | class [[nodiscard]] C{};            //Modifier class
      |                     ^
D:/project/CPPProject0710/main.cpp:25:7: warning: ignoring returned value of type 'E', declared with attribute 'nodiscard' [-Wunused-result]
   25 |     fe();    //ignoring returned value of type 'E', declared with attribute nodiscard
      |     ~~^~
D:/project/CPPProject0710/main.cpp:16:3: note: in call to 'E fe()', declared here
   16 | E fe()
      |   ^~
D:/project/CPPProject0710/main.cpp:9:26: note: 'E' declared here
    9 | enum class [[nodiscard]] E{e1, e2}; //Modified enum class
      |                          ^
[2/2] Linking CXX executable CPPProject0710.exe

Build finished
```

## 2.2去除warning方法

将上述改成cast to void或者接收一个返回值就不会出现警告

```c++
#include <iostream>
using namespace std;
[[nodiscard]] int fi()              //Modifies the return value of the function
{
    return 1;
}

class [[nodiscard]] C{};            //Modifier class
enum class [[nodiscard]] E{e1, e2}; //Modified enum class

C fc()
{
    return C();
}

E fe()
{
    return E::e1;
}

int main()
{
    //fi();    //ignoring return value of 'int fi()', declared with attribute nodiscard
    int ret = fi();
    static_cast<void>(fi());
    //fc();    //ignoring returned value of type 'C', declared with attribute nodiscard
    C c = fc();
    static_cast<void>(fc());
    //fe();    //ignoring returned value of type 'E', declared with attribute nodiscard
    E e = fe();
    static_cast<void>(fe());
    return 0;
}
```

输出

```c++
====================[ Build | CPPProject0710 | Debug ]==========================
"D:\Clion\CLion 2022.2.1\bin\cmake\win\bin\cmake.exe" --build D:\project\CPPProject0710\cmake-build-debug --target CPPProject0710 -j 3
[1/2] Building CXX object CMakeFiles/CPPProject0710.dir/main.cpp.obj
[2/2] Linking CXX executable CPPProject0710.exe

Build finished
```

## 2.3nodiscard失效

```c++
#include <iostream>
using namespace std;
[[nodiscard]]int* fi()              //Modifies the return value of the function
{
    return new int(1);
}

class [[nodiscard]] C{};            //Modifier class
enum class [[nodiscard]] E{e1, e2}; //Modified enum class

C* fc()
{
    return new C();
}

E&& fe()
{
    return E::e1;
}

int main()
{
    //还是这段代码，这里虽然转化成指针形式，但是依然会出现warning
    fi();    
    fc();    
    fe();   

    return 0;
}
```

结果

```c++
====================[ Build | CPPProject0710 | Debug ]==========================
"D:\Clion\CLion 2022.2.1\bin\cmake\win\bin\cmake.exe" --build D:\project\CPPProject0710\cmake-build-debug --target CPPProject0710 -j 3
[1/2] Building CXX object CMakeFiles/CPPProject0710.dir/main.cpp.obj
D:/project/CPPProject0710/main.cpp: In function 'E&& fe()':
D:/project/CPPProject0710/main.cpp:18:15: warning: returning reference to temporary [-Wreturn-local-addr]
   18 |     return E::e1;
      |            ~~~^~
D:/project/CPPProject0710/main.cpp: In function 'int main()':
D:/project/CPPProject0710/main.cpp:23:7: warning: ignoring return value of 'int* fi()', declared with attribute 'nodiscard' [-Wunused-result]
   23 |     fi();    //ignoring return value of 'int fi()', declared with attribute nodiscard
      |     ~~^~
D:/project/CPPProject0710/main.cpp:3:19: note: declared here
    3 | [[nodiscard]]int* fi()              //Modifies the return value of the function
      |                   ^~
[2/2] Linking CXX executable CPPProject0710.exe

Build finished
```

上面显示只有fi()函数返回值存在[[nodiscard]]的warning操作，剩余E::e1的警告是出现了局部变量的引用，望读者不要这样写，这里只是为了举例说明。即类对象的引用和指针是可以做到去除[[nodiscard]]的warning操作的，普通函数的加引用或者指针依然会出现warning警告。

## 3.4在源码中举例

```c++
//frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.h
namespace HWC2 {
    class Display {
public:
    virtual ~Display();

    virtual hal::HWDisplayId getId() const = 0;
    virtual bool isConnected() const = 0;
    virtual void setConnected(bool connected) = 0; // For use by Device only
    virtual bool hasCapability(
            aidl::android::hardware::graphics::composer3::DisplayCapability) const = 0;
    virtual bool isVsyncPeriodSwitchSupported() const = 0;
    virtual bool hasDisplayIdleTimerCapability() const = 0;
    virtual void onLayerDestroyed(hal::HWLayerId layerId) = 0;

    [[nodiscard]] virtual hal::Error acceptChanges() = 0;
    [[nodiscard]] virtual base::expected<std::shared_ptr<HWC2::Layer>, hal::Error>
    createLayer() = 0;
    [[nodiscard]] virtual hal::Error getChangedCompositionTypes(
            std::unordered_map<Layer*, aidl::android::hardware::graphics::composer3::Composition>*
                    outTypes) = 0;
    [[nodiscard]] virtual hal::Error getColorModes(std::vector<hal::ColorMode>* outModes) const = 0;
    // Returns a bitmask which contains HdrMetadata::Type::*.
    [[nodiscard]] virtual int32_t getSupportedPerFrameMetadata() const = 0;
    [[nodiscard]] virtual hal::Error getRenderIntents(
            hal::ColorMode colorMode, std::vector<hal::RenderIntent>* outRenderIntents) const = 0;
    [[nodiscard]] virtual hal::Error getDataspaceSaturationMatrix(hal::Dataspace dataspace,
                                                                  android::mat4* outMatrix) = 0;

    [[nodiscard]] virtual hal::Error getName(std::string* outName) const = 0;
    [[nodiscard]] virtual hal::Error getRequests(
            hal::DisplayRequest* outDisplayRequests,
            std::unordered_map<Layer*, hal::LayerRequest>* outLayerRequests) = 0;
    [[nodiscard]] virtual hal::Error getConnectionType(ui::DisplayConnectionType*) const = 0;
    [[nodiscard]] virtual hal::Error supportsDoze(bool* outSupport) const = 0;
    [[nodiscard]] virtual hal::Error getHdrCapabilities(
            android::HdrCapabilities* outCapabilities) const = 0;
    [[nodiscard]] virtual hal::Error getDisplayedContentSamplingAttributes(
            hal::PixelFormat* outFormat, hal::Dataspace* outDataspace,
            uint8_t* outComponentMask) const = 0;
    [[nodiscard]] virtual hal::Error setDisplayContentSamplingEnabled(bool enabled,
                                                                      uint8_t componentMask,
                                                                      uint64_t maxFrames) const = 0;
    [[nodiscard]] virtual hal::Error getDisplayedContentSample(
            uint64_t maxFrames, uint64_t timestamp,
            android::DisplayedFrameStats* outStats) const = 0;
    [[nodiscard]] virtual hal::Error getReleaseFences(
            std::unordered_map<Layer*, android::sp<android::Fence>>* outFences) const = 0;
    [[nodiscard]] virtual hal::Error present(android::sp<android::Fence>* outPresentFence) = 0;
    [[nodiscard]] virtual hal::Error setClientTarget(
            uint32_t slot, const android::sp<android::GraphicBuffer>& target,
            const android::sp<android::Fence>& acquireFence, hal::Dataspace dataspace) = 0;
    [[nodiscard]] virtual hal::Error setColorMode(hal::ColorMode mode,
                                                  hal::RenderIntent renderIntent) = 0;
    [[nodiscard]] virtual hal::Error setColorTransform(const android::mat4& matrix) = 0;
    [[nodiscard]] virtual hal::Error setOutputBuffer(
            const android::sp<android::GraphicBuffer>& buffer,
            const android::sp<android::Fence>& releaseFence) = 0;
    [[nodiscard]] virtual hal::Error setPowerMode(hal::PowerMode mode) = 0;
    [[nodiscard]] virtual hal::Error setVsyncEnabled(hal::Vsync enabled) = 0;
    [[nodiscard]] virtual hal::Error validate(nsecs_t expectedPresentTime, uint32_t* outNumTypes,
                                              uint32_t* outNumRequests) = 0;
    [[nodiscard]] virtual hal::Error presentOrValidate(nsecs_t expectedPresentTime,
                                                       uint32_t* outNumTypes,
                                                       uint32_t* outNumRequests,
                                                       android::sp<android::Fence>* outPresentFence,
                                                       uint32_t* state) = 0;
    [[nodiscard]] virtual ftl::Future<hal::Error> setDisplayBrightness(
            float brightness, float brightnessNits,
            const Hwc2::Composer::DisplayBrightnessOptions& options) = 0;
    [[nodiscard]] virtual hal::Error setActiveConfigWithConstraints(
            hal::HWConfigId configId, const hal::VsyncPeriodChangeConstraints& constraints,
            hal::VsyncPeriodChangeTimeline* outTimeline) = 0;
    [[nodiscard]] virtual hal::Error setBootDisplayConfig(hal::HWConfigId configId) = 0;
    [[nodiscard]] virtual hal::Error clearBootDisplayConfig() = 0;
    [[nodiscard]] virtual hal::Error getPreferredBootDisplayConfig(
            hal::HWConfigId* configId) const = 0;
    [[nodiscard]] virtual hal::Error setAutoLowLatencyMode(bool on) = 0;
    [[nodiscard]] virtual hal::Error getSupportedContentTypes(
            std::vector<hal::ContentType>*) const = 0;
    [[nodiscard]] virtual hal::Error setContentType(hal::ContentType) = 0;
    [[nodiscard]] virtual hal::Error getClientTargetProperty(
            aidl::android::hardware::graphics::composer3::ClientTargetPropertyWithBrightness*
                    outClientTargetProperty) = 0;
    [[nodiscard]] virtual hal::Error getDisplayDecorationSupport(
            std::optional<aidl::android::hardware::graphics::common::DisplayDecorationSupport>*
                    support) = 0;
    [[nodiscard]] virtual hal::Error setIdleTimerEnabled(std::chrono::milliseconds timeout) = 0;
    [[nodiscard]] virtual hal::Error getPhysicalDisplayOrientation(
            Hwc2::AidlTransform* outTransform) const = 0;
};

class Layer {
public:
    virtual ~Layer();

    virtual hal::HWLayerId getId() const = 0;

    [[nodiscard]] virtual hal::Error setCursorPosition(int32_t x, int32_t y) = 0;
    [[nodiscard]] virtual hal::Error setBuffer(uint32_t slot,
                                               const android::sp<android::GraphicBuffer>& buffer,
                                               const android::sp<android::Fence>& acquireFence) = 0;
    [[nodiscard]] virtual hal::Error setSurfaceDamage(const android::Region& damage) = 0;

    [[nodiscard]] virtual hal::Error setBlendMode(hal::BlendMode mode) = 0;
    [[nodiscard]] virtual hal::Error setColor(
            aidl::android::hardware::graphics::composer3::Color color) = 0;
    [[nodiscard]] virtual hal::Error setCompositionType(
            aidl::android::hardware::graphics::composer3::Composition type) = 0;
    [[nodiscard]] virtual hal::Error setDataspace(hal::Dataspace dataspace) = 0;
    [[nodiscard]] virtual hal::Error setPerFrameMetadata(const int32_t supportedPerFrameMetadata,
                                                         const android::HdrMetadata& metadata) = 0;
    [[nodiscard]] virtual hal::Error setDisplayFrame(const android::Rect& frame) = 0;
    [[nodiscard]] virtual hal::Error setPlaneAlpha(float alpha) = 0;
    [[nodiscard]] virtual hal::Error setSidebandStream(const native_handle_t* stream) = 0;
    [[nodiscard]] virtual hal::Error setSourceCrop(const android::FloatRect& crop) = 0;
    [[nodiscard]] virtual hal::Error setTransform(hal::Transform transform) = 0;
    [[nodiscard]] virtual hal::Error setVisibleRegion(const android::Region& region) = 0;
    [[nodiscard]] virtual hal::Error setZOrder(uint32_t z) = 0;

    // Composer HAL 2.3
    [[nodiscard]] virtual hal::Error setColorTransform(const android::mat4& matrix) = 0;

    // Composer HAL 2.4
    [[nodiscard]] virtual hal::Error setLayerGenericMetadata(const std::string& name,
                                                             bool mandatory,
                                                             const std::vector<uint8_t>& value) = 0;

    // AIDL HAL
    [[nodiscard]] virtual hal::Error setBrightness(float brightness) = 0;
    [[nodiscard]] virtual hal::Error setBlockingRegion(const android::Region& region) = 0;
};
}
```

举例说明

```c++
//frameworks/native/services/surfaceflinger/DisplayHardware/HWC2.h
[[nodiscard]] virtual hal::Error getChangedCompositionTypes(
            std::unordered_map<Layer*, aidl::android::hardware::graphics::composer3::Composition>*
                    outTypes) = 0;
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
error = hwcDisplay->getChangedCompositionTypes(&changedTypes);
```

可以发现这里的调用都是带有返回值的。

# 3总结

在上面的例子中，函数`calculateSum`被标记为[[nodiscard]]，但在`main`函数中，它的返回值没有被使用。编译器可能会发出警告或错误，提醒开发者注意。

需要注意的是，[[nodiscard]]属性只是一种编译器提供的静态检查机制，并不能强制要求开发者一定要使用函数的返回值。但通过标记函数为[[nodiscard]]，可以增加代码的可读性和可维护性，减少潜在的错误。

# 参考

[[1] qq_38617319, nodiscard介绍 C++, 2021.](https://blog.csdn.net/qq_38617319/article/details/115099855)

