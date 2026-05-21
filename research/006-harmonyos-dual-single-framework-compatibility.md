# 鸿蒙单双框架差异与兼容方案调研

> 文档类型：差异分析与兼容方案
> 适用系统：HarmonyOS NEXT（单框架） / EMUI / HarmonyOS 4（双框架）
> 更新日期：2026-05-19

---

## 1. 核心差异对比

### 系统版本与框架类型

| 系统版本 | 框架类型 | 存储方式 | 存储路径 |
|---------|---------|---------|---------|
| **EMUI** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS 4 及以前** | 双框架 | **嵌入文件模式** | `/storage/emulated/0/DCIM/Camera/` |
| **HarmonyOS NEXT** | 单框架 | **分离文件模式** | `/storage/media/100/local/files/Photo/` |

### 框架定义

**双框架**：Android + HarmonyOS 双系统架构
- 兼容 Android 应用
- 文件系统沿用 Android 模式（使用 `adb push` 命令）
- 动态照片采用**嵌入文件模式**（类似 Google/Samsung Motion Photo）

**单框架**：纯 HarmonyOS NEXT 架构
- 不兼容 Android 应用
- 使用原生 HarmonyOS 文件系统（使用 `hdc file send` + `mediatool send` 命令）
- 动态照片采用**分离文件模式**（JPG + MP4 独立文件）

---

## 2. 存储方式对比

### 双框架：嵌入文件模式

**文件结构**：

```
双框架动态照片文件结构（单一JPG文件）：
┌────────────────────────────────────────────────────────────┐
│ [0, m-n-40)        │ JPEG图片数据（封面帧）                 │
│ [m-n-40, m-p-60)   │ MP4视频数据                            │
│ [m-p-60, m-p-40)   │ CinemagraphInfo（可选）                │
│ [m-p-40, m-40)     │ Version & Frame Num（20字节）          │
│ [m-40, m-20)       │ Sight tremble metadata（20字节）       │
│ [m-20, m)          │ Video info metadata（20字节）          │
└────────────────────────────────────────────────────────────┘
```

**特点**：
- 单一文件，易于分享和传输
- 视频嵌入在JPEG文件末尾
- 通过末尾元数据标识和定位视频

### 单框架：分离文件模式

**文件结构**：

```
单框架存储目录结构：
|--Photo
|  |--0
|  |  |--A.jpg（效果图）
|  |  |--A.mp4（效果图）
|--.editData
|  |--Photo
|  |  |--0
|  |  |  |--A.jpg
|  |  |  |  |--origin_source.jpg（原始图）
|  |  |  |  |--origin_source.mp4（原始视频）
|  |  |  |  |--editdata
|  |  |  |  |--extraData（双框架来源的额外数据）
```

**特点**：
- 图片和视频独立存储
- 数据库通过 `subtype = 3` 关联
- 支持编辑可回退（存储原始文件）

---

## 3. API 差异对比

### 查询动态照片

| 功能 | 双框架 API | 单框架 API |
|-----|-----------|-----------|
| 查询资产 | MediaStore | PhotoAccessHelper |
| 检测动态照片 | 文件末尾元数据（LIVE_xxxx） | PhotoSubtype.MOVING_PHOTO |
| 过滤动态照片 | 无专用方法 | FetchOptions + predicates |

**双框架示例**：

```java
// 检测嵌入视频（读取文件末尾）
RandomAccessFile file = new RandomAccessFile(filePath, "r");
file.seek(file.length() - 20);
byte[] tailBytes = new byte[20];
file.read(tailBytes);
String tailString = new String(tailBytes);
boolean isLivePhoto = tailString.startsWith("LIVE_");
```

**单框架示例**：

```typescript
// 使用 PhotoSubtype 过滤
let predicates = new dataSharePredicates.DataSharePredicates();
predicates.equalTo('subtype', photoAccessHelper.PhotoSubtype.MOVING_PHOTO);
let fetchOptions = { fetchColumns: [], predicates: predicates };
let assetResult = await phAccessHelper.getAssets(fetchOptions);
```

### 读取动态照片内容

| 功能 | 双框架 API | 单框架 API |
|-----|-----------|-----------|
| 读取图片 | RandomAccessFile | MovingPhoto.requestContent(IMAGE_RESOURCE) |
| 读取视频 | 手动提取嵌入数据 | MovingPhoto.requestContent(VIDEO_RESOURCE) |
| 播放组件 | 无专用组件 | MovingPhotoView |

