# 动态照片/实况照片调研综述

> 调研日期：2026-05-16
> 调研深度：中等（核心原理、文件结构、检测方法、转换可行性）

---

## 1. 华为 Live Photo（动态照片）

> **⚠️ 重要：单/双框架存储方式差异**
> 详细规格见：[017-huawei-framework-difference-spec.md](017-huawei-framework-difference-spec.md)

### 框架类型与存储方式

| 系统版本 | 框架类型 | 存储方式 | 存储路径 |
|---------|---------|---------|---------|
| **EMUI** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS 4 及以前** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS NEXT** | 单框架 | **分离文件模式** | `/storage/media/100/local/files/Photo/` |

### 双框架：嵌入文件模式

**文件结构**：
- 单一 JPG 文件，视频嵌入在文件末尾
- 类似 Google/Samsung Motion Photo 格式
- 通过 XMP `MotionPhotoOffset` 或 `ftypmp4` 标记定位视频

```
双框架动态照片结构：
├── 单一 JPG 文件（如 IMG_XXXX.jpg）
│   ├── JPEG 图片数据（封面帧）
│   ├── XMP 元数据
│   │   └── MotionPhoto = 1
│   │   └── MotionPhotoOffset = 嵌入视频偏移量
│   └── 嵌入 MP4 视频数据（约3秒）
│       └── 视频从 offset 位置开始，到文件末尾
```

**嵌入视频提取**：
```python
import re

def extract_embedded_video(jpg_path):
    with open(jpg_path, "rb") as file:
        data = file.read()
        # 查找 'ftypmp4' 标记定位视频起始
        match = re.search(b'ftypmp4', data)
        if match:
            ofs = match.start()
            video_data = data[ofs-4:]  # 从 ftyp 前4字节开始
            return video_data
    return None
```

### 单框架：分离文件模式

**文件结构**：
- JPG + MP4 两个独立物理文件
- 通过 MovingPhoto API 关联
- 媒体库数据库记录关联关系

```
单框架动态照片结构：
├── IMG_XXXX.jpg  (静态图片 - 独立物理文件)
│   ├── JPEG 图片数据
│   └── EXIF MakerNotes: HwMotionPhoto = 1
│
├── IMG_XXXX.mp4  (动态视频 - 独立物理文件)
│   ├── 约3秒视频
│   └── 编码：H.264/H.265
│
└── 通过 MovingPhoto API 关联
    └── PhotoAsset.subType = MOVING_PHOTO
```

### 元数据存储

**双框架（嵌入模式）**：

XMP 字段（类似 Google Motion Photo）：
- `XMP-GCamera:MotionPhoto = 1` - 标识为动态照片
- `XMP-GCamera:MotionPhotoOffset = <偏移量>` - 视频数据偏移
- 视频定位：从文件末尾倒数 offset 字节

**单框架（分离模式）**：

EXIF MakerNotes 字段（嵌入在 JPG 文件中）：
- `MakerNotes:HwMotionPhoto = 1` - 标识为动态照片
- `MakerNotes:HwMotionPhotoIndex` - 字节偏移（部分设备）

关联机制：
1. **文件名规则**：图片与视频共享相同文件名前缀
2. **数据库记录**：媒体库通过 PhotoAsset.subType = MOVING_PHOTO 记录关联
3. **MakerNotes 标记**：图片 EXIF 中包含 `HwMotionPhoto` 标签

### 检测方法

**方法一：框架类型判断**
```typescript
function getFrameworkType(): 'dual' | 'single' {
    // 通过系统版本判断
    const osVersion = getOsVersion();
    if (osVersion.startsWith('HarmonyOS NEXT')) return 'single';
    if (osVersion.startsWith('HarmonyOS 4') || osVersion.startsWith('EMUI')) return 'dual';
    
    // 或通过文件路径判断
    const basePath = getMediaBasePath();
    if (basePath.includes('/storage/media/100/')) return 'single';
    if (basePath.includes('/storage/emulated/0/')) return 'dual';
}
```

