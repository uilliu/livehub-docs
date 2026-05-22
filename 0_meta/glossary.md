# LiveHub 统一术语表

> 版本：1.0
> 更新日期：2026-05-18
> 目的：确保项目中所有文档、代码、界面用语统一，消除歧义

---

## 一、核心概念命名

### 1.1 照片类型统一命名

| 厂商 | 中文名称 | 英文名称 | 英文代码标识 | 文件模式 | 说明 |
|------|---------|---------|------------|---------|------|
| **Apple** | 实况照片 | **Live Photo** | `LIVE_PHOTO` | 分离文件 | 官方术语 |
| **华为** | 动态照片 | **Moving Photo** | `MOVING_PHOTO` | 分离文件 | HarmonyOS官方API命名 |
| **小米** | 实况照片 | **Live Photo** | `XIAOMI_LIVE_PHOTO` | 分离/嵌入 | HyperOS术语 |
| **Google** | 动态照片 | **Motion Photo** | `MOTION_PHOTO` | 嵌入文件 | Android官方规范 |
| **Samsung** | 动态照片 | **Motion Photo** | `SAMSUNG_MOTION_PHOTO` | 嵌入文件 | MVIMG格式 |
| **通用（本项目）** | 动态照片 | **Motion Photo** | — | — | 作为通用术语使用 |

### 1.2 术语选择原则

**中文术语**：
- **动态照片**：用于通用描述和技术文档（华为、Google、Samsung 统称）
- **实况照片**：用于 Apple、小米特定描述（遵循厂商官方命名）

**英文术语**：
- **Live Photo**：仅用于 Apple 和小米（厂商官方）
- **Motion Photo**：用于 Google、Samsung 及通用技术描述
- **Moving Photo**：用于华为特定描述（HarmonyOS API命名）

**代码标识**：
- 枚举值使用大写下划线命名：`LIVE_PHOTO`, `MOVING_PHOTO`, `MOTION_PHOTO`
- 前缀区分厂商：`XIAOMI_LIVE_PHOTO`, `SAMSUNG_MOTION_PHOTO`

---

## 二、文件模式命名

### 2.1 文件结构模式

| 模式类型 | 中文名称 | 英文名称 | 英文代码标识 | 说明 |
|---------|---------|---------|------------|------|
| 双物理文件 | **分离文件模式** | **Separate-File Mode** | `SEPARATE_FILE` | 图片+视频为独立物理文件 |
| 单物理文件 | **嵌入文件模式** | **Embedded-File Mode** | `EMBEDDED_FILE` | 视频嵌入图片文件末尾 |

### 2.2 模式适用厂商

| 厂商 | 模式 | 文件结构 | 示例 |
|------|------|---------|------|
| **Apple** | 分离文件模式 | HEIC + MOV | `IMG_001.HEIC` + `IMG_001.MOV` |
| **华为** | 分离文件模式 | JPG + MP4 | `IMG_001.JPG` + `IMG_001.MP4` |
| **小米** | 分离文件模式 / 嵌入文件模式 | JPG + MP4 / MVIMG_JPG | 取决于机型和设置 |
| **Google** | 嵌入文件模式 | JPG（内含MP4） | `IMG_001.JPG`（文件末尾嵌入视频） |
| **Samsung** | 嵌入文件模式 | MVIMG_JPG（内含MP4） | `MVIMG_001.JPG` |

---

## 三、厂商命名规范

### 3.1 厂商中英文对照

| 厂商 | 中文名称 | 英文名称 | 英文代码标识 | 系统名称 |
|------|---------|---------|------------|---------|
| Apple | 苹果 | **Apple** | `APPLE` | iOS |
| 华为 | 华为 | **Huawei** | `HUAWEI` | EMUI/HarmonyOS |
| 小米 | 小米 | **Xiaomi** | `XIAOMI` | MIUI/HyperOS |
| Google | 谷歌 | **Google** | `GOOGLE` | Android |
| Samsung | 三星 | **Samsung** | `SAMSUNG` | Android (OneUI) |

### 3.2 系统命名规范