**双框架示例**：

```java
// 手动提取嵌入视频
RandomAccessFile file = new RandomAccessFile(filePath, "r");
long fileSize = file.length();
file.seek(fileSize - 20);
byte[] videoInfo = new byte[20];
file.read(videoInfo);
int videoLength = Integer.parseInt(new String(videoInfo).substring(5));
long videoStart = fileSize - videoLength - 40;
byte[] videoData = new byte[videoLength - 40];
file.seek(videoStart);
file.read(videoData);
```

**单框架示例**：

```typescript
// 使用 MovingPhoto API
let movingPhoto = await MediaAssetManager.requestMovingPhoto(context, asset, requestOptions, handler);
let imageData = await movingPhoto.requestContent(ResourceType.IMAGE_RESOURCE);
let videoData = await movingPhoto.requestContent(ResourceType.VIDEO_RESOURCE);
```

### 保存动态照片

| 功能 | 双框架 API | 单框架 API |
|-----|-----------|-----------|
| 保存方式 | 相机自动生成 | MediaAssetChangeRequest |
| 第三方应用 | 无法直接创建 | 支持创建动态照片 |
| 权限要求 | 相机权限 | READ_IMAGEVIDEO + WRITE_IMAGEVIDEO |

**双框架限制**：
- 第三方应用无法直接创建华为原生动态照片格式
- 需要写入 MakerNotes（HwMotionPhoto）+ 嵌入视频数据
- Android MediaStore API 不支持 MakerNotes 写入

**单框架支持**：

```typescript
// 第三方应用可创建动态照片
let changeRequest = MediaAssetChangeRequest.createAssetRequest(
  context, PhotoType.IMAGE, "jpg", { subtype: PhotoSubtype.MOVING_PHOTO }
);
changeRequest.addResource(ResourceType.IMAGE_RESOURCE, imageUri);
changeRequest.addResource(ResourceType.VIDEO_RESOURCE, videoUri);
await phAccessHelper.applyChanges(changeRequest);
```

---

## 4. 数据库字段对比

### Photos 表字段差异

| 字段 | 双框架 | 单框架 | 说明 |
|-----|--------|--------|-----|
| **subtype** | ❌ 无 | ✅ 有（值=3） | 动态照片标识 |
| **cover_position** | ❌ 无 | ✅ 有 | 封面帧时间戳(us) |
| **moving_photo_effect_mode** | ❌ 无 | ✅ 有 | 效果模式 |
| **media_extra_data** | ❌ 无 | ✅ 有 | 双框架额外信息（JSON） |
| **file_source_type** | ❌ 无 | ✅ 有 | 图片视频来源类型 |
| **storage_path** | ❌ 无 | ✅ 有 | 图片视频存储路径 |
| **original_subtype** | ❌ 无 | ✅ 有 | 原始照片类型 |
| **size** | 文件总大小 | image + video + extraData | 计算规则不同 |

### 数据库字段详细规格

**单框架新增字段**：

| 字段 | 含义 | 类型 | 动态照片规格 |
|-----|------|------|-------------|
| **subtype** | 照片子类型 | INT | 值为3（MOVING_PHOTO） |
| **cover_position** | 封面帧位置 | BIGINT | 动态照片封面帧对应的视频时间戳(us) |
| **moving_photo_effect_mode** | 效果模式 | INT | 表示效果模式（循环、来回、长曝光、多曝光） |
| **original_subtype** | 原始照片类型 | INT | 表示原始照片类型 |
| **media_extra_data** | 双框架额外信息 | TEXT | JSON格式存储 |
| **file_source_type** | 来源类型 | INT | MEDIA/FILE_MANAGER/PERIPHERAL/MEDIA_HO_LAKE |
| **storage_path** | 存储路径 | TEXT | 图片视频存储路径 |

**size字段计算规则**：
```
单框架动态照片size计算（与双框架保持一致）：

size = image文件大小 + video文件大小 + extraData文件大小

目的：确保单框架与双框架手机显示的大小一致
场景：克隆判断文件是否重复时，size需在1M误差以内
```

### media_extra_data字段内容

**JSON格式**：
```json
{
  "CinemagraphInfo": "data",      // CinemagraphInfo数据（字符串或需改为blob）
  "VersionAndFrameNum": "v3_f31_c", // 版本号和封面帧号
  "SightTrembleMetadata": "0:0"    // 微动瞬间播放信息
}
```

