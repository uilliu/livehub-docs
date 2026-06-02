# LiveHub MVP 架构设计文档

> 版本：2.0
> 设计日期：2026-05-22
> 设计方案：方案 C - 全格式 MVP（最完整）
> 基于调研报告：001-031 全系列调研报告

---

## 一、设计概述

### 1.1 MVP 目标

**核心目标**：支持所有5种动态照片格式的双向转换

| 格式 | 中文名称 | 英文名称 | 代码标识 | 文件模式 | MVP 输入 | MVP 输出 |
|------|---------|---------|---------|---------|---------|---------|
| **Apple** | 实况照片 | Live Photo | `APPLE_LIVE_PHOTO` | 分离文件 | ✅ P0 | ✅ P0 |
| **华为(单框架)** | 动态照片 | Moving Photo | `HUAWEI_MOVING_PHOTO_SINGLE` | 分离文件 | ✅ P0 | ✅ API支持 |
| **华为(双框架)** | 动态照片 | Moving Photo | `HUAWEI_MOVING_PHOTO_DUAL` | 嵌入文件 | ✅ P0 | ⚠️ 末尾元数据 |
| **小米** | 实况照片 | Live Photo | `XIAOMI_LIVE_PHOTO` | 分离/嵌入 | ✅ P0 | ⚠️ 嵌入文件 |
| **Google** | 动态照片 | Motion Photo | `GOOGLE_MOTION_PHOTO` | 嵌入文件 | ✅ P0 | ✅ P0 |
| **Samsung** | 动态照片 | Motion Photo | `SAMSUNG_MOTION_PHOTO` | 嵌入文件 | ✅ P1 | ✅ P1 |

### 1.2 关键调研发现（2026-05-19 更新）

#### 华为单框架/双框架差异（024）

| 系统版本 | 框架类型 | 存储方式 | 存储路径 |
|---------|---------|---------|---------|
| **EMUI** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS 4 及以前** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS NEXT** | 单框架 | **分离文件模式** | `/storage/media/100/local/files/Photo/` |

#### 华为双框架末尾元数据格式（022）

```
双框架动态照片文件结构（单一JPG文件）：
┌────────────────────────────────────────────────────────────┐
│ [0, m-n-40)        │ JPEG图片数据（封面帧）                 │
│ [m-n-40, m-p-60)   │ MP4视频数据                            │
│ [m-p-60, m-p-40)   │ CinemagraphInfo（可选，4字节长度标记）  │
│ [m-p-40, m-40)     │ Version & Frame Num（20字节）          │
│ [m-40, m-20)       │ Sight tremble metadata（20字节）       │
│ [m-20, m)          │ Video info metadata（20字节）          │
└────────────────────────────────────────────────────────────┘

末尾元数据详细规格：
├── Video info metadata（末尾20字节）：LIVE_xxxx
├── Sight tremble metadata（倒数20~40字节）：xx:xx（微动瞬间时间范围）
├── Version & Frame Num（倒数40~60字节）：v3_f31_c
└── CinemagraphInfo（可选）：仅当版本号以_c结尾时存在
```

#### Native P0 验证结果（027）

| 验证项 | 结论 | 可行性 |
|--------|------|--------|
| Native层合并JPG+MP4为单一文件 | ⚠️ **部分可行** | 中等 |
| Adobe XMP Toolkit编译可行性 | ✅ **可行** | 高（2-3周工作量） |
| 合并文件写入XMP元数据 | ✅ **可行** | 高 |
| **合并文件保存到媒体库** | ❌ **不可行** | **API不支持嵌入文件模式** |

**关键限制**：
- HarmonyOS PhotoAccessHelper API 仅支持**分离文件模式**
- 嵌入文件模式（小米、Google、华为双框架）无法通过 API 保存到媒体库
- 必须使用导出/分享功能作为替代方案

#### 华为小米兼容性（025）

| 原始格式 | 华为相册 | 小米相册 | Google Photos |
|---------|---------|---------|--------------|
| **华为双框架嵌入** | ✅ 全功能 | ⚠️ 仅静态 | ⚠️ 仅静态 |
| **华为单框架分离** | ⚠️ 仅静态 | ⚠️ 仅静态 | ⚠️ 仅静态 |
| **小米嵌入** | ⚠️ 仅静态 | ✅ 全功能 | ✅ 有限支持 |
| **Google Motion Photos** | ⚠️ 仅静态 | ✅ 有限支持 | ✅ 全功能 |