**方法二：双框架嵌入视频检测**
```typescript
function isHuaweiDualFrameworkLivePhoto(imagePath: string): boolean {
    const fileData = readFile(imagePath);
    // 查找 ftypmp4 标记（MP4 文件标识）
    const mp4Marker = fileData.indexOf('ftypmp4');
    if (mp4Marker > 0) {
        // 存在嵌入的 MP4 视频
        return true;
    }
    // 或检查 XMP MotionPhotoOffset
    const xmp = getXmpData(imagePath);
    return xmp?.MotionPhoto === 1;
}
```

**方法三：单框架分离文件检测**
```typescript
function findHuaweiSingleFrameworkLivePhotoPair(imagePath: string): string | null {
    const nameWithoutExt = imagePath.replace(/\.(jpg|dng)$/i, '');
    const videoPath = nameWithoutExt + '.mp4';
    if (fileExists(videoPath)) {
        // 验证时间戳一致性（2秒内视为配对）
        const imgTime = getCreationTime(imagePath);
        const videoTime = getCreationTime(videoPath);
        if (Math.abs(imgTime - videoTime) < 2) {
            return videoPath;
        }
    }
    return null;
}
```

**方法四：系统 API 检测（单框架）**
```typescript
// HarmonyOS NEXT MovingPhoto API
let predicates = new dataSharePredicates.DataSharePredicates();
predicates.equalTo(
    photoAccessHelper.PhotoKeys.PHOTO_SUBTYPE,
    photoAccessHelper.PhotoSubtype.MOVING_PHOTO
);
let fetchOptions: photoAccessHelper.FetchOptions = {
    fetchColumns: [],
    predicates: predicates
};
let assetResult = await phAccessHelper.getAssets(fetchOptions);
```

### 转换要点

**华为 → Apple Live Photo**：

双框架（嵌入模式）：
1. 提取嵌入的 MP4 视频（从 JPG 文件末尾）
2. MP4 转 MOV（需 FFmpeg，HarmonyOS 不支持 MOV）
3. 为视频添加 `ContentIdentifier` 元数据
4. 为图片添加相同的 `ContentIdentifier`

单框架（分离模式）：
1. 直接使用独立的 JPG + MP4 文件
2. MP4 转 MOV（需 FFmpeg）
3. 写入配对的 `ContentIdentifier`

**华为 → Samsung/Google Motion Photo**：

双框架（嵌入模式）：
1. 保持嵌入格式（与 Samsung/Google 格式兼容）
2. 可直接使用或调整 XMP 标签

单框架（分离模式）：
1. 将 MP4 嵌入 JPG 文件末尾
2. 添加 XMP 元数据标记偏移位置
3. 计算偏移量：`offset = 文件总长度 - 视频起始位置`

**华为 → 小米**：

双框架（嵌入模式）：
1. 保持嵌入格式（与小米 MVIMG 格式兼容）
2. 或提取视频转为分离文件模式

单框架（分离模式）：
1. 保持双文件结构（JPG + MP4）
2. 写入小米格式的 XMP 元数据标签

**格式转换API**：
- 双→单转换：ConvertToMovingPhoto API
- 数据克隆场景支持双框架到单框架迁移
- extraData 目录存储转换临时数据

### 开源参考

- **ExifTool**: 可读取华为 MakerNotes 中的 `HwMotionPhoto` 标签
  - 参考: https://exiftool.org/TagNames/Huawei.html
  - https://github.com/exiftool/exiftool
- **pyexiv2**: Python EXIF 库，支持 MakerNotes 解析
  - 参考: https://github.com/LeoHsiao1/pyexiv2
- HarmonyOS PhotoAccessHelper API 提供原生支持

---

## 2. 小米 Live Photo（实况照片）

### 文件结构

**分离文件模式**（主流，Separate-File Mode）：
- 静态图片：`.jpg` 或 `.heic`（HEIF格式）- 独立物理文件
- 动态视频：`.mp4` 格式 - 独立物理文件
- 与华为类似，文件系统中存在两个独立文件，相册APP通过元数据关联显示为一个"实况照片"

