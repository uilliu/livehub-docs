# MakerNotes替代方案：XMP自定义命名空间调研报告

> 调研日期：2026-05-16
> 调研目标：深入分析XMP自定义命名空间作为MakerNotes写入受限的替代方案
> 关联问题：HarmonyOS API不支持MakerNotes写入，需寻找可行的元数据写入替代方案

---

## 1. XMP命名空间基础

### 1.1 什么是XMP命名空间

XMP（Extensible Metadata Platform，可扩展元数据平台）是Adobe制定的ISO标准（ISO 16684），用于在数字文件中嵌入元数据。XMP采用XML命名空间机制来组织不同类型的元数据：

**命名空间结构**：
```
xmlns:prefix="namespaceURI"
```

- **namespaceURI**：唯一标识符（通常为URL格式，如 `http://ns.adobe.com/xap/1.0/`）
- **prefix**：简短别名，用于在XMP数据中引用该命名空间（如 `xmp:`）

**标准XMP命名空间示例**：

| 前缀 | 命名空间URI | 用途 |
|-----|------------|------|
| `xmp` | `http://ns.adobe.com/xap/1.0/` | 基础XMP属性 |
| `xmpDM` | `http://ns.adobe.com/xmp/1.0/DynamicMedia/` | 动态媒体 |
| `dc` | `http://purl.org/dc/elements/1.1/` | Dublin Core |
| `photoshop` | `http://ns.adobe.com/photoshop/1.0/` | Photoshop特定 |
| `tiff` | `http://ns.adobe.com/tiff/1.0/` | TIFF属性 |
| `exif` | `http://ns.adobe.com/exif/1.0/` | EXIF属性 |

### 1.2 如何定义自定义命名空间

**核心原理**：
1. 选择一个由你控制的唯一URI（通常使用公司域名）
2. 定义简短有意义的前缀
3. 使用Adobe XMP Toolkit注册该命名空间
4. 在该命名空间下定义属性结构

**命名空间URI设计原则**：
- 使用稳定、可解析的URL格式
- 包含版本号便于未来升级（如 `/1.0/`）
- 使用你拥有控制的域名确保唯一性

**示例：自定义LiveHub命名空间**：
```
xmlns:livehub="http://livehub.app/ns/metadata/1.0/"
```

---

## 2. 各厂商XMP命名空间

### 2.1 Google Camera (GCamera)

**命名空间信息**：
- **前缀**：`GCamera`
- **URI**：`http://ns.google.com/photos/1.0/camera/`

**主要字段**（来自ExifTool文档）：

| 标签名 | 类型 | 说明 |
|-------|------|------|
| `GCamera:GCameraVersion` | String | Google Camera版本 |
| `GCamera:MotionPhoto` | Boolean | 是否为动态照片 |
| `GCamera:MotionPhotoVersion` | Integer | 动态照片格式版本 |
| `GCamera:MotionPhotoPresentationTimestampUs` | Long | 动态照片展示时间戳（微秒） |

**用途**：
- Google Pixel设备拍摄的Motion Photo（动态照片）
- Samsung设备也使用类似命名空间存储动态照片信息

### 2.2 Samsung (SamsungCam)

**命名空间信息**：
- **前缀**：`Samsung` 或 `samsung`
- **URI**：ExifTool记录为 `XMP-samsung` 前缀

**主要字段**：

| 标签名 | 类型 | 说明 |
|-------|------|------|
| `Samsung:SamsungCam` | 结构体 | Samsung相机信息容器 |
| 相关动态照片字段 | - | 与GCamera类似结构 |

**兼容性**：
- Samsung动态照片使用Google Motion Photo兼容格式
- XMP字段与GCamera命名空间有重叠

### 2.3 Xiaomi (XMP-micro)

**命名空间信息**（来自ExifTool `XMP-micro.html`）：
- **前缀**：`micro` 或 `XMP-micro`
- **URI**：可能为 `http://ns.xiaomi.com/photos/1.0/micro/`（待实测确认）

**ExifTool支持情况**：
- ExifTool已定义 `XMP-micro` 标签组
- 具体字段需通过小米设备实测获取

**推测用途**：
- 小米"动态照片"功能元数据
- 可能包含AI场景识别、相机设置等小米特定数据

**⚠️ 注意**：小米官方未公开命名空间URI定义，需要：
1. 使用小米手机拍摄动态照片
2. 用ExifTool提取完整XMP字段
3. 确认命名空间URI和字段结构

### 2.4 Huawei

