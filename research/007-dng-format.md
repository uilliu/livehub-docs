# RAW/DNG 格式处理方案调研报告

> 调研日期：2026-05-16
> 调研深度：中等（格式结构、平台支持、处理方案、MVP优先级）
> 关联文档：`2026-05-16-live-photo-format-research.md`

---

## 1. DNG 格式基础知识

### 1.1 DNG 文件结构

**基于 TIFF 的容器格式**：

DNG (Digital Negative) 是 Adobe 开发的开源 RAW 图像格式，基于 TIFF/EP (Tagged Image File Format / Electronic Photography) 标准。

```
DNG 文件结构:
┌─────────────────────────────────────┐
│ TIFF Header (8 bytes)               │
│ - Byte Order (II/MM)                │
│ - Magic Number (42)                 │
│ - First IFD Offset                  │
├─────────────────────────────────────┤
│ IFD0 (主图像 IFD)                    │
│ - 图像宽度/高度                      │
│ - 原始数据偏移                       │
│ - 压缩信息                           │
│ - DNG 特定标签                       │
├─────────────────────────────────────┤
│ SubIFDs (子图像 IFD)                 │
│ - 预览图像                           │
│ - 缩略图                             │
├─────────────────────────────────────┤
│ EXIF IFD                             │
│ - 标准 EXIF 数据                     │
│ - 拍摄参数                           │
├─────────────────────────────────────┤
│ XMP Metadata                         │
│ - 嵌入的 XMP 数据块                  │
│ - 自定义命名空间支持                 │
├─────────────────────────────────────┤
│ Raw Image Data                       │
│ - 传感器原始数据                     │
│ - 未处理的像素值                     │
├─────────────────────────────────────┤
│ Preview/Thumbnail Data               │
│ - JPEG 预览                          │
├─────────────────────────────────────┤
│ Camera Profile (可选)                │
│ - ICC 色彩配置                       │
└─────────────────────────────────────┘
```

**关键特点**：
- 使用 IFD (Image File Directory) 结构组织元数据
- 支持嵌入式 XMP 元数据块
- 单文件包含所有信息（与厂商 RAW + XMP sidecar 模式不同）
- 可存储多级预览图像

### 1.2 元数据存储位置

**三层元数据结构**：

| 层级 | 存储位置 | 内容类型 |
|-----|---------|---------|
| **TIFF Tags** | IFD 标签 | 图像尺寸、压缩方式、DNG 版本 |
| **EXIF** | EXIF IFD | 拍摄参数（曝光、焦距、GPS） |
| **XMP** | 嵌入块（可扩展） | 自定义元数据、IPTC、自定义命名空间 |

**XMP 元数据访问**：
- DNG 支持 XMP 元数据直接嵌入文件
- 无需外部 .xmp sidecar 文件
- 支持自定义命名空间扩展

### 1.3 与实况照片的关系

**核心结论**：DNG 格式与实况照片功能**互斥**

**原因分析**：

1. **DNG 不支持嵌入视频**：
   - DNG 规范仅定义静态图像存储
   - 无视频数据存储能力
   - 无法像 HEIC/JPEG 那样嵌入视频帧

2. **RAW 模式与动态照片冲突**：
   - RAW 模式需要保存完整的传感器数据
   - 动态照片需要同步录制视频帧
   - 两种模式对系统资源需求完全不同
   - 手机厂商普遍不支持同时开启

**实际设备行为**：

| 品牌 | RAW 模式 | 动态照片 | 同时支持 |
|-----|---------|---------|---------|
| 华为 | 支持（Pro模式） | 支持（动态照片） | ❌ 不支持 |
| 小米 | 支持（专业模式） | 支持（实况照片） | ❌ 不支持 |
| Apple | 不支持 RAW | Live Photo | - |
| Samsung | 支持（专业模式） | Motion Photo | ❌ 不支持 |

**华为 RAW 模式实况照片**：

根据现有调研，华为 RAW 模式下：
- 保存 `.dng` 文件（静态 RAW 图像）
- **无关联的 MP4 视频**（动态照片功能被禁用）
- 仅保留原始 RAW 数据，无动态效果

---