| 系统 | 中文名称 | 英文名称 | 代码标识 | 备注 |
|------|---------|---------|---------|
| Apple iOS | iOS | **iOS** | `IOS` | - |
| HUAWEI EMUI | 华为情感化操作系统 | **EMUI** | `EMUI` | 华为基于AOSP深度定制 |
| HarmonyOS | 鸿蒙系统(双框架) | **HarmonyOS** | `HARMONY_OS` | 同时支持 AOSP和华为自研宏内核两大框架的操作系统 |
| HarmonyOS NEXT| 鸿蒙系统(单框架) | **HarmonyOS Next** | `HARMONY_OS_NEXT` | 华为自研宏内核的操作系统 |
| Xiaomi HyperOS | 小米澎湃OS | **HyperOS** | `HYPER_OS` | - |
| Xiaomi MIUI | 小米MIUI | **MIUI** | `MIUI` | 小米基于AOSP深度定制 |
| Android | 安卓系统 | **Android** | `ANDROID` | - |

---

## 四、元数据术语

### 4.1 核心元数据概念

| 中文术语 | 英文术语 | 代码标识 | 说明 |
|---------|---------|---------|------|
| 内容标识符 | **Content Identifier** | `CONTENT_IDENTIFIER` | Apple/华为用于配对的UUID |
| 静态帧时间 | **Still Image Time** | `STILL_IMAGE_TIME` | Apple MOV中标记主帧时间点 |
| 动态照片偏移 | **Motion Photo Offset** | `MOTION_PHOTO_OFFSET` | Google/Samsung 视频偏移量（已废弃） |
| 容器项目长度 | **Container Item Length** | `CONTAINER_ITEM_LENGTH` | Android 1.0规范视频定位（新） |
| 动态照片标识 | **Motion Photo Flag** | `MOTION_PHOTO_FLAG` | 标识文件为动态照片 |

### 4.2 元数据命名空间

| 厂商 | 命名空间前缀 | URI | 说明 |
|------|------------|-----|------|
| Apple | `QuickTime` | Apple私有 | MOV视频元数据 |
| 华为 | `MakerNotes:Hw` | 华为私有 | MakerNotes嵌入（双框架） |
| 小米 | `GCamera` | `http://ns.google.com/photos/1.0/camera/` | 与Google Motion Photo兼容 |
| Google | `GCamera` | `http://ns.google.com/photos/1.0/camera/` | Camera元数据（Motion Photo标准） |
| Google | `GContainer` | Google私有 | Container元数据 |
| Samsung | `Micro` | Samsung私有 | 微视频标记 |

**注意**：
- 小米、Google使用相同的XMP-GCamera命名空间，格式完全兼容
- 华为双框架使用末尾元数据（非XMP），华为单框架使用数据库关联（subtype=3）
- 建议使用标准XMP-GCamera命名空间，无需自定义命名空间

### 4.3 废弃字段说明

| 字段 | 状态 | 替代方案 | 来源 |
|------|------|---------|------|
| `MicroVideoOffset` | ❌ **已废弃** | `GContainer:ItemLength` | Android Motion Photo 1.0规范 |
| `MicroVideo` | ❌ **已废弃** | `Camera:MotionPhoto` | Android Motion Photo 1.0规范 |

---

## 五、应用界面用语

### 5.1 功能名称（面向用户）

| 功能 | 中文界面显示 | 英文界面显示 | 说明 |
|------|-------------|-------------|------|
| 格式转换 | 转换 | Convert | 核心功能 |
| 批量转换 | 批量转换 | Batch Convert | 批量操作 |
| 选择照片 | 选择照片 | Select Photos | 照片选择 |
| 选择输出格式 | 输出格式 | Output Format | 格式选择 |
| 转换进度 | 转换进度 | Progress | 进度显示 |
| 转换完成 | 完成 | Completed | 完成提示 |

### 5.2 格式显示名称（面向用户）

| 格式 | 中文界面显示 | 英文界面显示 | 说明 |
|------|-------------|-------------|------|
| Apple Live Photo | Apple实况照片 | Apple Live Photo | 用户可见选项 |
| 华为动态照片 | 华为动态照片 | Huawei Moving Photo | 用户可见选项 |
| 小米实况照片 | 小米实况照片 | Xiaomi Live Photo | 用户可见选项 |
| Google动态照片 | Google动态照片 | Google Motion Photo | 用户可见选项 |
| Samsung动态照片 | Samsung动态照片 | Samsung Motion Photo | 用户可见选项 |
| 嵌入文件格式（通用） | 图片+视频（通用） | Embedded Photo (Universal) | 兼容性更好的兜底选项 |