**命名空间信息**：
- **前缀**：`Huawei` 或 `HUAWEI`
- **URI**：待确认（ExifTool有 `XMP-huawei` 标签组）

**已知元数据形式**：
- 华为实况照片主要使用**MakerNotes**存储（而非XMP）
- MakerNotes位于EXIF结构的厂商扩展区
- 标签名：`LivePhoto`、`VideoFileName`等

**XMP使用情况**：
- 华为可能同时使用XMP存储部分元数据
- 主要元数据载体仍为MakerNotes

**⚠️ 关键限制**：
- MakerNotes写入需要特定的厂商SDK或特殊API
- HarmonyOS当前API不支持MakerNotes写入
- 这是本调研的核心问题所在

### 2.5 其他厂商命名空间

| 厂商 | 前缀 | 说明 |
|-----|------|------|
| Apple | `apple-fi` | Apple FaceInfo（面部信息） |
| Canon | `Canon` | Canon相机特定元数据 |
| Nikon | `Nikon` | Nikon相机元数据 |
| Sony | `Sony` | Sony相机元数据 |

---

## 3. Adobe XMP Toolkit集成

### 3.1 库介绍和BSD协议

**Adobe XMP Toolkit SDK**：
- **官方地址**：https://github.com/adobe/XMP-Toolkit-SDK
- **语言**：C++
- **协议**：BSD 3-Clause License（"New BSD License"）

**BSD 3-Clause协议要点**：

| 权限 | 要求 | 限制 |
|-----|------|------|
| ✅ 商业使用 | 包含版权声明和许可证 | ❌ 不得用Adobe名义推广产品 |
| ✅ 修改分发 | 提供源代码中的许可证副本 | ❌ 未经许可不得使用Adobe商标 |
| ✅ 闭源集成 | 二进制分发时包含许可证 | - |
| ✅ 无需付费 | - | - |

**对商业应用的影响**：
- **完全可行**：BSD协议是商业友好型协议
- **无开源义务**：不需要公开你的应用源代码
- **唯一要求**：在产品文档或License文件中声明使用了Adobe XMP Toolkit

### 3.2 HarmonyOS Native集成方法

**集成架构**：

```
HarmonyOS App (ArkTS)
    └── Native Module (NAPI)
        └── Adobe XMP Toolkit (C++)
            └── XMP读写操作
```

**CMakeLists配置示例**：

```cmake
# CMakeLists.txt for HarmonyOS Native + XMP Toolkit

cmake_minimum_required(VERSION 3.5.0)
project(LiveHubNative)

set(NATIVERENDER_ROOT_PATH ${CMAKE_CURRENT_SOURCE_DIR})

# XMP Toolkit源码目录（假设放置在 third_party/xmp-toolkit）
set(XMP_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/third_party/xmp-toolkit)

include_directories(
    ${NATIVERENDER_ROOT_PATH}
    ${XMP_ROOT}/public/include
    ${XMP_ROOT}/source
)

# 编译XMP Toolkit静态库
add_library(xmp_static STATIC
    ${XMP_ROOT}/source/XMPMeta.cpp
    ${XMP_ROOT}/source/XMPUtils.cpp
    ${XMP_ROOT}/source/XMPFiles.cpp
    # ... 其他XMP源文件
)

# 主Native模块
add_library(entry SHARED
    napi_init.cpp
    xmp_handler.cpp
    metadata_writer.cpp
)

# 链接HarmonyOS NDK库 + XMP静态库
target_link_libraries(entry PUBLIC
    libace_napi.z.so
    libhilog_ndk.z.so
    libohimage.so
    xmp_static
)
```

**集成步骤**：

1. **下载XMP Toolkit SDK**
   - 从Adobe官方仓库获取：`github.com/adobe/XMP-Toolkit-SDK`
   - 复制到项目 `third_party/xmp-toolkit` 目录

2. **适配HarmonyOS编译**
   - 修改XMP Toolkit的构建配置适配OHOS工具链
   - 可能需要处理平台特定的文件操作API差异

3. **创建NAPI接口**
   - 封装XMP操作为ArkTS可调用的Native API
   - 处理异步操作（XMP文件操作可能耗时）

4. **处理依赖**
   - XMP Toolkit依赖Expat XML解析器（MIT协议）
   - 需一并编译集成

### 3.3 代码示例

**注册自定义命名空间**：