**结论**：华为、小米不识别对方格式，必须实现目标设备格式输出策略。

---

## 二、架构设计

### 2.1 总体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LiveHub MVP 架构 v2.0                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      HarmonyOS 应用层 (ArkTS)                         │   │
│   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │
│   │  │  UI 界面    │ │  文件管理   │ │  任务调度   │ │  设置管理   │    │   │
│   │  │  (ArkUI)    │ │(PhotoAccess)│ │(Background) │ │(Preferences)│    │   │
│   │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘    │   │
│   │         │               │               │               │            │   │
│   │         └───────────────┴───────────────┴───────────────┘            │   │
│   │                                    │                                 │   │
│   │                         ┌──────────▼──────────┐                      │   │
│   │                         │   ConversionService │                      │   │
│   │                         │   (转换业务逻辑)    │                      │   │
│   │                         └──────────┬──────────┘                      │   │
│   └─────────────────────────────────────┼───────────────────────────────┘   │
│                                         │                                    │
│                                         │ NAPI 调用                          │
│                                         │                                    │
│   ┌─────────────────────────────────────▼───────────────────────────────┐   │
│   │                      Native C++ 层 (livehub-core)                     │   │
│   │  ┌───────────────────────────────────────────────────────────────┐  │   │
│   │  │                    NAPI 模块 (livehub_native)                  │  │   │
│   │  └───────────────────────────────────────────────────────────────┘  │   │
│   │                                │                                     │   │
│   │  ┌──────────────┬──────────────┼──────────────┬──────────────┐     │   │
│   │  │              │              │              │              │     │   │
│   │  │ ┌──────────▼─▼──────────┐ ┌─▼──────────┐ ┌─▼──────────┐ ┌─▼───┐ │   │
│   │  │ │ FormatDetector        │ │Converter   │ │ Metadata   │ │FFmpeg│ │   │
│   │  │ │ 格式检测引擎           │ │转换引擎    │ │元数据引擎  │ │Remux │ │   │
│   │  │ │                        │ │            │ │            │ │      │ │   │
│   │  │ │ ✅ 华为双框架末尾检测  │ │ ✅ 华为双框架│ │ ✅ XMP    │ │ ✅MOV │ │   │
│   │  │ │ ✅ 华为单框架API检测  │ │ ✅ 华为单框架│ │ ✅ 末尾元数据│ │转换 │ │   │
│   │  │ └───────────────────────┘ └────────────┘ └────────────┘ └─────┘ │   │
│   │  │                                                                 │   │
│   │  │ ┌─────────────────────────────────────────────────────────────┐ │   │
│   │  │ │                  第三方库集成                                 │ │   │
│   │  │ │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │ │   │
│   │  │ │  │ FFmpeg   │ │ Adobe XMP │ │ libheif │ │ HarmonyOS API   │ │ │   │
│   │  │ │  │ (LGPL)   │ │ Toolkit   │ │ (可选)  │ │ MovingPhoto     │ │ │   │
│   │  │ │  └──────────┘ │ (BSD)    │ └──────────┘ │ (分离文件)      │ │ │   │
│   │  │ │               └──────────┘              └──────────────────┘ │ │   │
│   │  │ └─────────────────────────────────────────────────────────────┘ │   │
│   │  └──────────────────────────────────────────────────────────────────┘   │
│   └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐   │
│   │                      输出模式选择策略                                   │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐  │   │
│   │  │  目标设备检测 → 选择输出格式                                      │  │   │
│   │  │  ├─ 华为单框架 → 分离文件模式（API支持）                          │  │   │
│   │  │  ├─ 华为双框架 → 嵌入文件+末尾元数据（导出功能）                  │  │   │
│   │  │  ├─ 小米 → 嵌入文件+XMP-GCamera（导出功能）                      │  │   │
│   │  │  ├─ Apple → 分离文件+MOV（API支持）                              │  │   │
│   │  │  └─ 其他 → 嵌入文件+GCamera（导出功能）                          │  │   │
│   │  └─────────────────────────────────────────────────────────────────┘  │   │
│   └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 FormatDetector（格式检测引擎）

**检测流程**：