---

## 5. 格式转换机制详解

### 5.1 双框架 → 单框架转换

**转换场景**：数据克隆（双框架手机迁移到单框架手机）、端云下载

**转换流程**：

```
双框架嵌入格式 → 单框架分离格式

输入：单一JPG文件（含嵌入视频）
├── 解析末尾元数据（LIVE_xxxx）
├── 解析Version & Frame Num（v3_f31_c）
├── 解析CinemagraphInfo（可选）
├── 解析Sight tremble metadata
├── 提取JPEG图片数据 [0, m-n-40)
├── 提取MP4视频数据 [m-n-40, m-p-60)
└── 解析封面帧时间戳（帧号→pts）

输出：
├── IMG_XXXX.jpg（封面）
├── IMG_XXXX.mp4（视频）
├── extraData（额外信息，存储在.editData目录）
└── 数据库记录：
    ├── subtype = 3（MOVING_PHOTO）
    ├── size = jpg + mp4 + extraData总大小
    ├── cover_position = 封面帧时间戳(us)
    └── media_extra_data = JSON格式的额外信息
```

**文件拆分算法**：
```python
def split_livephoto(file_path):
    """双框架动态照片 → 分离格式"""
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)  # 文件总长度
        
        # 解析末尾元数据
        video_info = data[m-20:m]  # "LIVE_xxxx"
        video_length = int(video_info[5:])  # xxxx
        
        # Sight tremble metadata
        sight_tremble = data[m-40:m-20]  # "xx:xx"
        
        # Version & Frame Num
        version_frame = data[m-60:m-40]  # "v3_f31_c"
        
        # 检查是否有CinemagraphInfo
        has_cinema = version_frame.endswith('_c')
        if has_cinema:
            cinema_length_bytes = data[m-64:m-60]
            cinema_length = int.from_bytes(cinema_length_bytes, 'big')
            p = cinema_length
        else:
            p = 0
        
        # 计算边界
        n = video_length
        image_end = m - n - 40
        video_start = m - n - 40
        video_end = m - p - 60
        
        # 提取内容
        image_data = data[0:image_end]
        video_data = data[video_start:video_end]
        
        return {
            'image': image_data,
            'video': video_data,
            'extra_data': {
                'video_info': video_info,
                'sight_tremble': sight_tremble,
                'version_frame': version_frame,
                'cinema_data': data[m-p-60:m-60] if has_cinema else None
            }
        }
```

### 5.2 单框架 → 双框架转换

**转换场景**：华为分享跨端分享（单框架手机分享给双框架手机）、端云上传

**转换流程**：

```
单框架分离格式 → 双框架嵌入格式

输入：JPG + MP4 两个独立文件
├── 读取JPG封面数据
├── 读取MP4视频数据
├── 判断来源（单框架/双框架）
├── 单框架来源：构造简单元数据（无CinemagraphInfo）
│   ├── Sight tremble: 0:0
│   ├── Version: v6（无_c）
│   └── LIVE_<视频长度+20>
├── 双框架来源：从extraData读取原始元数据
└── 组合为单一JPG文件

输出：单一JPG文件（含嵌入视频 + 末尾元数据）
```

**单框架来源转换**：
```python
def create_dual_format_from_single(image_data, video_data):
    """单框架来源 → 双框架嵌入格式"""
    # 构造元数据
    sight_tremble = "0:0"
    version_frame = "v6_f0"  # 无_c结尾（无CinemaGraph）
    video_length = len(video_data) + 40
    video_info = f"LIVE_{video_length:04d}"
    
    # 组合文件
    combined_data = image_data + video_data
    combined_data += sight_tremble.encode()  # 20字节
    combined_data += version_frame.encode()  # 20字节
    combined_data += video_info.encode()  # 20字节
    
    return combined_data
```

**双框架来源转换**：
```python
def create_dual_format_from_dual(image_data, video_data, extra_data):
    """双框架来源 → 双框架嵌入格式（无损转换）"""
    # 从extraData读取原始元数据
    sight_tremble = extra_data['sight_tremble']
    version_frame = extra_data['version_frame']
    cinema_data = extra_data['cinema_data']
    
    video_length = len(video_data) + len(cinema_data) + 40 + 4
    video_info = f"LIVE_{video_length:04d}"
    
    # 组合文件
    combined_data = image_data + video_data
    if cinema_data:
        combined_data += cinema_data
    combined_data += version_frame.encode()
    combined_data += sight_tremble.encode()
    combined_data += video_info.encode()
    
    return combined_data
```