```cpp
#include "XMPMeta.hpp"
#include "XMP.hpp"

// 注册LiveHub自定义命名空间
void RegisterLiveHubNamespace() {
    std::string registeredPrefix;
    
    SXMPMeta::RegisterNamespace(
        "http://livehub.app/ns/metadata/1.0/",  // 命名空间URI
        "livehub",                               // 建议前缀
        &registeredPrefix                        // 返回实际注册的前缀
    );
    
    // registeredPrefix == "livehub:" (冒号自动添加)
}

// 写入自定义元数据
void WriteCustomMetadata(const std::string& filePath) {
    SXMPFiles xmpFile;
    
    // 打开文件
    xmpFile.OpenFile(
        filePath,
        kXMP_UnknownFile,
        kXMPFiles_OpenForUpdate | kXMPFiles_OpenUseSmartHandler
    );
    
    SXMPMeta meta;
    xmpFile.GetXMP(&meta);
    
    // 确保命名空间已注册
    RegisterLiveHubNamespace();
    
    // 设置自定义属性
    meta.SetProperty(
        "http://livehub.app/ns/metadata/1.0/",
        "LivePhotoType",
        "xiaomi"
    );
    
    meta.SetProperty_Int(
        "http://livehub.app/ns/metadata/1.0/",
        "VideoDuration",
        3000  // 毫秒
    );
    
    meta.SetProperty(
        "http://livehub.app/ns/metadata/1.0/",
        "VideoFileName",
        "IMG_20260516_123456.mp4"
    );
    
    // 验证可写入后提交
    if (xmpFile.CanPutXMP(meta)) {
        xmpFile.PutXMP(meta);
        xmpFile.CloseFile(kXMPFiles_UpdateSafely);  // 安全写入
    } else {
        xmpFile.CloseFile();  // 取消写入
    }
}

// 读取自定义元数据
void ReadCustomMetadata(const std::string& filePath) {
    SXMPFiles xmpFile;
    xmpFile.OpenFile(filePath, kXMP_UnknownFile);
    
    SXMPMeta meta;
    xmpFile.GetXMP(&meta);
    
    std::string livePhotoType;
    bool exists = meta.GetProperty(
        "http://livehub.app/ns/metadata/1.0/",
        "LivePhotoType",
        &livePhotoType,
        nullptr
    );
    
    if (exists) {
        // 处理读取到的数据
    }
    
    xmpFile.CloseFile();
}
```

**NAPI封装示例**（供ArkTS调用）：

```cpp
// napi_xmp_writer.cpp
#include <napi/native_api.h>
#include "XMPMeta.hpp"

static napi_value WriteLiveHubMetadata(napi_env env, napi_callback_info info) {
    size_t argc = 2;
    napi_value args[2];
    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);
    
    // 获取文件路径
    char filePath[256];
    napi_get_value_string_utf8(env, args[0], filePath, 256, nullptr);
    
    // 获取元数据对象
    napi_value metadataObj = args[1];
    
    // 执行XMP写入
    WriteCustomMetadata(filePath);
    
    // 返回成功状态
    napi_value result;
    napi_create_int32(env, 1, &result);
    return result;
}

// 导出模块
EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports) {
    napi_property_descriptor desc[] = {
        { "writeLiveHubMetadata", nullptr, WriteLiveHubMetadata, nullptr, nullptr, nullptr, napi_default, nullptr }
    };
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END

NAPI_MODULE(entry, Init)
```

---

## 4. 跨平台兼容性分析

### 4.1 iOS读取自定义XMP

**iOS原生支持**：
- iOS使用ImageIO框架读取图片元数据
- 支持读取XMP数据包，包括自定义命名空间

**Swift代码示例**：

```swift
import ImageIO
import CoreGraphics

func readCustomXMP(from imageURL: URL) -> [String: Any]? {
    guard let imageSource = CGImageSourceCreateWithURL(imageURL as CFURL, nil) else {
        return nil
    }
    
    // 获取完整元数据
    guard let metadata = CGImageSourceCopyPropertiesAtIndex(imageSource, 0, nil) as? [String: Any] else {
        return nil
    }
    
    // XMP数据位于特定字典键
    // iOS将XMP解析为字典结构
    if let xmpData = metadata["{XMP}"] as? [String: Any] {
        // 查找自定义命名空间数据
        // iOS可能将命名空间映射到特定字典键
        if let livehub = xmpData["livehub"] as? [String: Any] {
            return livehub
        }
    }
    
    // 或者直接获取原始XMP包
    if let rawXMP = CGImageSourceCopyMetadataAtIndex(imageSource, 0, nil) {
        // 使用CGImageMetadata处理
        // 可以查询任意命名空间的属性
    }
    
    return nil
}
```