```
检测流程（依次检查，首次匹配即返回）：

1. Apple Live Photo 检测
   ├─ 输入为文件配对（HEIC + MOV）
   ├─ EXIF ContentIdentifier 存在
   ├─ MOV QuickTime ContentIdentifier 存在
   ├─ 两ContentIdentifier匹配
   └─ → 返回 APPLE_LIVE_PHOTO

2. 华为单框架 Moving Photo 检测
   ├─ HarmonyOS MovingPhoto API检测（首选）
   │  └─ PhotoKeys.PHOTO_SUBTYPE == MOVING_PHOTO (subtype=3)
   ├─ 或检查同名MP4文件存在
   ├─ 或检查数据库字段 subtype=3
   └─ → 返回 HUAWEI_MOVING_PHOTO_SINGLE

3. 华为双框架 Moving Photo 检测（嵌入文件）
   ├─ 读取文件末尾20字节
   ├─ 检查是否以"LIVE_"开头
   ├─ 解析视频长度：xxxx
   ├─ 检查Version & Frame Num（倒数40~60字节）
   │  └─ 格式：v3_f31_c（版本号+封面帧号+可选_c）
   ├─ 检查是否含_c（支持CinemagraphInfo）
   └─ → 返回 HUAWEI_MOVING_PHOTO_DUAL

4. 小米 Live Photo 检测
   ├─ 嵌入文件模式：
   │  ├─ 文件名以 MVIMG_ 开头
   │  ├─ XMP GCamera:MotionPhoto = 1
   │  ├─ 存在 GContainer:Directory 或 MotionPhotoOffset
   │  └─ → 返回 XIAOMI_LIVE_PHOTO_EMB
   ├─ 分离文件模式：
   │  ├─ JPG + 存在同名MP4文件
   │  └─ 时间戳一致性验证
   │  └─ → 返回 XIAOMI_LIVE_PHOTO_SEP

5. Google Motion Photo 检测（Android 1.0规范）
   ├─ JPEG/HEIC 文件
   ├─ XMP Camera:MotionPhoto = 1 存在
   ├─ 存在 GContainer:Directory（新规范）
   │  └─ 使用 ItemLength 定位视频
   ├─ 或存在 Camera:MotionPhotoOffset（兼容旧设备）
   └─ → 返回 GOOGLE_MOTION_PHOTO

6. Samsung Motion Photo 检测
   ├─ JPEG 文件（扩展名可能为 .mvimg）
   ├─ 文件名以 MVIMG_ 开头
   ├─ XMP Micro:MicroVideoOffset 存在
   ├─ 或 XMP GCamera:MotionPhoto = 1
   └─ → 返回 SAMSUNG_MOTION_PHOTO

7. 普通静态图片
   └─ 无上述任何标记 → 返回 STATIC_IMAGE
```

**华为双框架末尾元数据解析示例**：

```cpp
// 华为双框架末尾元数据解析
struct HuaweiDualMetadata {
    uint32_t video_length;      // 从LIVE_xxxx解析
    uint16_t sight_start_ms;    // 微动瞬间开始时间
    uint16_t sight_end_ms;      // 微动瞬间结束时间
    uint8_t version;            // 版本号（v1-v7）
    uint16_t cover_frame_num;   // 封面帧号
    bool has_cinemagraph;       // 是否含CinemagraphInfo
    uint32_t cinema_length;     // CinemagraphInfo长度
};

HuaweiDualMetadata parse_huawei_dual_metadata(const char* file_path) {
    FILE* f = fopen(file_path, "rb");
    fseek(f, 0, SEEK_END);
    long m = ftell(f);
    
    // 读取末尾20字节：Video info metadata
    char video_info[20];
    fseek(f, m - 20, SEEK_SET);
    fread(video_info, 1, 20, f);
    // 格式：LIVE_xxxx
    
    // 读取倒数20~40字节：Sight tremble metadata
    char sight_tremble[20];
    fseek(f, m - 40, SEEK_SET);
    fread(sight_tremble, 1, 20, f);
    // 格式：xx:xx
    
    // 读取倒数40~60字节：Version & Frame Num
    char version_frame[20];
    fseek(f, m - 60, SEEK_SET);
    fread(version_frame, 1, 20, f);
    // 格式：v3_f31_c
    
    fclose(f);
    
    // 解析...
}
```

### 2.3 Converter（转换引擎）

**转换矩阵（基于P0验证结果更新）**：

