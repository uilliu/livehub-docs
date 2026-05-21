# Android 动态照片格式 1.0 规范摘要

> 来源：https://developer.android.com/media/platform/motion-photo-format?hl=zh-cn
> 获取日期：2026-05-18
> 状态：官方规范（已获取）

---

## 一、概述

Android 动态照片格式 1.0 是 Google 定义的标准格式，用于将静态图片和视频合并为一个文件。

### 1.1 文件结构

```
[主静态图片][XMP 元数据][视频文件数据]
```

- **主静态图片**：JPEG 或 HEIC 格式
- **XMP 元数据**：包含 Camera 元数据和 Container 元数据
- **视频文件数据**：MP4/MOV 等格式，附加在图片末尾

---

## 二、XMP 元数据结构

### 2.1 两组 XMP 元数据

动态照片格式使用两组 XMP 元数据：

| 类型 | 前缀 | 说明 |
|-----|-----|-----|
| **Camera 元数据** | `Camera:` | 描述如何呈现主图片和视频 |
| **Container 元数据** | `Container:` 或 `GContainer:` | 定义后续媒体的顺序和属性 |

### 2.2 Camera 元数据属性

| 属性名 | 类型 | 说明 |
|-------|-----|-----|
| `Camera:MotionPhoto` | Integer | 0=不应视为动态照片；1=应视为动态照片 |
| `Camera:MotionPhotoVersion` | Integer | 格式版本号 |
| `Camera:MotionPhotoPresentationTimestampUs` | Integer | 静态帧展示时间（微秒），指定主图片在视频中的时间点 |

### 2.3 废弃的属性（微视频 V1 规范）

| 属性名 | 状态 | 说明 |
|-------|-----|-----|
| `Camera:MicroVideo` | ❌ **已废弃** | 必须忽略 |
| `Camera:MicroVideoVersion` | ❌ **已废弃** | 必须忽略 |
| `Camera:MicroVideoOffset` | ❌ **已废弃** | 已替换为 `GContainer:ItemLength` |
| `Camera:MicroVideoPresentationTimestampUs` | ❌ **已废弃** | 必须忽略 |

**关键发现**：`MicroVideoOffset` 已从此规范中删除，改用 `GContainer:ItemLength` 定位视频数据。

### 2.4 Container 元数据属性

| 属性名 | 说明 |
|-------|-----|
| `Container:Item` | 有序数组，定义容器布局和内容 |
| `Container:Directory` | Item 结构的有序数组 |

**命名空间**：
- 默认前缀：`Container`
- Google 前缀：`GContainer`

---

## 三、Item 元素结构

Container:Item 数组中的每个元素包含：

| 属性 | 说明 |
|-----|-----|
| `Item:Semantic` | 内容语义（如 "Primary"、"Video"） |
| `Item:Length` | 内容长度（字节） |
| `Item:URI` | 内容位置标识 |

### 3.1 示例布局

```
Container:Directory = [
  { Item:Semantic="Primary", Item:Length=<主图片长度> },
  { Item:Semantic="Video", Item:Length=<视频长度> }
]
```

---

## 四、视频定位方法

### 4.1 新方法（动态照片格式 1.0）

使用 `GContainer:ItemLength` 定位：

```
视频起始位置 = 文件长度 - GContainer:ItemLength(Video项)
```

### 4.2 旧方法（已废弃）

使用 `MicroVideoOffset`：

```
视频起始位置 = 文件长度 - MicroVideoOffset
```

**注意**：新规范明确要求**忽略** `MicroVideoOffset`，改用 Container 元数据。

---

## 五、对小米格式的意义

### 5.1 关键发现

1. **Android 动态照片格式 1.0 是官方标准**
2. **`MicroVideoOffset` 已废弃**，不应依赖此字段定位视频
3. **应使用 `GContainer:ItemLength`** 获取视频长度
4. **小米可能兼容旧版微视频 V1 规范**（使用 MicroVideoOffset）

## 六、技术建议

### 6.1 视频提取策略

**推荐策略**：先尝试新规范，后尝试旧规范

```typescript
function extractMotionPhotoVideo(filePath: string): ArrayBuffer | null {
  const xmp = getXmpData(filePath);

  // 方法1：动态照片格式 1.0（推荐）
  const containerItems = xmp?.['GContainer:Directory'];
  if (containerItems) {
    const videoItem = containerItems.find(item => item['Item:Semantic'] === 'Video');
    if (videoItem) {
      const videoLength = parseInt(videoItem['Item:Length']);
      const fileLength = getFileSize(filePath);
      return readFileRange(filePath, fileLength - videoLength, fileLength);
    }
  }

  // 方法2：微视频 V1 规范（兼容旧设备）
  const microVideoOffset = parseInt(xmp?.['Camera:MicroVideoOffset'] || '0');
  if (microVideoOffset > 0) {
    const fileLength = getFileSize(filePath);
    return readFileRange(filePath, fileLength - microVideoOffset, fileLength);
  }

  return null;
}
```

### 6.2 检测方法更新

```typescript
function isMotionPhoto(filePath: string): boolean {
  const xmp = getXmpData(filePath);

  // 检查新规范标识
  if (xmp?.['Camera:MotionPhoto'] === 1) return true;

  // 检查旧规范标识（兼容）
  if (xmp?.['Camera:MicroVideo'] === 1) return true;

  // 检查 GContainer 目录
  if (xmp?.['GContainer:Directory']) return true;

  return false;
}
```

---

## 七、参考资料

- [Android 动态照片格式 1.0](https://developer.android.com/media/platform/motion-photo-format?hl=zh-cn)
- [ISO 16684-1:2011(E) XMP 规范第 1 部分](https://github.com/adobe/XMP-Toolkit-SDK/blob/main/docs/XMPSpecificationPart1.pdf)
- [Adobe XMP 规范第 3 部分](https://github.com/adobe/XMP-Toolkit-SDK/blob/main/docs/XMPSpecificationPart3.pdf)