**关键点**：
- iOS ImageIO能读取XMP，但自定义命名空间可能不被自动解析
- iOS可能对自定义命名空间的识别有限制
- 需要实测验证iOS能否读取LiveHub自定义命名空间

### 4.2 Android各厂商读取方式

**Android ExifInterface（API 25+）**：

```java
import android.media.ExifInterface;

// 读取XMP（AndroidX ExifInterface）
ExifInterface exif = new ExifInterface(imagePath);

// 获取原始XMP数据包
String xmpPacket = exif.getXmp();

// Android原生ExifInterface不直接支持解析自定义XMP命名空间
// 需要使用第三方库解析XMP XML
```

**Android局限性**：
- ExifInterface仅提供原始XMP字符串
- 不支持直接查询特定命名空间的属性
- 需配合XML解析器处理

**使用第三方库（推荐）**：

```java
// 使用metadata-extractor库
import com.drew.imaging.ImageMetadataReader;
import com.drew.metadata.Metadata;
import com.drew.metadata.xmp.XmpDirectory;

Metadata metadata = ImageMetadataReader.readMetadata(imageFile);
XmpDirectory xmpDir = metadata.getFirstDirectoryOfType(XmpDirectory.class);

// 获取XMP属性
String livePhotoType = xmpDir.getString("livehub:LivePhotoType");
```

**各厂商兼容性矩阵**：

| 平台/厂商 | 自定义XMP读取 | 自定义XMP写入 | 备注 |
|----------|--------------|--------------|------|
| iOS (原生) | ⚠️ 需验证 | ❌ 不支持写入自定义 | 仅读取，写入受限 |
| Android (ExifInterface) | ✅ 可读取原始包 | ⚠️ setXmp()受限 | 需第三方库解析 |
| Android (metadata-extractor) | ✅ 支持解析 | ❌ 仅读取 | Drew Noakes库 |
| Android (Apache Commons Imaging) | ✅ 支持 | ✅ 可写入 | 纯Java方案 |
| HarmonyOS (Native) | ✅ 支持 | ✅ 通过XMP Toolkit | 需Native开发 |
| 小米相册APP | ⚠️ 待验证 | - | 是否识别自定义字段未知 |
| 华为相册APP | ⚠️ 待验证 | - | 主要依赖MakerNotes |

**⚠️ 关键验证需求**：
1. 验证小米/华为相册APP是否识别自定义XMP命名空间
2. 验证跨设备分享后，自定义元数据是否保留
3. 确认第三方相册APP（如Google Photos）对自定义XMP的处理

---

## 5. 推荐实现方案

### 5.1 自定义命名空间定义

**LiveHub命名空间设计**：

```
命名空间URI: http://livehub.app/ns/livephoto/1.0/
前缀建议: livehub
```

**字段结构设计**：

| 字段名 | 类型 | 必填 | 说明 |
|-------|------|-----|------|
| `SourceFormat` | String | ✅ | 原始格式：huawei/xiaomi/apple/google |
| `TargetFormat` | String | ✅ | 目标格式 |
| `VideoFileName` | String | ⚠️ | 关联视频文件名（双文件模式） |
| `VideoDurationMs` | Integer | ✅ | 视频时长（毫秒） |
| `ConversionTimestamp` | DateTime | ✅ | 转换时间戳 |
| `OriginalMakerNotes` | String | ⚠️ | 原始MakerNotes备份（如适用） |
| `VideoEmbedded` | Boolean | ✅ | 是否内嵌视频（Motion Photo模式） |
| `VideoOffset` | Integer | ⚠️ | 内嵌视频起始偏移（字节） |

**命名空间注册代码**：

```cpp
// livehub_namespace.h
namespace LiveHubXMP {
    const char* NS_URI = "http://livehub.app/ns/livephoto/1.0/";
    const char* NS_PREFIX = "livehub";
    
    // 字段定义
    struct Fields {
        static const char* SOURCE_FORMAT = "SourceFormat";
        static const char* TARGET_FORMAT = "TargetFormat";
        static const char* VIDEO_FILE_NAME = "VideoFileName";
        static const char* VIDEO_DURATION_MS = "VideoDurationMs";
        static const char* CONVERSION_TIMESTAMP = "ConversionTimestamp";
        static const char* ORIGINAL_MAKER_NOTES = "OriginalMakerNotes";
        static const char* VIDEO_EMBEDDED = "VideoEmbedded";
        static const char* VIDEO_OFFSET = "VideoOffset";
    };
}
```