| 输入→输出 | Apple | 华为单框架 | 华为双框架 | 小米 | Google | Samsung |
|----------|-------|----------|-----------|------|--------|---------|
| **Apple** | — | ✅ API | ⚠️ 导出 | ⚠️ 导出 | ✅ 导出 | ✅ 导出 |
| **华为单框架** | ✅ API | — | ⚠️ 导出 | ⚠️ 导出 | ✅ 导出 | ✅ 导出 |
| **华为双框架** | ✅ API+提取 | ✅ API | — | ⚠️ 导出 | ✅ 导出 | ✅ 导出 |
| **小米嵌入** | ✅ API+提取 | ✅ API | ⚠️ 导出 | — | ✅ 导出 | ✅ 导出 |
| **小米分离** | ✅ API | ✅ API | ⚠️ 导出 | — | ✅ 导出 | ✅ 导出 |
| **Google** | ✅ API+提取 | ✅ API | ⚠️ 导出 | ⚠️ 导出 | — | ✅ 导出 |
| **Samsung** | ✅ API+提取 | ✅ API | ⚠️ 导出 | ⚠️ 导出 | ✅ 导出 | — |

**API支持 vs 导出功能说明**：
- ✅ **API支持**：可通过 HarmonyOS PhotoAccessHelper API 保存到媒体库
- ⚠️ **导出功能**：需保存到应用沙箱，通过分享/导出功能提供给用户

**华为双框架 → Apple 转换流程**：

```cpp
// 华为双框架嵌入格式 → Apple Live Photo
int convert_huawei_dual_to_apple(const char* huawei_file, 
                                  const char* output_dir) {
    // 1. 解析华为双框架末尾元数据
    HuaweiDualMetadata meta = parse_huawei_dual_metadata(huawei_file);
    
    // 2. 提取嵌入视频数据
    long file_size = get_file_size(huawei_file);
    long video_start = file_size - meta.video_length - 40;
    long video_end = file_size - 60 - (meta.has_cinemagraph ? meta.cinema_length + 4 : 0);
    
    // 提取图片数据 [0, video_start)
    // 提取视频数据 [video_start, video_end)
    
    // 3. FFmpeg MP4 → MOV转换
    // ffmpeg -i extracted.mp4 -c copy -tag:v hvc1 output.mov
    
    // 4. 生成ContentIdentifier UUID
    
    // 5. 写入元数据
    
    // 6. ArkTS层创建动态照片（API支持）
}
```

**华为双框架 → 小米 转换流程**：

```cpp
// 华为双框架嵌入格式 → 小米嵌入格式
int convert_huawei_dual_to_xiaomi(const char* huawei_file,
                                   const char* output_file) {
    // 1. 解析华为双框架末尾元数据
    HuaweiDualMetadata meta = parse_huawei_dual_metadata(huawei_file);
    
    // 2. 提取嵌入视频数据
    
    // 3. 计算小米格式的视频偏移量
    // video_offset = video_length（从文件末尾倒数）
    
    // 4. 写入XMP-GCamera元数据
    // GCamera:MotionPhoto = 1
    // GCamera:MotionPhotoOffset = video_offset
    
    // 5. 输出为单一JPG文件
    // ⚠️ 注意：无法保存到媒体库，需通过导出功能
}
```

**小米嵌入 → 华为双框架 转换流程**：

```cpp
// 小米嵌入格式 → 华为双框架嵌入格式
int convert_xiaomi_to_huawei_dual(const char* xiaomi_file,
                                   const char* output_file) {
    // 1. 解析小米XMP-GCamera元数据
    // GCamera:MotionPhotoOffset
    
    // 2. 提取嵌入视频数据
    
    // 3. 构造华为双框架末尾元数据
    char video_info[20] = "LIVE_1234";  // 视频长度+40
    char sight_tremble[20] = "0:0";     // 无微动瞬间
    char version_frame[20] = "v6_f0";   // 单框架来源版本
    
    // 4. 组合文件
    // [JPEG数据] + [视频数据] + [末尾60字节]
    
    // 5. 输出为单一JPG文件
    // ⚠️ 注意：无法保存到媒体库，需通过导出功能
}
```

---

## 三、华为格式输出方案（重大更新）

### 3.1 华为单框架输出（API支持）

**使用 HarmonyOS PhotoAccessHelper API**：