**嵌入文件模式**（部分机型，Embedded-File Mode，仿三星格式）：
- 文件名格式：`MVIMG_YYYYMMDD_HHMMSS.jpg`
- **一个物理文件**：视频数据嵌入在 JPEG 文件末尾
- 类似 Samsung Motion Photo 格式
- 通过 XMP `MotionPhotoOffset` 标识视频数据起始位置

**示例**：
```
分离文件模式：
DCIM/Camera/
├── IMG_20260516_001.jpg    # 主图 - 独立物理文件
└── IMG_20260516_001.mp4    # 关联视频 - 独立物理文件
# 相册APP显示为一个"实况照片"条目

嵌入文件模式：
DCIM/Camera/
└── MVIMG_20260516_001.jpg  # 单一物理文件，内含静态图片+嵌入视频
```

### 元数据存储

**官方 XMP 标签定义**（基于 ExifTool XMP-GCamera 命名空间）：

| 标签名 | 类型 | 命名空间 | 说明 |
|-------|-----|---------|-----|
| `MotionPhoto` | Integer | `XMP-GCamera` | 标识为动态照片（值=1） |
| `MotionPhotoVersion` | Integer | `XMP-GCamera` | 动态照片格式版本 |
| `MotionPhotoPresentationTimestampUs` | Integer | `XMP-GCamera` | 静态帧展示时间（微秒） |
| `MicroVideo` | Integer | `XMP-GCamera` | 微视频标识（旧版标签） |
| `MicroVideoOffset` | Integer | `XMP-GCamera` | 视频数据偏移量（嵌入模式） |
| `MicroVideoVersion` | Integer | `XMP-GCamera` | 微视频格式版本 |
| `MicroVideoPresentationTimestampUs` | Integer | `XMP-GCamera` | 展示时间戳（微秒） |

**关键说明**：
- `MicroVideoOffset` = 文件总长度 - 视频数据起始位置
- 视频数据从 `(文件长度 - MicroVideoOffset)` 字节开始，直到文件末尾
- 小米嵌入模式与 **Google Motion Photo 完全兼容**，使用相同的 GCamera 命名空间

**小米特有标签**（部分机型）：
- `MicroDescription` - 小米相机描述信息（推测）
- `MicroVersion` - MIUI 相机版本（推测）
- **注**：这些标签在实际文件中可能不存在，需实测验证

### 检测方法

**方法一：文件名模式检测**
```typescript
function isXiaomiMvimg(imagePath: string): boolean {
  return getFileName(imagePath).startsWith('MVIMG_');
}
```

**方法二：XMP 标签检测**
```typescript
function isXiaomiLivePhoto(imagePath: string): boolean {
  const xmp = getXmpData(imagePath);
  return (
    xmp?.MotionPhoto === 1 ||
    xmp?.['{http://ns.google.com/photos/1.0/camera/}MotionPhoto'] === 1
  );
}
```

**方法三：配对文件检测**
```typescript
function findXiaomiLivePhotoPair(imagePath: string): string | null {
  const nameWithoutExt = imagePath.replace(/\.(jpg|heic)$/i, '');
  for (const ext of ['.mp4', '.MP4']) {
    const videoPath = nameWithoutExt + ext;
    if (fileExists(videoPath)) return videoPath;
  }
  return null;
}
```

### 转换要点

**小米 → 华为**：
1. 双文件模式：保持 JPG + MP4 结构，写入华为 `HwMotionPhoto` MakerNotes
2. 单文件模式：提取嵌入视频，创建独立 MP4 文件

**提取嵌入视频（单文件模式）**：
```typescript
function extractEmbeddedVideo(mvimgPath: string): ArrayBuffer | null {
  const xmp = getXmpData(mvimgPath);
  const offset = parseInt(xmp?.MotionPhotoOffset || '0');
  if (offset > 0) {
    const fileSize = getFileSize(mvimgPath);
    const videoStart = fileSize - offset;
    return readFileRange(mvimgPath, videoStart, fileSize);
  }
  return null;
}
```

**小米 → Apple Live Photo**：
1. 双文件模式：与华为处理类似，写入 `ContentIdentifier`
2. 单文件模式：先提取视频，再处理

### 开源参考

- **ExifTool**: 支持 Xiaomi/Micro XMP 标签解析
  - 参考: https://exiftool.org/TagNames/XMP.html#micro
