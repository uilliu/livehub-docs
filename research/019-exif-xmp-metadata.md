# 动态照片元数据结构总结

> **版本**：1.0
> **更新日期**：2026-05-21
> **目的**：系统化整理EXIF/XMP元数据结构和厂商自定义字段协议

---

## 一、EXIF元数据结构

### 1.1 IFD层级结构

```
JPEG/HEIC 文件结构
├── SOI (Start of Image) ──────────────────── 0xFFD9
├── APP1 (EXIF marker) ────────────────────── 0xFFE1
│   └── EXIF Header
│       ├── "Exif\x00\x00" 标识 (6字节)
│       ├── TIFF Header (8字节)
│       │   ├── Byte Order: "II" (小端) 或 "MM" (大端)
│       │   ├── Magic Number: 0x002A
│       │   └── IFD0 Offset: 通常是 0x0008
│       │
│       ├── IFD0 (主图像IFD) ──────────────── 基础图像属性
│       │   ├── ImageWidth (Tag: 0x0100)
│       │   ├── ImageLength (Tag: 0x0101)
│       │   ├── Make (Tag: 0x010F)
│       │   ├── Model (Tag: 0x0110)
│       │   ├── Orientation (Tag: 0x0112)
│       │   ├── DateTime (Tag: 0x0132)
│       │   ├── ExifIFD Pointer (Tag: 0x8769) ──→ 指向Exif IFD
│       │   ├── GPSInfo IFD Pointer (Tag: 0x8825) ──→ 指向GPS IFD
│       │   └── Next IFD Offset: 指向IFD1或0(结束)
│       │
│       ├── Exif IFD ──────────────────────── 拍摄参数
│       │   ├── ExposureTime (Tag: 0x829A)
│       │   ├── FNumber (Tag: 0x829D)
│       │   ├── ISOSpeedRatings (Tag: 0x8827)
│       │   ├── DateTimeOriginal (Tag: 0x9003)
│       │   ├── DateTimeDigitized (Tag: 0x9004)
│       │   ├── MakerNote (Tag: 0x927C) ──→ 厂商私有数据
│       │   ├── UserComment (Tag: 0x9286)
│       │   └── SubSecTime (Tag: 0x9290)
│       │
│       ├── GPS IFD ──────────────────────── GPS定位信息
│       │   ├── GPSLatitude (Tag: 0x0002)
│       │   ├── GPSLongitude (Tag: 0x0004)
│       │   ├── GPSAltitude (Tag: 0x0006)
│       │   └── GPSTimeStamp (Tag: 0x0007)
│       │
│       └── IFD1 (缩略图IFD) ─────────────── 缩略图信息
│           └── ThumbnailData (偏移指针)
│
├── 图像数据
└── [嵌入视频数据] (Motion Photo格式)
```

### 1.2 IFD Entry结构（每个条目12字节）

```
┌─────────────────────────────────────────────────────┐
│ Tag (2字节) │ Type (2字节) │ Count (4字节) │ Value/Offset (4字节) │
└─────────────────────────────────────────────────────┘
```

**数据类型码**：

| Type值 | 类型 | 大小 |
|--------|------|------|
| 1 | BYTE | 1字节 |
| 2 | ASCII | 1字节（含终止符） |
| 3 | SHORT | 2字节 |
| 4 | LONG | 4字节 |
| 5 | RATIONAL | 8字节（2个LONG） |
| 7 | UNDEFINED | 可变长度 |
| 9 | SLONG | 4字节（有符号） |
| 10 | SRATIONAL | 8字节（2个SLONG） |

### 1.3 MakerNotes结构

**MakerNotes在EXIF中的位置**：
- **Tag号**：0x927C (ExifIFD中的MakerNote)
- **数据类型**：UNDEFINED (Type=7)
- **长度**：厂商自定义（通常几百到几千字节）
- **偏移**：指向厂商私有数据区的起始位置

**各厂商头部格式**：

| 厂商 | 头部标识 | 版本位置 | 字节序处理 |
|------|---------|---------|-----------|
| Canon | "Canon" | 无显式版本 | 使用TIFF字节序 |
| Nikon | "Nikon\x00" + 版本 | 第7字节开始 | 内部可能使用不同字节序 |
| Sony | "SONY DSC " | 无显式版本 | 使用TIFF字节序 |
| Huawei | "HUAWEI" 或无标识 | 私有格式 | 使用TIFF字节序 |
| Samsung | "Samsung" | 无显式版本 | 使用TIFF字节序 |
| Apple | 无标识 | Apple Dictionary | 使用TIFF字节序 |