## 2. HarmonyOS DNG 支持情况

### 2.1 ImageKit 解码能力

**支持的解码格式**（来源：官方文档）：

```
解码支持格式:
png, jpeg, bmp, gif, webp, dng, heic, wbmp, heifs, tiff
```

**DNG 解码特性**：

| 能力 | API 支持 | 说明 |
|-----|---------|-----|
| DNG 格式识别 | ✅ 支持 | MIME type: `image/x-adobe-dng` |
| 基础解码 | ✅ 支持 | 可生成 PixelMap |
| EXIF 读取 | ✅ 支持 | 通过 `getImageProperty` |
| XMP 读取 | ✅ 支持 | 通过 `getMetadata` |
| HDR 解码 | ⚠️ 设备依赖 | 取决于设备硬件 |
| 采样解码 | ❌ 不支持 | DNG 不支持 downsampling |

**API 示例**：

```typescript
import { image } from '@kit.ImageKit';

// 创建 ImageSource
let imageSource = image.createImageSource(fd);

// 获取图片信息
let imageInfo = await imageSource.getImageInfo();

// 创建 PixelMap（解码 RAW 到位图）
let decodingOptions: image.DecodingOptions = {
  editable: true,
  desiredPixelFormat: image.PixelMapFormat.RGBA_8888,
  desiredDynamicRange: image.DecodingDynamicRange.AUTO,
};
let pixelMap = await imageSource.createPixelMap(decodingOptions);

// 读取 EXIF
let orientation = imageSource.getImagePropertySync(image.PropertyKey.ORIENTATION);
```

### 2.2 ImageKit 编码能力

**支持的编码格式**：

```
编码支持格式:
jpeg, webp, png, heic, gif
```

**DNG 编码状态**：

| 能力 | API 支持 | 说明 |
|-----|---------|-----|
| DNG 编码 | ❌ 不支持 | 无法创建新 DNG 文件 |
| DNG 修改 | ❌ 不支持 | 无法修改 DNG 文件元数据 |
| DNG → JPEG 转换 | ⚠️ 间接支持 | 解码后重新编码为 JPEG |

### 2.3 元数据操作能力

**EXIF/XMP 操作限制**：

根据 HarmonyOS 官方文档：

> **DNG 格式仅支持读取，不支持修改**
> 来源：`2026-05-16-harmonyos-api-research.md`

**具体限制**：

| 操作 | JPEG/HEIC | DNG |
|-----|-----------|-----|
| EXIF 读取 | ✅ | ✅ |
| EXIF 写入 | ✅ | ❌ |
| XMP 读取 | ✅ | ✅ |
| XMP 写入 | ⚠️ 需 Native | ❌ |
| MakerNotes 读取 | ✅ | ❌ 不适用 |

**C++ Native API**：

```cpp
// 仅支持 JPEG/HEIF/WEBP/PNG，不支持 DNG
OH_ImageSourceNative_ModifyImageProperty(source, &key, &value);
```

**可用的 PropertyKey**：

仅标准 EXIF 标签，不包括自定义命名空间：
- `IMAGE_WIDTH`, `IMAGE_LENGTH`
- `ORIENTATION`
- `MAKE`, `MODEL`
- `BITS_PER_SAMPLE`
- 等 TIFF/EXIF 标准字段

---

## 3. 实况照片元数据在 DNG 中的位置

### 3.1 技术可行性分析

**关键结论**：DNG 文件**无法作为实况照片载体**

**原因**：

1. **无视频存储能力**：
   - DNG 规范不支持视频数据嵌入
   - 无法存储实况照片的动态视频部分

2. **无实况照片标识**：
   - 华为实况照片依赖 `HwMotionPhoto` MakerNotes 标签
   - DNG 文件通常不包含 MakerNotes 区域
   - HarmonyOS API 不支持 DNG 元数据写入

3. **设备层面互斥**：
   - RAW 模式拍摄时，动态照片功能被禁用
   - 实际场景中不存在"RAW 实况照片"的组合

### 3.2 华为 RAW 模式实况照片情况

**实际文件产出**：

当用户开启华为相机 RAW 模式拍摄时：