- 小米官方未公开完整格式规范
- 格式与 Google Motion Photo 兼容度高

---

## 3. Apple Live Photo（概要）

### 文件结构

**双文件模式**：
- 图片：`.heic`（HEIF格式，默认）或 `.jpg`
- 视频：`.mov`（QuickTime格式，H.264编码）

**文件名规则**：
- 共享相同 `ContentIdentifier`（UUID）
- 文件名格式：`IMG_XXXX` 或用户自定义

### 元数据存储

**关键 EXIF/XMP 标签**：
- `ContentIdentifier = <UUID>` - 图片和视频共享（必须）
- `MediaGroupUUID = <UUID>` - iOS 9+ 替代字段

**图片文件元数据**：
- `QuickTime:ContentIdentifier = <UUID>`
- `XMP-apple-livephoto:LivePhotoVitalityScore = <分数>`

**视频文件元数据**：
- `QuickTime:ContentIdentifier = <UUID>` - 必须与图片一致
- `QuickTime:MajorBrand = qt`
- `QuickTime:CompatibleBrands = ['qt', 'MSNV']`

### 检测方法

```typescript
function isAppleLivePhotoPair(imagePath: string, videoPath: string): boolean {
  const imgUuid = getContentIdentifier(imagePath);
  const videoUuid = getContentIdentifier(videoPath);
  return imgUuid && imgUuid === videoUuid;
}

function getContentIdentifier(filePath: string): string | null {
  const exif = getExifData(filePath);
  return exif?.ContentIdentifier || exif?.MediaGroupUUID;
}
```

### 转换要点

**创建 Apple Live Photo**：
1. 确保 MOV 视频使用 H.264 编码
2. 生成唯一 UUID 作为 `ContentIdentifier`
3. 为图片和视频写入相同的 UUID
4. 视频时长必须约3秒
5. 视频必须包含音轨（可为静音）

**其他格式 → Apple**：
1. 提取源格式的图片和视频
2. 视频需重编码为 MOV/H.264（如原格式非 MOV）
3. 写入配对的 `ContentIdentifier`

### 开源参考

- **makelive** (GitHub): Python 工具，创建 Apple Live Photos
- **live-photo-conv**: 跨平台转换工具
- **live-photo-js**: JavaScript 解析库
- **ExifTool**: 支持 `ContentIdentifier` 操作

---

## 4. Samsung Motion Photo（概要）

### 文件结构

**单文件模式**：
- 文件名格式：`MVIMG_YYYYMMDD_HHMMSS.jpg`
- 视频嵌入在 JPEG 文件末尾

**文件组成**：
```
[JPEG Image Data][XMP Metadata][MP4 Video Data]
```

### 元数据存储

**XMP 字段**（GCamera 命名空间）：
- `XMP-GCamera:MotionPhoto = 1`
- `XMP-GCamera:MotionPhotoVersion = 1`
- `XMP-GCamera:MotionPhotoOffset = <字节偏移量>`
- `XMP-GCamera:MotionPhotoPresentationTimestampUs = <时间戳>`

**偏移量说明**：
- `MotionPhotoOffset` = 文件总长度 - 视频起始位置
- 从文件末尾倒数 offset 字节即为视频数据起点

### 检测方法

```typescript
function isSamsungMotionPhoto(imagePath: string): boolean {
  const xmp = getXmpData(imagePath);
  return xmp?.MotionPhoto === 1 && getFileName(imagePath).startsWith('MVIMG_');
}

function getVideoOffset(imagePath: string): number {
  const xmp = getXmpData(imagePath);
  const offset = parseInt(xmp?.MotionPhotoOffset || '0');
  const totalSize = getFileSize(imagePath);
  return offset > 0 ? totalSize - offset : -1;
}
```

### 转换要点

**Samsung → Apple Live Photo**：
1. 提取嵌入的 MP4 视频
2. 为视频添加 `ContentIdentifier` 元数据
3. 为图片添加相同的 `ContentIdentifier`
4. 将视频重命名为 `.MOV`（需重编码）