### 5.3 HEIF动态照片转换

**HEIF格式特点**：
- 支持HEIF格式动态照片的单双转换
- 单框架拍摄HEIF动态照片版本号为v6
- 出端场景（上云、分享）需合并为双框架格式

**转换要点**：
- 合并时补齐字段：Video info metadata、Sight tremble metadata、Version & Frame Num
- 验收标准：端云和分享场景支持HEIF动态照片由多文件融合为单文件

### 5.4 端云同步转换

| 方向 | 转换逻辑 |
|------|----------|
| **上传** | 调用`IsMovingPhoto`判断 → 调用`ConvertToLivePhoto`合并为双框架 → 上传云空间 |
| **下载** | 判断`fileType=9` → 调用`ConvertToMovingPhoto`拆分为单框架 → 保存本地 |

### 5.5 转换函数API

```cpp
// movingPhoto → livePhoto（移动到文管）
int32_t MovingPhotoFileUtils::ConvertToLivePhoto(
    const string& movingPhotoImagepath, 
    int64_t coverPosition,
    std::string &livePhotoPath, 
    int32_t userId
);

// livePhoto → movingPhoto（移动到图库）
int32_t MovingPhotoFileUtils::ConvertToMovingPhoto(
    const std::string &livePhotoPath, 
    const string &movingPhotoImagePath,
    const string &movingPhotoVideoPath, 
    const string &extraDataPath
);

// 判断是否为动态照片
bool MovingPhotoFileUtils::IsMovingPhoto(const string& path);

// 判断是否为livePhoto
bool MovingPhotoFileUtils::IsLivePhoto(const string& path);
```

---

## 6. 分享跨端对比

### 华为分享跨端流程

| 场景 | 发送方 | 接收方 | 处理方式 |
|-----|-------|-------|---------|
| 双→双 | 双框架 | 双框架 | 直接传输单一JPG文件 |
| 双→单 | 双框架 | 单框架 | 接收方自动拆分存储 |
| 单→双 | 单框架 | 双框架 | 媒体库融合为嵌入格式 |
| 单→单 | 单框架 | 单框架 | 保持分离格式 |

### 媒体库扫描识别逻辑

**双框架接收方**：
- 直接支持双框架格式
- 图库识别末尾元数据并播放

**单框架接收方**：
- 媒体库扫描识别末尾元数据（LIVE_xxxx）
- 自动执行 ConvertToMovingPhoto 拆分逻辑
- 存储为分离格式：JPG + MP4 + extraData

---

## 7. 文件导入方式对比

### 双框架导入

```bash
# adb push 命令
adb push IMG_XXXX.jpg /storage/emulated/0/DCIM/Camera/
adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE \
    -d file:///storage/emulated/0/DCIM/Camera/IMG_XXXX.jpg
```

### 单框架导入

```bash
# hdc + mediatool 命令
hdc file send IMG_XXXX.jpg /storage/media/100/local/files/Pictures/
hdc file send IMG_XXXX.mp4 /storage/media/100/local/files/Pictures/
hdc shell mediatool send /storage/media/100/local/files/Pictures/
```

---

## 8. 编辑功能对比

### 双框架编辑

- 图库编辑动态照片时，直接修改嵌入文件
- 无编辑可回退机制
- 编辑后文件仍为嵌入格式

### 单框架编辑

- 支持编辑可回退
- 存储原始文件（origin_source.jpg/mp4）
- 存储编辑数据（editdata）
- 可还原到初始设置（revertToOriginal）

---

## 9. 动态照片版本功能对比

### 版本演进对比表