| 模式 | 产出文件 | 动态效果 |
|-----|---------|---------|
| 默认模式 + 动态照片 | JPG + MP4 | ✅ 动态效果 |
| RAW 模式 | DNG | ❌ 仅静态 RAW |
| RAW 模式 + 动态照片 | ❌ 不可同时启用 | - |

**结论**：
- RAW 模式下拍摄的 DNG 文件**不是实况照片**
- 无关联的 MP4 视频
- 无动态照片元数据标识
- 不属于实况照片转换的目标文件

---

## 4. 处理方案

### 方案 A：直接处理 DNG

**可行性**：❌ **不可行**

**原因**：
1. HarmonyOS ImageKit 不支持 DNG 元数据写入
2. DNG 格式不支持嵌入视频数据
3. 实际场景中不存在需要处理的 DNG 实况照片
4. 无技术手段在 DNG 中写入实况照片标识

**结论**：此方案技术上不可行，场景上不必要。

---

### 方案 B：转换后处理

**可行性**：⚠️ **部分可行**

**处理流程**：

```
DNG (RAW) → 解码 → PixelMap → 编码 → JPEG
         ↓                              ↓
    读取 EXIF                      写入 EXIF
    读取 XMP                       写入 XMP
                                  (可添加实况照片标识)
```

**技术实现**：

```typescript
// 1. 解码 DNG
let imageSource = image.createImageSource(dngFd);
let pixelMap = await imageSource.createPixelMap(decodingOptions);

// 2. 获取原始 EXIF
let originalExif = {};
for (const key of supportedKeys) {
  originalExif[key] = imageSource.getImagePropertySync(key);
}

// 3. 编码为 JPEG
let packer = image.createImagePacker();
let jpegBuffer = await packer.pack(pixelMap, {
  format: 'image/jpeg',
  quality: 95
});

// 4. 创建新 JPEG ImageSource
let newSource = image.createImageSource(jpegBuffer);

// 5. 写入 EXIF（需 Native 层）
// 注意：从 buffer 创建的 ImageSource 不支持属性修改
// 必须使用文件路径创建 ImageSource
```

**限制**：

1. **EXIF 写入需文件路径**：
   - `modifyImageProperty` 不支持从 buffer 创建的 ImageSource
   - 必须先将 JPEG 写入文件，再修改元数据

2. **元数据不完整**：
   - 部分 RAW 特有元数据无法保留
   - 色彩配置信息丢失

3. **质量损失**：
   - RAW → JPEG 转换会损失原始数据
   - 不符合 RAW 用户保留原始数据的期望

4. **无视频部分**：
   - DNG 原本无关联视频
   - 转换后仍不是实况照片
   - 仅得到普通 JPEG 图片

**适用场景**：
- 用户希望将 RAW 转换为可分享的 JPEG
- 不涉及实况照片功能

**不适用场景**：
- 实况照片转换（DNG 本身不是实况照片）

---

### 方案 C：仅处理 JPG 副本

**可行性**：⚠️ **理论可行，实际场景不存在**

**前提条件**：
- DNG 文件有对应的 JPG 副本
- JPG 副本是实况照片（有 `HwMotionPhoto` 标签）

**实际情况**：
- 华为 RAW 模式不产出 JPG + DNG 组合
- RAW 模式下动态照片功能被禁用
- 不存在"DNG 实况照片需处理 JPG 副本"的场景

**结论**：此方案在实况照片场景下不适用。

---

### 方案 D：跳过 DNG 文件

**可行性**：✅ **推荐方案**

**逻辑**：

```typescript
function isProcessableLivePhoto(filePath: string): boolean {
  const ext = getExtension(filePath).toLowerCase();
  
  // DNG 文件不是实况照片，跳过处理
  if (ext === 'dng') {
    return false;
  }
  
  // 仅处理 JPEG/HEIC 实况照片
  if (ext === 'jpg' || ext === 'jpeg' || ext === 'heic') {
    return hasLivePhotoMetadata(filePath);
  }
  
  return false;
}
```

**理由**：
1. DNG 文件不包含实况照片数据
2. RAW 模式下不存在实况照片
3. 用户使用 RAW 模式是为了保留原始数据，不期望转换
4. 跳过 DNG 符合用户预期和技术现实