**提取嵌入视频**：
```typescript
function extractSamsungVideo(mvimgPath: string, outputPath: string): boolean {
  const xmp = getXmpData(mvimgPath);
  const offset = parseInt(xmp?.MotionPhotoOffset || '0');
  if (offset > 0) {
    const fileSize = getFileSize(mvimgPath);
    const videoStart = fileSize - offset;
    const videoData = readFileRange(mvimgPath, videoStart, fileSize);
    writeFile(outputPath, videoData);
    return true;
  }
  return false;
}
```

### 开源参考

- **MotionPhoto2**: 创建 Google/Samsung Motion Photos
- **mvimg-parser**: 解析 Samsung MVIMG 文件
- **motion-photo-extractor**: Samsung/Google 视频提取工具

---

## 5. Google Motion Photos（Android 动态照片格式 1.0）

> **官方规范**：2026-05-18 从 Android 开发者文档获取
> **详见**：[013-android-motion-photo-spec.md](002-android-motion-photo)

### 文件结构

**单文件模式**：
- 文件扩展名：`.jpg`
- 结构：`[主静态图片][XMP 元数据][视频文件数据]`
- 视频内嵌（MP4/MOV 格式）

### 元数据存储（官方规范）

**Camera 元数据**：

| 属性名 | 命名空间 | 类型 | 说明 |
|-------|---------|-----|-----|
| `MotionPhoto` | Camera | Integer | 0=非动态照片；1=动态照片 |
| `MotionPhotoVersion` | Camera | Integer | 格式版本号 |
| `MotionPhotoPresentationTimestampUs` | Camera | Integer | 静态帧展示时间（微秒） |

**Container 元数据**（新增）：

| 属性名 | 命名空间 | 说明 |
|-------|---------|-----|
| `Directory` | GContainer | Item 结构的有序数组 |
| `Item:Semantic` | Container | 内容语义（"Primary"、"Video"） |
| `Item:Length` | Container | 内容长度（字节） |

**⚠️ 关键发现：`MicroVideoOffset` 已废弃**

根据 Android 动态照片格式 1.0 规范：
- ❌ `Camera:MicroVideoOffset` **已从规范中删除**，必须忽略
- ✅ 视频定位改用 `GContainer:ItemLength`
- 视频起始位置 = `文件长度 - ItemLength(Video项)`

**废弃属性（微视频 V1 规范）**：

| 属性名 | 状态 | 说明 |
|-------|-----|-----|
| `MicroVideo` | ❌ 废弃 | 必须忽略 |
| `MicroVideoOffset` | ❌ 废弃 | 已替换为 `GContainer:ItemLength` |

### 检测与提取方法

详见 [013-android-motion-photo-spec.md](002-android-motion-photo) 第六章"技术建议"

### 开源参考