---

## 二、XMP元数据结构

### 2.1 XMP Packet结构

```
┌─────────────────────────────────────────────────────────┐
│                    XMP Packet                           │
├─────────────────────────────────────────────────────────┤
│  Header (23 bytes)                                      │
│  <?xpacket begin="?" id="W5M0MpCehiHzreSzNTczkc9d"?>   │
├─────────────────────────────────────────────────────────┤
│  Data Area (RDF/XML)                                    │
│  <x:xmpmeta xmlns:x="adobe:ns:meta/">                  │
│    <rdf:RDF xmlns:rdf="http://www.w3.org/...">         │
│      <rdf:Description ...>                              │
│        <!-- XMP Properties Here -->                     │
│      </rdf:Description>                                 │
│    </rdf:RDF>                                           │
│  </x:xmpmeta>                                           │
├─────────────────────────────────────────────────────────┤
│  Tailer (18 bytes)                                      │
│  <?xpacket end="w"?>  (or end="r" for read-only)       │
└─────────────────────────────────────────────────────────┘
```

### 2.2 XMP嵌入位置

| 文件格式 | 嵌入位置 | 说明 |
|---------|---------|------|
| JPEG | APP1段（0xFFE1标记后） | 在EXIF数据之后，或独立的APP1段 |
| HEIC/HEIF | `meta` box中的`iloc`项 | ISO Base Media File Format容器 |
| MOV/MP4 | `moov/udta/meta`或`moov/uuid` box | QuickTime/ISO BMFF结构 |
| PNG | `eXIf`或`iTXTt`块 | PNG规范扩展 |

### 2.3 标准命名空间URI对照表

| 前缀 | 命名空间URI | 说明 |
|-----|------------|------|
| dc: | `http://purl.org/dc/elements/1.1/` | Dublin Core元数据 |
| xmp: | `http://ns.adobe.com/xap/1.0/` | XMP基础属性 |
| xmpRights: | `http://ns.adobe.com/xap/1.0/rights/` | XMP版权信息 |
| exif: | `http://ns.adobe.com/exif/1.0/` | EXIF属性 |
| tiff: | `http://ns.adobe.com/tiff/1.0/` | TIFF属性 |
| photoshop: | `http://ns.adobe.com/photoshop/1.0/` | Photoshop属性 |

---

## 三、动态照片XMP命名空间

### 3.1 Google Camera命名空间

```
命名空间URI: http://ns.google.com/photos/1.0/camera/
前缀: GCamera:

属性定义:
┌─────────────────────────────────────────────────────────┐
│ 属性名                        │ 类型   │ 说明           │
├─────────────────────────────────────────────────────────┤
│ GCamera:MotionPhoto            │ Integer │ 是否为动态照片 │
│ GCamera:MotionPhotoVersion     │ Integer │ 格式版本号     │
│ GCamera:MotionPhotoPresentationTimestampUs │ Integer │ 展示时间戳(微秒) │
│ GCamera:MotionPhotoPlayback    │ String │ 播放模式       │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Google Container命名空间

```
命名空间URI: http://ns.google.com/photos/1.0/container/
前缀: GContainer:

属性定义:
┌─────────────────────────────────────────────────────────┐
│ 属性名                    │ 类型      │ 说明            │
├─────────────────────────────────────────────────────────┤
│ GContainer:Container      │ Structure │ 容器结构        │
│ GContainer:Directory      │ Array     │ Item结构数组    │
│ GContainer:Item:Length    │ Integer   │ 项长度(字节)    │
│ GContainer:Item:Mime      │ String    │ MIME类型        │
│ GContainer:Item:Semantic  │ String    │ 语义类型        │
└─────────────────────────────────────────────────────────┘

语义类型:
- "Primary"     主图像
- "MotionPhoto" 动态照片视频
- "Video"       视频数据
```

### 3.3 Samsung Micro命名空间

```
命名空间URI: http://ns.samsung.com/micro/1.0/
前缀: Micro:

属性定义:
┌─────────────────────────────────────────────────────────┐
│ 属性名                │ 类型    │ 说明                 │
├─────────────────────────────────────────────────────────┤
│ Micro:MicroVideoOffset │ Integer │ 视频数据偏移量(字节) │
│ Micro:MicroVideo       │ Boolean │ 是否为微视频         │
└─────────────────────────────────────────────────────────┘

注意: MicroVideoOffset已废弃，改用GCamera命名空间
```

---

## 四、厂商自定义字段协议

### 4.1 厂商字段对照表

| 厂商 | 格式 | 关键字段 | 字段位置 | 字段格式 |
|------|------|---------|---------|---------|
| Apple | Live Photo | ContentIdentifier | HEIC: MakerNotes Key 17; MOV: QuickTime metadata | UUID v4 |
| Apple | Live Photo | StillImageTime | MOV: QuickTime metadata | Integer (通常=0) |
| 华为(双框架) | Moving Photo | LIVE_xxxx | 文件末尾倒数20字节 | `LIVE_` + 4位数字 |
| 华为(双框架) | Moving Photo | Version & Frame Num | 文件末尾倒数40~60字节 | `v3_f31_c` 格式 |
| 华为(单框架) | Moving Photo | subtype | 数据库 Photos表 | Integer (值为3) |
| 华为 | Moving Photo | HwMotionPhoto | JPG MakerNotes | Integer (值为1) |
| 小米 | Live Photo | MotionPhoto | XMP-GCamera | Integer (值为1) |
| 小米 | Live Photo | MotionPhotoOffset | XMP-GCamera | Integer (偏移量) |
| Google | Motion Photo | Camera:MotionPhoto | XMP Camera | Integer (0或1) |
| Google | Motion Photo | GContainer:Directory | XMP GContainer | Item数组 |
| Samsung | Motion Photo | MotionPhotoOffset | XMP-GCamera | Integer (偏移量) |

### 4.2 Apple Live Photo字段详情

**ContentIdentifier字段**：
- **格式**：UUID v4 字符串
- **示例**：`A5E2B1D8-4C9A-4F3E-A8E7-123456789ABC`
- **HEIC位置**：EXIF MakerNotes Apple Dictionary，Key=17
- **MOV位置**：QuickTime元数据轨道，键为 `com.apple.quicktime.content.identifier`
- **用途**：图片+视频配对标识（共享相同UUID）

**StillImageTime字段**：
- **格式**：Integer
- **常见值**：`0` (视频起始帧) 或负数
- **MOV位置**：QuickTime元数据，键为 `com.apple.quicktime.still-image-time`
- **用途**：标记用户按下快门时的静态帧时间点

### 4.3 华为Moving Photo字段详情

**双框架末尾元数据结构**：
```
文件末尾60字节结构：
├── 倒数60-40字节: Version & Frame Num (如 "v3_f31_c")
│   v3：版本号
│   f31：封面帧号（第31帧）
│   _c：CinemaGraph标识（可选）
├── 倒数40-20字节: Sight tremble metadata (如 "0:0")
│   分号前：微动瞬间开始时间(ms)
│   分号后：微动瞬间结束时间(ms)
├── 倒数20-0字节: Video info metadata (如 "LIVE_12345")
│   LIVE_xxxx 表示视频长度（含Cinemagraph+Version）
```

**版本号规格**：

| 版本 | 规格说明 | TAG状态 |
|------|---------|--------|
| v1 | 720P老版本 | ❌ 无TAG |
| v2 | 4K版本 | ✅ 有TAG |
| v3 | 封面4K，视频1080P 75帧 | ✅ 有TAG，_c结尾 |
| v4 | 无TAG的4K版本 | ❌ 无TAG |
| v5 | 4.3 cepi动态照片 | ✅ 有TAG |
| v6 | 单框架拍摄 | ✅ 有TAG |
| v7 | iOS克隆转换 | ✅ 有TAG |

**单框架数据库字段**：

| 字段 | 类型 | 值 | 说明 |
|------|------|---|------|
| subtype | INT | 3 | 标识为MOVING_PHOTO |
| cover_position | BIGINT | 微秒us | 封面帧时间戳 |
| moving_photo_effect_mode | INT | 0-4 | 效果模式 |

### 4.4 小米/Google/Samsung字段详情

**XMP-GCamera命名空间字段**：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| MotionPhoto | Integer | 标识为动态照片（值=1） |
| MotionPhotoVersion | Integer | 动态照片格式版本 |
| MotionPhotoOffset | Integer | 视频数据偏移量（嵌入模式） |
| MotionPhotoPresentationTimestampUs | Integer | 静态帧展示时间（微秒） |

**关键发现**：
- 小米嵌入文件模式与Google Motion Photo完全兼容
- 使用相同的XMP-GCamera命名空间
- Samsung也使用XMP-GCamera命名空间（与Google兼容）

**废弃字段**：
- `MicroVideoOffset` ❌ 已废弃（Android动态照片格式1.0）
- 改用 `GContainer:ItemLength` 定位视频

---

## 五、视频定位算法

### 5.1 Google/Samsung格式（新规范）

```
视频起始位置 = 文件长度 - GContainer:ItemLength(Video项)
```

**GContainer:Directory结构**：
```
Container:Directory = [
  { Item:Semantic="Primary", Item:Length=<主图片长度> },
  { Item:Semantic="Video", Item:Length=<视频长度> }
]
```

### 5.2 华为双框架格式

```
视频起始位置 = 文件长度 - LIVE_xxxx值 - 60字节元数据
```

解析末尾元数据：
```python
def parse_huawei_metadata(file_path):
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)
        # 末尾20字节：LIVE_xxxx
        video_info = data[m-20:m].decode('utf-8')
        video_length = int(video_info[5:])
        # 倒数40-60字节：版本号
        version_info = data[m-60:m-40].decode('utf-8')
        # 视频起始位置
        video_start = m - video_length