### 5.2 写入流程

**完整写入流程**：

```
1. 注册命名空间
   └─> SXMPMeta::RegisterNamespace(NS_URI, NS_PREFIX)

2. 打开目标文件
   └─> SXMPFiles::OpenFile(path, format, OpenForUpdate)

3. 读取现有XMP
   └─> SXMPFiles::GetXMP(&meta)

4. 设置自定义属性
   └─> meta.SetProperty(NS_URI, fieldName, value)

5. 可选：设置标准XMP属性
   └─> meta.SetProperty(kXMP_NS_XMP, "CreateDate", timestamp)

6. 验证写入可行性
   └─> SXMPFiles::CanPutXMP(meta)

7. 提交更改
   └─> SXMPFiles::PutXMP(meta)

8. 安全关闭
   └─> SXMPFiles::CloseFile(kXMPFiles_UpdateSafely)
```

**HarmonyOS NAPI完整封装**：

```cpp
// livehub_xmp_writer.cpp
#include <napi/native_api.h>
#include <hilog/log.h>
#include "XMPMeta.hpp"
#include "XMPFiles.hpp"
#include "livehub_namespace.h"

class LiveHubXMPWriter {
public:
    static napi_value WriteMetadata(napi_env env, napi_callback_info info);
    static napi_value ReadMetadata(napi_env env, napi_callback_info info);
    
private:
    static void RegisterNamespace();
    static bool WriteToFile(const std::string& path, const LiveHubMetadata& data);
    static LiveHubMetadata ReadFromFile(const std::string& path);
};

struct LiveHubMetadata {
    std::string sourceFormat;
    std::string targetFormat;
    std::string videoFileName;
    int32_t videoDurationMs;
    std::string conversionTimestamp;
    bool videoEmbedded;
    int32_t videoOffset;
};

void LiveHubXMPWriter::RegisterNamespace() {
    std::string registeredPrefix;
    SXMPMeta::RegisterNamespace(
        LiveHubXMP::NS_URI,
        LiveHubXMP::NS_PREFIX,
        &registeredPrefix
    );
}

bool LiveHubXMPWriter::WriteToFile(const std::string& path, const LiveHubMetadata& data) {
    try {
        RegisterNamespace();
        
        SXMPFiles xmpFile;
        xmpFile.OpenFile(path, kXMP_UnknownFile, 
            kXMPFiles_OpenForUpdate | kXMPFiles_OpenUseSmartHandler);
        
        SXMPMeta meta;
        xmpFile.GetXMP(&meta);
        
        // 写入所有字段
        meta.SetProperty(LiveHubXMP::NS_URI, LiveHubXMP::Fields::SOURCE_FORMAT, 
            data.sourceFormat);
        meta.SetProperty(LiveHubXMP::NS_URI, LiveHubXMP::Fields::TARGET_FORMAT, 
            data.targetFormat);
        if (!data.videoFileName.empty()) {
            meta.SetProperty(LiveHubXMP::NS_URI, LiveHubXMP::Fields::VIDEO_FILE_NAME, 
                data.videoFileName);
        }
        meta.SetProperty_Int(LiveHubXMP::NS_URI, LiveHubXMP::Fields::VIDEO_DURATION_MS, 
            data.videoDurationMs);
        meta.SetProperty(LiveHubXMP::NS_URI, LiveHubXMP::Fields::CONVERSION_TIMESTAMP, 
            data.conversionTimestamp);
        meta.SetProperty_Bool(LiveHubXMP::NS_URI, LiveHubXMP::Fields::VIDEO_EMBEDDED, 
            data.videoEmbedded);
        if (data.videoEmbedded) {
            meta.SetProperty_Int(LiveHubXMP::NS_URI, LiveHubXMP::Fields::VIDEO_OFFSET, 
                data.videoOffset);
        }
        
        if (xmpFile.CanPutXMP(meta)) {
            xmpFile.PutXMP(meta);
            xmpFile.CloseFile(kXMPFiles_UpdateSafely);
            return true;
        }
        
        xmpFile.CloseFile();
        return false;
    } catch (const XMP_Error& e) {
        OH_LOG_Print(LOG_APP, LOG_ERROR, 0, "LiveHubXMP", "XMP Error: %{public}s", e.mErrMsg);
        return false;
    }
}
```

### 5.3 验证方法