- **Android 官方文档**: [动态照片格式 1.0](https://developer.android.com/media/platform/motion-photo-format?hl=zh-cn)
- **libmphoto**: GitHub，专门处理 Motion Photos
- **XMP 规范**: [ISO 16684-1:2011(E)](https://github.com/adobe/XMP-Toolkit-SDK/blob/main/docs/XMPSpecificationPart1.pdf)

---

## 6. HarmonyOS 实况照片处理能力

### PhotoAccessHelper API

**核心能力**：
```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';

// 获取 PhotoAccessHelper 实例
let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);

// 查询媒体资产
let fetchResult = await phAccessHelper.getAssets(fetchOptions);

// 创建图片资产
let uri = await phAccessHelper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'jpg');

// 创建视频资产
let videoRequest = photoAccessHelper.MediaAssetChangeRequest.createVideoAssetRequest(context, fileUri);
await phAccessHelper.applyChanges(videoRequest);
```

**PhotoViewPicker 选择器**：
```typescript
let photoPicker = new photoAccessHelper.PhotoViewPicker();
let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
photoSelectOptions.maxSelectNumber = 50; // 批量选择上限

let result = await photoPicker.select(photoSelectOptions);
// result.photoUris: 选中的照片 URI 数组
```

**权限要求**：
- `ohos.permission.READ_IMAGEVIDEO` - 读取图片/视频
- `ohos.permission.WRITE_IMAGEVIDEO` - 写入图片/视频

### 元数据处理

**读取 EXIF/XMP**：
```typescript
// 从 PixelMap 获取元数据
let metadata = await pictureObj.getMetadata(image.MetadataType.EXIF_METADATA);
```

**修改 EXIF**：
- 支持 C++ Native API: `OH_ImageSourceNative_ModifyImageProperty`
- ArkTS API 有限支持，复杂操作需 Native 层

### 视频处理

**封装格式支持**：
- MP4: `AV_OUTPUT_FORMAT_MPEG_4`
- MOV: 需确认是否支持（QuickTime 格式）

**编码/解码**：
- 使用 `OH_AVCodec` 系列 API
- 支持硬件加速编码

### 分享功能

**系统分享面板**：
```typescript
import { systemShare } from '@kit.ShareKit';

let shareData = new systemShare.SharedData({
  utd: utd.UniformDataType.IMAGE,
  content: fileUri,
  title: '实况照片'
});

let controller = new systemShare.ShareController(shareData);
await controller.show(context, {
  selectionMode: systemShare.SelectionMode.SINGLE
});
```

### 后台任务

**长时任务支持**：
- `backgroundTaskManager.BackgroundMode` 支持数据处理类型
- 需在 `module.json5` 声明 `backgroundModes`

### 进度通知

**进度条通知**：
```typescript
import { notificationManager } from '@kit.NotificationKit';

let notificationRequest = {
  id: 1,
  content: { ... },
  template: {
    name: 'downloadTemplate',
    data: {
      title: '转换进度',
      fileName: '实况照片',
      progressValue: 45
    }
  }
};
await notificationManager.publish(notificationRequest);
```

---

## 7. MVP 可行性判断

### 技术可行性（修正版）

| 功能 | 可行性 | 关键依赖 | 备注 |
|-----|-------|---------|-----|
| 华为实况照片检测 | ✅ 高 | MakerNotes读取 + 文件名配对 | 双文件模式，结构清晰 |
| 小米实况照片检测 | ✅ 高 | XMP读取 + 文件名配对/提取 | 双文件+单文件均有开源参考 |
| 华为→小米转换 | ⚠️ 中 | 保持MP4 + 写入小米XMP | 双文件互转难度较低 |
| 小米→华为转换 | ⚠️ 中 | 保持MP4 + 写入华为MakerNotes | 需Native层写入MakerNotes |
| 华为/小米→Apple转换 | ⚠️ 中 | MP4→MOV转码 + ContentIdentifier | MOV编码要求需验证 |
| 单文件→双文件转换 | ⚠️ 中 | 嵌入视频提取 + XMP偏移解析 | 偏移计算有成熟方案 |
| EXIF保留 | ⚠️ 中 | 元数据复制 + 视频元数据同步 | 需处理视频部分元数据 |
| 批量选择 | ✅ 高 | PhotoViewPicker (maxSelectNumber) | 最多50张 |
| 进度显示 | ✅ 高 | 通知系统 + 进度模板 | downloadTemplate |
| 系统分享 | ✅ 高 | ShareKit | 支持多文件分享 |
| 后台转换 | ⚠️ 中 | BackgroundTaskManager | 有时长限制，需分片处理 |

### 关键技术风险

**高风险（可能阻塞部分功能）**：

| 风险 | 说明 | 缓解措施 |
|-----|------|---------|
| MOV封装不支持 | HarmonyOS可能不原生支持MOV写入 | 引入FFmpeg或仅输出MP4格式 |
| MakerNotes写入受限 | ArkTS层写入MakerNotes能力有限 | Native层扩展或使用XMP替代方案 |
| 后台任务时长限制 | 视频转码可能超时 | 分片处理+断点续传 |

**中风险（增加开发复杂度）**：

| 风险 | 说明 | 缓解措施 |
|-----|------|---------|
| 视频编码兼容性 | 不同编码格式转码失败 | 编码检测+格式降级 |
| 内存压力 | 批量处理时内存溢出 | 限制批量数量+内存监控 |

### 建议

1. **Phase 1**: 先实现华为/小米双文件模式检测和解析（最确定）
2. **Phase 2**: 实现华为↔小米转换（同为双文件+MP4，难度最低）
3. **Phase 3**: 扩展单文件格式支持（Samsung/Google）和Apple格式输出
4. **技术选型**: 如 Native 开发复杂，考虑集成 FFmpeg 库处理视频转码

---

## 参考资料

### 官方文档
- HarmonyOS PhotoAccessHelper: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/photoaccesshelper-overview-V5
- **HarmonyOS MovingPhoto API**: [详细规格](014-harmonyos-movingphoto-api.md)
- Apple Live Photos: https://developer.apple.com/documentation/avfoundation/cameras_and_media_capture/capturing_still_and_live_photos
- **Apple Live Photo 技术规格**: [详细规格](001-apple-livephoto)
- Android MotionPhotoUtils: https://developer.android.com/reference/androidx/exifinterface/media/MotionPhotoUtils
- **Android 动态照片格式 1.0**: [规格摘要](002-android-motion-photo)

### 开源项目与工具参考
- ExifTool: https://exiftool.org/ (支持华为MakerNotes、小米XMP)
- **开源工具汇总**: [详细参考](016-motion-photo-opensource-kits)
- LimitPoint/LivePhoto (Swift): Apple Live Photo 创建/提取工具
- GoMoPho (Go): Google Motion Photo 视频提取器
- MotionPhotoMuxer (Python): Apple ↔ Google 双向转换
- MotionPhoto2: 创建 Google/Samsung Motion Photos
- makelive (Python): 创建 Apple Live Photos
- live-photo-js (JavaScript): Apple Live Photo 解析
- motion-photo-extractor: Samsung/Google 视频提取工具

### 格式参考
- ExifTool Huawei Tags: https://exiftool.org/TagNames/Huawei.html
- ExifTool XMP Tags: https://exiftool.org/TagNames/XMP.html#micro
- Google Motion Photo Format: https://android.googlesource.com/platform/packages/providers/MediaProvider/

### 技术博客
- Limit Point Blog: https://www.limit-point.com/blog/2018/live-photos/
- Aaron.cc FFmpeg HEVC: https://aaron.cc/ffmpeg-hevc-apple-devices/
- StackOverflow FFmpeg iPhone MOV: https://stackoverflow.com/questions/61390646

---

## 附录：关键技术要点汇总

### FFmpeg HEVC Apple 兼容性

**关键参数**：`-tag:v hvc1`

Apple 软件（Photos、QuickTime）要求 HEVC 使用 `hvc1` 标签，而非默认的 `hev1`。缺少此参数会导致 Live Photo 无法识别。

```bash
# MP4 转 MOV（Apple Live Photo 兼容）
ffmpeg -i input.mp4 -c:v libx265 -crf 28 -c:a aac -b:a 128k -tag:v hvc1 -f mov output.mov

# 仅重封装（不重编码）
ffmpeg -i input.mp4 -c copy -tag:v hvc1 output.mov
```

### Android Motion Photo Format 1.0 关键变更

**⚠️ `MicroVideoOffset` 已废弃**

根据 Android 动态照片格式 1.0 规范：
- ❌ `Camera:MicroVideoOffset` 已从规范中删除，必须忽略
- ✅ 视频定位改用 `GContainer:ItemLength`
- 视频起始位置 = `文件长度 - ItemLength(Video项)`

### HarmonyOS MOV 支持

**已确认不支持 MOV 封装格式**

根据 `OH_AVOutputFormat` 枚举定义，HarmonyOS 原生仅支持：
- `AV_OUTPUT_FORMAT_MPEG_4` (MP4)
- `AV_OUTPUT_FORMAT_AMR` (AMR)
- `AV_OUTPUT_FORMAT_ADTS` (ADTS/AAC)

**解决方案**：Apple Live Photo 输出需引入 FFmpeg 库处理 MOV 封装。

---

*报告更新于 2026-05-18*
*Apple MOV 编码要求已确认（hvc1 标签），HarmonyOS MOV 不支持已验证*
*华为 MovingPhoto API、开源工具参考已补充完整*