```typescript
// ArkTS层创建华为单框架动态照片
import { photoAccessHelper } from '@kit.MediaLibraryKit';

async function createHuaweiSingleMovingPhoto(
  imageUri: string, 
  videoUri: string
): Promise<string> {
  let context = getContext(this) as common.UIAbilityContext;
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
  
  // 创建动态照片资产（分离文件模式）
  let changeRequest = photoAccessHelper.MediaAssetChangeRequest.createAssetRequest(
    context,
    photoAccessHelper.PhotoType.IMAGE,
    'jpg',
    { subtype: photoAccessHelper.PhotoSubtype.MOVING_PHOTO }  // subtype=3
  );
  
  // 添加图片资源
  changeRequest.addResource(
    photoAccessHelper.ResourceType.IMAGE_RESOURCE,
    imageUri
  );
  
  // 添加视频资源
  changeRequest.addResource(
    photoAccessHelper.ResourceType.VIDEO_RESOURCE,
    videoUri
  );
  
  // 应用变更
  await phAccessHelper.applyChanges(changeRequest);
  
  return changeRequest.getAssetUri();
}
```

**关键发现**：
- ✅ HarmonyOS API **完全支持**创建分离文件模式动态照片
- ✅ 可直接保存到媒体库
- ✅ 华为单框架设备完整识别

### 3.2 华为双框架输出（导出功能）

**Native层构造末尾元数据**：

```cpp
// Native层创建华为双框架嵌入格式
int create_huawei_dual_format(
    const char* image_data, 
    size_t image_size,
    const char* video_data,
    size_t video_size,
    const char* output_path
) {
    // 构造末尾元数据
    char sight_tremble[20] = "0:0";
    char version_frame[20] = "v6_f0";  // 单框架来源，无_c
    
    uint32_t video_length = video_size + 40;  // +末尾40字节
    char video_info[20];
    snprintf(video_info, 20, "LIVE_%04d", video_length);
    
    // 组合文件
    FILE* f = fopen(output_path, "wb");
    
    // 写入JPEG数据
    fwrite(image_data, 1, image_size, f);
    
    // 写入视频数据
    fwrite(video_data, 1, video_size, f);
    
    // 写入末尾元数据（共60字节）
    fwrite(sight_tremble, 1, 20, f);     // 倒数20~40字节
    fwrite(version_frame, 1, 20, f);     // 倒数40~60字节
    fwrite(video_info, 1, 20, f);        // 末尾20字节
    
    fclose(f);
    
    return 0;
}
```

**关键限制**：
- ❌ 无法通过 API 保存嵌入文件格式到媒体库
- ⚠️ 必须使用导出/分享功能
- ⚠️ 华为双框架设备可识别，单框架设备需自动拆分

### 3.3 CinemagraphInfo处理（可选）

**当源格式含微动瞬间效果数据时**：

```cpp
// CinemagraphInfo数据结构
struct CinemagraphInfo {
    uint32_t length;           // 前4字节长度标记
    char data[90 * 60];        // 90帧微动瞬间数据（典型60字节）
};

// 检测是否含CinemagraphInfo
bool has_cinemagraph(const char* version_frame) {
    return strstr(version_frame, "_c") != NULL;
}

// 转换时保留CinemagraphInfo
int preserve_cinemagraph_info(
    const CinemagraphInfo* cinema,
    const char* video_data,
    size_t video_size,
    FILE* output
) {
    // 写入视频数据
    fwrite(video_data, 1, video_size, f);
    
    // 写入CinemagraphInfo（若有）
    if (cinema) {
        uint32_t length_bytes = cinema->length;
        fwrite(&length_bytes, 4, 1, f);  // 前4字节长度标记
        fwrite(cinema->data, cinema->length, 1, f);
    }
    
    return 0;
}
```

---

## 四、小米格式输出方案

### 4.1 小米嵌入格式输出（导出功能）

**使用 Adobe XMP Toolkit 写入 GCamera 命名空间**：