```

### 5.3 Apple Live Photo

```
通过ContentIdentifier UUID配对HEIC和MOV文件
```

---

## 六、ExifTool常用命令

### 6.1 Apple Live Photo

```bash
# 读取ContentIdentifier
exiftool -ContentIdentifier image.HEIC
exiftool -QuickTime:ContentIdentifier video.MOV

# 写入ContentIdentifier
UUID="A5E2B1D8-4C9A-4F3E-A8E7-123456789ABC"
exiftool "-ContentIdentifier=$UUID" image.HEIC
exiftool "-ContentIdentifier=$UUID" video.MOV

# 读取StillImageTime
exiftool -QuickTime:StillImageTime video.MOV
```

### 6.2 华为动态照片

```bash
# 读取华为标签
exiftool -MakerNotes:HwMotionPhoto image.jpg
```

### 6.3 Google/Samsung Motion Photo

```bash
# 读取Motion Photo标签
exiftool -GCamera:MotionPhoto motion.jpg
exiftool -GCamera:MotionPhotoOffset motion.jpg
exiftool -GContainer:Directory motion.jpg

# 写入Motion Photo标签
exiftool "-GCamera:MotionPhoto=1" output.jpg
exiftool "-GCamera:MotionPhotoOffset=$offset" output.jpg
```

---

## 七、MakerNotes写入限制

| 平台 | MakerNotes写入 | 原因 | 替代方案 |
|------|---------------|------|---------|
| HarmonyOS | ❌ 不支持 | API无接口 | XMP自定义命名空间 |
| Android | ❌ 不支持 | MediaStore限制 | XMP-GCamera |
| iOS | ❌ 不支持 | ImageIO限制 | QuickTime元数据 |

**最佳实践**：
1. 读取：使用ExifTool或厂商特定解析器
2. 写入：使用XMP标准命名空间（GCamera兼容）
3. 跨平台：同时写入EXIF备份字段
4. 华为原生格式：无法创建（MakerNotes限制）

---

## 八、参考资料

| 文档编号 | 文档名称 | 关键内容 |
|---------|---------|---------|
| 013 | Android Motion Photo规范 | GContainer新规范 |
| 015 | Apple Live Photo规格 | MOV元数据要求 |
| 022 | EMUI动态照片格式 | 双框架末尾元数据 |
| 023 | HarmonyOS动态照片格式 | 单框架数据库字段 |
| 024 | 双单框架兼容性 | 转换机制 |
| 025 | 华为小米兼容性 | XMP字段对比 |
| 029 | 华为Apple兼容性 | ContentIdentifier机制 |

---