---

## 5. MVP 优先级判断

### 5.1 功能优先级矩阵

| 功能 | 用户需求 | 技术可行性 | MVP 优先级 |
|-----|---------|-----------|-----------|
| DNG 实况照片处理 | ❌ 不存在 | ❌ 不可行 | P0-跳过 |
| DNG → JPEG 转换 | ⚠️ 低需求 | ⚠️ 部分可行 | P2-未来版本 |
| JPEG 实况照片处理 | ✅ 高需求 | ✅ 可行 | P1-MVP核心 |
| HEIC 实况照片处理 | ✅ 高需求 | ✅ 可行 | P1-MVP核心 |

### 5.2 DNG 处理在 MVP 中的定位

**结论**：DNG 格式处理**不纳入 MVP 范围**

**依据**：

1. **场景不存在**：
   - RAW 模式拍摄不产出实况照片
   - DNG 文件无动态照片数据
   - 用户需求为零

2. **技术不支持**：
   - HarmonyOS 不支持 DNG 元数据写入
   - 无法在 DNG 中写入实况照片标识
   - 即使强制转换也会丢失 RAW 原始数据价值

3. **用户预期**：
   - RAW 用户期望保留原始数据
   - 不希望对 RAW 文件进行转换处理
   - RAW 模式使用场景与实况照片完全不同

### 5.3 建议处理逻辑

**在实况照片选择界面**：

```typescript
// 过滤逻辑
function filterLivePhotoCandidates(assets: PhotoAsset[]): PhotoAsset[] {
  return assets.filter(asset => {
    // 排除 DNG 格式
    if (asset.extension === 'dng') {
      logInfo(`跳过 RAW 文件: ${asset.uri}`);
      return false;
    }
    
    // 仅保留 JPEG/HEIC
    if (asset.extension !== 'jpg' && asset.extension !== 'heic') {
      return false;
    }
    
    // 检查实况照片标识
    return hasLivePhotoMetadata(asset.uri);
  });
}
```

**用户提示**：

在选择实况照片时，可向用户说明：
> "RAW 格式照片（DNG）不是实况照片，不会出现在选择列表中"

---

## 6. 结论和建议

### 6.1 核心结论

| 结论 | 说明 |
|-----|-----|
| DNG ≠ 实况照片 | RAW 模式拍摄不产出实况照片 |
| DNG 无视频数据 | DNG 格式不支持嵌入视频 |
| HarmonyOS 不支持 DNG 写入 | 无法修改 DNG 元数据 |
| MVP 不需处理 DNG | 跳过 DNG 文件符合预期 |

### 6.2 技术建议

1. **实况照片检测逻辑**：
   - 仅检测 JPEG/HEIC 格式
   - 明确排除 DNG 格式
   - 不尝试读取 DNG 的实况照片标识

2. **用户界面设计**：
   - 照片选择器仅显示 JPEG/HEIC 实况照片
   - RAW 文件自动过滤
   - 可在帮助文档中说明 RAW 模式与实况照片的关系

3. **错误处理**：
   - 如用户通过其他方式选择 DNG 文件
   - 明确提示"RAW 照片不支持实况照片转换"

4. **未来版本考虑**：
   - 如需支持 RAW → JPEG 转换（非实况照片场景）
   - 需实现完整的 RAW 解码和 EXIF 保留逻辑
   - 作为独立功能，不与实况照片转换关联

### 6.3 参考资料

**官方文档**：
- Adobe DNG Specification: https://helpx.adobe.com/camera-raw/digital-negative.html
- HarmonyOS ImageKit: https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/arkts-apis-image-V5
- HarmonyOS ImageSource: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/image-source-c

**开源工具**：
- ExifTool: https://exiftool.org/ (支持 DNG 元数据读写)
- Adobe XMP Toolkit: https://github.com/nicklockwood/XMPToolkit

**格式参考**：
- ExifTool DNG Tags: https://exiftool.org/TagNames/DNG.html
- TIFF/EP Standard (ISO 12234-2)

---

*报告完成于 2026-05-16*
*DNG 格式不纳入 MVP 实况照片处理范围*