```cpp
// Native层创建小米嵌入格式（兼容Google Motion Photos）
#include "XMPMeta.hpp"
#include "XMPFiles.hpp"

int create_xiaomi_embedded_format(
    const char* image_path,
    const char* video_path,
    const char* output_path
) {
    // 1. 合并文件
    FILE* f_img = fopen(image_path, "rb");
    FILE* f_vid = fopen(video_path, "rb");
    FILE* f_out = fopen(output_path, "wb");
    
    // 复制JPEG数据
    // 复制MP4数据（追加到末尾）
    
    // 2. 计算视频偏移量
    size_t image_size = get_file_size(image_path);
    size_t video_size = get_file_size(video_path);
    int32_t video_offset = video_size;  // 从文件末尾倒数
    
    fclose(f_img);
    fclose(f_vid);
    fclose(f_out);
    
    // 3. 写入XMP-GCamera元数据
    try {
        SXMPMeta::RegisterNamespace(
            "http://ns.google.com/photos/1.0/camera/",
            "GCamera"
        );
        
        SXMPFiles xmpFile;
        xmpFile.OpenFile(output_path, kXMP_UnknownFile,
            kXMPFiles_OpenForUpdate | kXMPFiles_OpenUseSmartHandler);
        
        SXMPMeta meta;
        xmpFile.GetXMP(&meta);
        
        meta.SetProperty_Bool("http://ns.google.com/photos/1.0/camera/",
            "MotionPhoto", true);
        meta.SetProperty_Int("http://ns.google.com/photos/1.0/camera/",
            "MotionPhotoVersion", 1);
        meta.SetProperty_Int("http://ns.google.com/photos/1.0/camera/",
            "MotionPhotoOffset", video_offset);
        
        if (xmpFile.CanPutXMP(meta)) {
            xmpFile.PutXMP(meta);
            xmpFile.CloseFile(kXMPFiles_UpdateSafely);
            return 0;
        }
        
        xmpFile.CloseFile();
        return -1;
    } catch (XMP_Error& e) {
        return -2;
    }
}
```

**关键限制**：
- ❌ 无法通过 API 保存嵌入文件格式到媒体库
- ⚠️ 必须使用导出/分享功能
- ✅ 小米相册完整识别（使用标准 GCamera 命名空间）

---

## 五、输出模式选择策略

### 5.1 目标设备检测

```typescript
// ArkTS层检测目标设备
enum TargetDevice {
  HUAWEI_SINGLE_FRAME,   // HarmonyOS NEXT
  HUAWEI_DUAL_FRAME,     // EMUI / HarmonyOS 4
  XIAOMI,
  APPLE_IOS,
  GOOGLE_ANDROID,
  UNKNOWN
}

function detectTargetDevice(): TargetDevice {
  // 根据系统版本判断
  const osVersion = deviceInfo.osVersion;
  
  if (osVersion.startsWith('HarmonyOS NEXT')) {
    return TargetDevice.HUAWEI_SINGLE_FRAME;
  }
  if (osVersion.startsWith('HarmonyOS 4') || osVersion.startsWith('EMUI')) {
    return TargetDevice.HUAWEI_DUAL_FRAME;
  }
  if (osVersion.startsWith('HyperOS') || osVersion.startsWith('MIUI')) {
    return TargetDevice.XIAOMI;
  }
  // ...其他检测逻辑
  
  return TargetDevice.UNKNOWN;
}
```

### 5.2 输出格式选择逻辑

```typescript
// 根据目标设备选择输出格式
function selectOutputFormat(target: TargetDevice): OutputFormatConfig {
  switch (target) {
    case TargetDevice.HUAWEI_SINGLE_FRAME:
      // 华为单框架：使用API创建分离文件
      return {
        format: OutputFormat.HUAWEI_SINGLE,
        saveMethod: SaveMethod.API_SUPPORTED,
        fileMode: FileMode.SEPARATE
      };
      
    case TargetDevice.HUAWEI_DUAL_FRAME:
      // 华为双框架：导出嵌入文件+末尾元数据
      return {
        format: OutputFormat.HUAWEI_DUAL,
        saveMethod: SaveMethod.EXPORT_REQUIRED,
        fileMode: FileMode.EMBEDDED,
        metadata: MetadataType.TAIL_METADATA
      };
      
    case TargetDevice.XIAOMI:
      // 小米：导出嵌入文件+XMP-GCamera
      return {
        format: OutputFormat.XIAOMI_EMBEDDED,
        saveMethod: SaveMethod.EXPORT_REQUIRED,
        fileMode: FileMode.EMBEDDED,
        metadata: MetadataType.XMP_GCAMERA
      };
      
    case TargetDevice.APPLE_IOS:
      // Apple：API创建分离文件+MOV
      return {
        format: OutputFormat.APPLE_LIVE_PHOTO,
        saveMethod: SaveMethod.API_SUPPORTED,
        fileMode: FileMode.SEPARATE,
        videoFormat: VideoFormat.MOV_HVC1
      };
      
    default:
      // 其他：导出嵌入文件+GCamera（Google兼容）
      return {
        format: OutputFormat.GOOGLE_MOTION_PHOTO,
        saveMethod: SaveMethod.EXPORT_REQUIRED,
        fileMode: FileMode.EMBEDDED,
        metadata: MetadataType.XMP_GCAMERA
      };
  }
}
```