| 功能维度 | 动态照片1.0 | 动态照片2.0 | 动态照片3.0 |
|---------|------------|------------|------------|
| **核心功能** | 720P动态照片<br>封面帧为单帧处理<br>只在普通模式生效 | 连续拍照帧编码<br>视频大小为照片size<br>支持封面优选帧 | 封面帧质量提升<br>视频支持1080P<br>支持封面帧重选<br>照片/视频二阶段处理<br>场景支持：普通/人像/闪拍<br>首尾帧去模糊处理 |
| **可见条件** | 1. 支持ZSL<br>2. ro.hwcamera.livephoto_enable=true | 1. 支持ZSL<br>2. livephoto_enable=true<br>3. livePhotoBestMomentSupported=1 | 1. 支持ZSL<br>2. livephoto_enable=true<br>3. livePhotoBestMomentAvailableResolutions上报size（双数） |
| **下参配置** | livePhotoMode=1<br>aeTargetFpsRange=[24,24] | livePhotoMode=1<br>livePhotoBestMomentMode=1<br>aeTargetFpsRange=[24,24]<br>frameDuration=41666667L | livePhotoMode=1<br>aeTargetFpsRange=[30,30]<br>hwFetchMoreFFD=1 |
| **编码策略** | 缓存1.1s image+audio<br>拍照后编码1.1s<br>总计约2.2s | image缓存3帧<br>onCaptureStarted后编码40帧<br>audio缓存1.1s | image缓存30帧<br>拍照后编码30帧+后续45帧<br>共75帧（约2.5s） |
| **保存格式** | JPEG+MP4写入同一文件<br>末尾写入startTime、timeDuration、tag | JPEG+MP4写入同一文件<br>末尾写入版本、startTime、timeDuration、tag | JPEG+MP4写入同一文件<br>中间写入每帧cinemagraphData<br>末尾写入版本、startTime、timeDuration、tag |

### 版本号规格

| 版本 | 规格说明 | 是否有TAG | 特点 |
|-----|---------|----------|------|
| **v1** | 720P老版本 | ❌ 无版本号及封面帧号tag | 视频分辨率720P |
| **v2** | 4K版本 | ✅ 有版本号及封面帧号tag | 封面和视频均为4K，版本号=2 |
| **v3** | 封面4K，视频1080P 75帧 | ✅ 有版本号及封面帧号tag，_c结尾 | 支持CinemaGraph |
| **v4** | 无TAG的4K版本 | ❌ 无版本号及封面帧号tag | 需通过其他方式识别 |
| **v5** | 4.3 cepi动态照片 | ✅ 有版本号及封面帧号tag | 特殊版本 |
| **v6** | 单框架拍摄的动态照片 | ✅ 有版本号及封面帧号tag | 单框架默认版本 |
| **v7** | iOS克隆转换版本 | ✅ 有版本号及封面帧号tag | 能力不支持标记 |

### 单框架版本

- 版本号v6为单框架默认版本
- 通过数据库字段记录信息
- 支持多种视频格式（mp4/mov/ts）
- iOS克隆版本为v7（能力不支持标记）

---

## 10. 对 LiveHub 的影响

### 格式检测逻辑调整

```python
def detect_huawei_moving_photo(file_path, device_info):
    # 1. 判断设备框架类型
    if device_info.os_version in ['EMUI', 'HarmonyOS 4']:
        framework_type = 'dual'
    elif device_info.os_version == 'HarmonyOS NEXT':
        framework_type = 'single'
    else:
        # 根据文件系统路径判断
        if '/storage/emulated/0/' in file_path:
            framework_type = 'dual'
        elif '/storage/media/100/' in file_path or '/storage/cloud/' in file_path:
            framework_type = 'single'
    
    # 2. 根据框架类型选择检测方法
    if framework_type == 'dual':
        return detect_dual_framework_livephoto(file_path)
    else:
        return detect_single_framework_movingphoto(file_path)
```

### 转换逻辑调整

```python
def convert_huawei_to_apple(file_path, device_info):
    detection = detect_huawei_moving_photo(file_path, device_info)
    
    if detection['framework'] == 'dual':
        # 双框架：提取嵌入视频
        result = convert_livephoto_to_movingphoto(file_path)
        image_data = result['image']
        video_data = result['video']
    else:
        # 单框架：直接读取分离文件
        with open(file_path, 'rb') as f:
            image_data = f.read()
        with open(detection['video_path'], 'rb') as f:
            video_data = f.read()
    
    # MP4 → MOV转换（需FFmpeg）
    mov_data = convert_mp4_to_mov(video_data)
    
    return {'image': image_data, 'video': mov_data}
```

---

## 参考资料

### 相关文档
- `harmonyos-movingphoto-api.md` - HarmonyOS动态照片API规格
- `emui-movingphoto-api.md` - EMUI动态照片API规格
- `harmonyos-movingphoto-format.md` - HarmonyOS动态照片格式分析
- `emui-movingphoto-format.md` - EMUI动态照片格式分析