**注意**：
- 嵌入文件模式在小米、Google、华为双框架上兼容性更好（推荐兜底方案）
- 分离文件模式仅在华为单框架、Apple上兼容性好

---

## 六、代码命名规范

### 6.1 枚举定义示例

```typescript
// 格式类型枚举
enum MotionPhotoFormat {
  UNKNOWN = 0,
  STATIC_IMAGE = 1,
  
  // 分离文件模式
  APPLE_LIVE_PHOTO = 2,       // Apple实况照片
  HUAWEI_MOVING_PHOTO = 3,    // 华为动态照片
  XIAOMI_LIVE_PHOTO_SEP = 4,  // 小米实况照片（分离文件）
  
  // 嵌入文件模式
  XIAOMI_LIVE_PHOTO_EMB = 5,  // 小米实况照片（嵌入文件）
  GOOGLE_MOTION_PHOTO = 6,    // Google动态照片
  SAMSUNG_MOTION_PHOTO = 7,   // Samsung动态照片
}

// 文件模式枚举
enum FileMode {
  SEPARATE_FILE = 'separate',  // 分离文件模式
  EMBEDDED_FILE = 'embedded',  // 嵌入文件模式
}

// 厂商枚举
enum Manufacturer {
  APPLE = 'apple',
  HUAWEI = 'huawei',
  XIAOMI = 'xiaomi',
  GOOGLE = 'google',
  SAMSUNG = 'samsung',
}
```

### 6.2 类/模块命名示例

```typescript
// 检测器
class FormatDetector { }      // 格式检测器
class HuaweiMovingPhotoDetector { }  // 华为动态照片检测器

// 转换器
class MotionPhotoConverter { }       // 动态照片转换器
class AppleLivePhotoConverter { }    // Apple实况照片转换器

// 元数据处理
class MetadataReader { }     // 元数据读取器
class MetadataWriter { }     // 元数据写入器
class XmpProcessor { }       // XMP处理器
```

---

## 七、文档用语规范

### 7.1 技术文档用语

| 场景 | 推荐用语 | 禁用语 | 说明 |
|------|---------|-------|------|
| 文件结构描述 | 分离文件模式 / 嵌入文件模式 | 双文件 / 单文件 | 消除歧义 |
| 厂商格式描述 | 华为动态照片 / Apple实况照片 | 华为Live Photo | 使用官方命名 |
| 元数据描述 | MakerNotes / XMP命名空间 | 私有标签 / 自定义标签 | 使用技术术语 |

### 7.2 用户文档用语

| 场景 | 推荐用语 | 说明 |
|------|---------|------|
| 功能介绍 | 动态照片转换 | 通用术语，易懂 |
| 格式列表 | 各品牌动态照片格式 | 避免技术术语 |
| 操作指引 | 选择照片 → 选择格式 → 转换 | 步骤清晰 |

---

## 九、术语对照速查表

### 中文→英文

| 中文 | 英文 | 适用场景 |
|------|------|---------|
| 实况照片 | Live Photo | Apple、小米 |
| 动态照片 | Motion Photo | 通用、Google、Samsung |
| 动态照片 | Moving Photo | 华为特定 |
| 分离文件模式 | Separate-File Mode | 文件结构描述 |
| 嵌入文件模式 | Embedded-File Mode | 文件结构描述 |
| 内容标识符 | Content Identifier | 元数据 |
| 静态帧时间 | Still Image Time | 元数据 |

### 英文→中文

| 英文 | 中文 | 适用场景 |
|------|------|---------|
| Live Photo | 实况照片 | Apple、小米 |
| Motion Photo | 动态照片 | 通用、Google、Samsung |
| Moving Photo | 动态照片 | 华为特定 |
| Separate-File Mode | 分离文件模式 | 文件结构描述 |
| Embedded-File Mode | 嵌入文件模式 | 文件结构描述 |
| Content Identifier | 内容标识符 | 元数据 |
| Still Image Time | 静态帧时间 | 元数据 |

---

*统一术语表 v1.0*
*2026-05-18*