**验证步骤**：

1. **写入验证**
   - 使用XMP Toolkit写入自定义字段
   - 用ExifTool检查输出：`exiftool -XMP:all image.jpg`

2. **读取验证**
   - 用XMP Toolkit读取写入的字段
   - 验证字段值完整性

3. **跨平台验证**
   - 将图片导入iOS设备，检查相册是否识别元数据
   - 将图片导入Android设备，用metadata-extractor验证
   - 分享到其他应用，检查元数据保留情况

4. **ExifTool验证命令**：

```bash
# 查看所有XMP字段
exiftool -XMP:all converted_photo.jpg

# 查看特定命名空间（如果ExifTool已定义）
exiftool -XMP-livehub:all converted_photo.jpg

# 导出完整XMP XML
exiftool -xmp -b converted_photo.jpg > xmp_dump.xml

# 验证自定义字段存在
exiftool -s -XMP:LivePhotoType converted_photo.jpg
```

---

## 6. 方案可行性评估

### 6.1 技术可行性

| 评估项 | 结论 | 备注 |
|-------|------|------|
| XMP Toolkit集成 | ✅ 可行 | BSD协议无商业风险 |
| HarmonyOS Native编译 | ⚠️ 需适配 | 平台API差异需处理 |
| 自定义命名空间定义 | ✅ 可行 | 需选择稳定URI |
| 写入JPEG | ✅ 支持 | XMP Toolkit原生支持 |
| 写入HEIF | ⚠️ 需验证 | 需测试HEIF格式支持 |
| 读取跨平台 | ⚠️ 需实测验证 | 各平台解析能力不同 |

### 6.2 相对MakerNotes的优劣势

**优势**：
- ✅ XMP是开放标准，任何工具都能读取
- ✅ 自定义命名空间完全可控
- ✅ 无厂商SDK依赖
- ✅ 跨平台兼容性理论上更好

**劣势**：
- ❌ 华为/小米原生相册可能不识别自定义XMP
- ❌ 需要Native开发投入
- ❌ 不兼容现有厂商的MakerNotes读取逻辑
- ❌ 元数据可能在某些场景被丢弃

### 6.3 与原需求的匹配度

**核心问题回顾**：
- HarmonyOS API不支持MakerNotes写入
- 华为实况照片依赖MakerNotes存储元数据

**方案匹配**：
- ✅ 可写入自定义元数据到XMP
- ⚠️ 但华为相册可能不识别
- ⚠️ 转换后照片可能无法被华为设备识别为实况照片

**建议**：
1. 将XMP方案作为**备用方案**
2. 优先尝试其他替代：
   - 华为官方SDK（如有）
   - 系统级API扩展申请
3. XMP方案用于"转换标记"，而非"实况照片功能激活"

---

## 7. 行动建议

### 7.1 立即行动项

| 优先级 | 行动项 | 方法 |
|-------|-------|------|
| P0 | 验证华为相册识别自定义XMP | 实测写入后导入华为相册 |
| P0 | 验证小米相册识别自定义XMP | 实测写入后导入小米相册 |
| P1 | XMP Toolkit HarmonyOS编译适配 | 创建测试Native项目 |
| P1 | 确定最终命名空间URI | 考虑使用公开域名或申请标准化 |
| P2 | 设计完整的元数据字段表 | 与需求规格对齐 |

### 7.2 风险缓解

| 风险 | 缓解措施 |
|-----|---------|
| 相册不识别自定义XMP | 使用GCamera命名空间格式（业界通用） |
| XMP Toolkit编译失败 | 寻找Android移植版本作为参考 |
| 元数据被丢弃 | 同时写入标准EXIF字段作为备份 |
| 转换后照片不被识别 | 标明为"已转换"而非原生实况照片 |

---

## 参考资料

- [Adobe XMP Toolkit SDK GitHub](https://github.com/adobe/XMP-Toolkit-SDK)
- [ExifTool XMP Tags Documentation](https://exiftool.org/TagNames/XMP.html)
- [ExifTool XMP-micro Tags (Xiaomi)](https://exiftool.org/TagNames/XMP-micro.html)
- [ExifTool GCamera Tags](https://exiftool.org/TagNames/XMP.html#GCamera)
- [HarmonyOS Native Development Guide](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ndk-development)
- [ISO 16684-1:2019 XMP Specification](https://www.iso.org/standard/75297.html)

---

*调研完成于 2026-05-16*
*下一步：实测验证各厂商相册对自定义XMP的识别能力*