---

## 六、关键技术依赖

### 6.1 第三方库集成

| 库 | 版本 | 许可证 | 功能 | 集成方式 | 工作量 |
|------|------|-------|------|---------|--------|
| **FFmpeg** | 5.1.x | LGPL v2.1+ | MOV转换 | 动态链接 | 2-3周 |
| **Adobe XMP Toolkit** | 202x.x | BSD 3-Clause | XMP写入 | 静态链接 | 2-3周 |
| **libheif** | 1.x | LGPL | HEIF处理 | 动态链接 | 1周（可选） |

### 6.2 Adobe XMP Toolkit 编译可行性（已验证）

**验证结果**（027）：
- ✅ HarmonyOS NDK 支持 C++17（XMP Toolkit 要求）
- ✅ 依赖库简单：Expat（MIT协议）
- ✅ 需适配平台API差异：文件操作、线程API
- ⚠️ 工作量：2-3周

**编译配置**：

```cmake
# Adobe XMP Toolkit HarmonyOS 集成
cmake_minimum_required(VERSION 3.10)
project(livehub_xmp)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Expat XML解析器
add_library(expat STATIC
    ${EXPAT_ROOT}/lib/xmlparse.c
    ${EXPAT_ROOT}/lib/xmltok.c
    ${EXPAT_ROOT}/lib/xmlrole.c
)

# XMP Toolkit
add_library(xmp STATIC
    ${XMP_ROOT}/source/XMPMeta.cpp
    ${XMP_ROOT}/source/XMPFiles.cpp
    ${XMP_ROOT}/source/XMPUtils.cpp
    # ...其他源文件
)

# Native模块
add_library(livehub_native SHARED
    napi_init.cpp
    xmp_handler.cpp
    metadata_writer.cpp
)

target_link_libraries(livehub_native
    libace_napi.z.so
    libhilog_ndk.z.so
    xmp
    expat
)
```

---

## 七、风险评估与缓解

### 7.1 高风险

| 风险 | 影响 | 缓解措施 | 状态 |
|------|------|---------|------|
| **嵌入文件无法保存到媒体库** | 小米/华为双框架/Google输出受限 | 提供导出/分享功能 | ✅ 已确认方案 |
| **华为双框架末尾元数据不被识别** | 华为双框架用户体验降级 | 实测验证，提供分离文件选项 | ⚠️ 待验证 |
| **XMP Toolkit编译失败** | 小米格式无法创建 | 参考Android移植版本 | ⚠️ 待验证 |

### 7.2 中风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| HEVC MOV在iOS不识别 | Apple输出失败 | FFmpeg `-tag:v hvc1`参数 |
| CinemagraphInfo丢失 | 微动瞬间效果丢失 | 尝试保留转换 |
| 目标设备检测不准确 | 格式选择错误 | 提供手动选择选项 |

---

## 八、参考资料

| 报告编号 | 主题 | 关键发现 |
|---------|------|---------|
| **022** | EMUI动态照片格式 | 双框架末尾元数据详细规格（LIVE_xxxx等） |
| **023** | HarmonyOS动态照片格式 | 单框架分离文件模式，数据库subtype=3 |
| **024** | 单双框架差异 | 存储模式、API差异、格式转换机制 |
| **025** | 华为小米兼容性 | 双方不识别对方格式，必须目标设备格式输出 |
| **026** | Native可行性 | API能力矩阵、嵌入文件保存限制 |
| **027** | Native P0验证 | 嵌入文件无法保存到媒体库，XMP Toolkit编译可行 |
| **030** | CinemagraphInfo结构 | 微动瞬间数据存储规格 |
| **031** | 东湖LivePhoto识别 | 湖内/湖外动态照片处理 |

---

*MVP架构设计 v2.0*
*设计日期：2026-05-19*