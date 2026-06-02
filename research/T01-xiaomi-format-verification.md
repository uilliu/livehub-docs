# 小米实况照片格式验证方案


> **验证类别**：格式分析
> **验证工具**：ExifTool（已安装至 `D:\Tools\exiftool`）
> **验证状态**：待执行（需用户提供小米实况照片文件）

---

## 一、验证目标

确定小米实况照片的实际文件结构和元数据字段，为华为↔小米转换方案提供技术依据。

### 1.1 需确认的核心问题

| 问题 | 当前状态 | 验证后目标 |
|-----|---------|----------|
| 文件结构模式 | 推测（分离文件/嵌入文件） | 实测确认 |
| XMP命名空间 | 推测（XMP-micro/XMP-GCamera） | 实测确认字段列表 |
| 与Google Motion Photo兼容性 | 推测 | 实测对比 |

---

## 二、前置条件

### 2.1 ExifTool 已就绪

**安装位置**：`D:\Tools\exiftool\exiftool.exe`

**验证安装**：
```powershell
D:\Tools\exiftool\exiftool.exe -ver
# 预期输出：13.xx（版本号）
```

### 2.2 用户需提供

| 信息项 | 示例 | 要求 |
|-------|------|-----|
| 小米实况照片文件路径 | `D:\Photos\小米\IMG_xxx.jpg` | 必须是小米手机拍摄的实况照片 |
| 是否有配套视频文件 | 同目录下的 `.mp4` 文件 | 如有，一并提供路径 |

**如何确认是小米实况照片**：
- 在小米手机相册中，实况照片有"动态"标识
- 长按图片会有动态效果播放
- 导出后文件名通常为 `IMG_` 或 `MVIMG_` 开头

---

## 三、分析执行步骤

### 3.1 文件结构分析

**用户提供路径后，我将执行**：

```powershell
# 命令1：检查目录结构
dir "<小米照片目录>" /b
dir "<小米照片目录>" /b | findstr "IMG_ MVIMG_ .mp4 .jpg"

# 预期输出解读：
# - 如同时存在 IMG_xxx.jpg 和 IMG_xxx.mp4 → 分离文件模式
# - 如仅有 MVIMG_xxx.jpg（无配套mp4） → 嵌入文件模式
```

### 3.2 元数据提取

```powershell
# 设置 ExifTool 路径
$EXIFTOOL = "D:\Tools\exiftool\exiftool.exe"

# 命令2：提取所有元数据
& $EXIFTOOL -a -G -s "<小米图片路径>" > xiaomi-metadata-raw.txt

# 命令3：重点提取 XMP 标签
& $EXIFTOOL -XMP:all "<小米图片路径>" > xiaomi-xmp.txt

# 命令4：检查 Motion Photo 相关标签
& $EXIFTOOL -XMP-GCamera:all "<小米图片路径>" > xiaomi-motion.txt

# 命令5：检查小米特定命名空间
& $EXIFTOOL -XMP-micro:all "<小米图片路径>" > xiaomi-micro.txt

# 命令6：检查 MakerNotes（如有）
& $EXIFTOOL -MakerNotes:all "<小米图片路径>" > xiaomi-makernotes.txt
```

### 3.3 关键标签对照表（已确认）

**基于 ExifTool XMP-GCamera 命名空间的官方标签列表**：

| 标签名 | 类型 | 命名空间 | 存在则说明 |
|-------|-----|---------|----------|
| `MotionPhoto` | Integer | XMP-GCamera | 嵌入文件模式，值为 1 |
| `MotionPhotoVersion` | Integer | XMP-GCamera | 动态照片格式版本 |
| `MotionPhotoOffset` | Integer | XMP-GCamera | 视频嵌入偏移量（用于提取） |
| `MotionPhotoPresentationTimestampUs` | Integer | XMP-GCamera | 动态播放时间点（微秒） |
| `MicroVideo` | Integer | XMP-GCamera | 微视频标识（旧版标签） |
| `MicroVideoOffset` | Integer | XMP-GCamera | 视频数据偏移量（嵌入模式） |
| `MicroVideoVersion` | Integer | XMP-GCamera | 微视频格式版本 |
| `MicroVideoPresentationTimestampUs` | Integer | XMP-GCamera | 展示时间戳（微秒） |

**关键结论**：
- 小米嵌入文件模式与 **Google Motion Photo 完全兼容**
- 使用相同的 `XMP-GCamera` 命名空间
- `MicroVideoOffset` = 文件总长度 - 视频数据起始位置
- 视频提取公式：从 `(文件长度 - MicroVideoOffset)` 字节开始，到文件末尾

**可能不存在的小米特有标签**（需实测验证）：
- `MicroDescription` - 小米相机描述（推测）
- `MicroVersion` - MIUI相机版本（推测）
- **注**：ExifTool 的 `XMP-micro` 命名空间为空，表明这些标签可能未标准化

### 3.4 嵌入视频提取（如为嵌入文件模式）

```powershell
# 获取视频偏移量
& $EXIFTOOL -XMP-GCamera:MotionPhotoOffset "<小米图片路径>"
# 输出示例：Motion Photo Offset : 1234567

# 如需提取嵌入视频，使用 Python 脚本
python extract_embedded_video.py "<小米图片路径>" xiaomi-extracted.mp4
```

---

## 四、输出报告模板

**报告文件**：`003-xiaomi-format-analysis.md`

```markdown
# 小米实况照片格式分析报告

> 分析日期：2026-05-18
> 分析对象：<实际文件路径>

## 1. 文件结构结论

- **模式**：分离文件 / 嵌入文件
- **文件组成**：
  - 图片：<文件名>.jpg
  - 视频：<文件名>.mp4（如分离模式）

## 2. XMP 元数据字段列表

| 标签名 | 命名空间 | 值 | 说明 |
|-------|---------|---|-----|

## 3. 与 Google Motion Photo 对比

- 兼容性：完全兼容 / 部分兼容 / 不兼容
- 差异字段：<具体差异>

## 4. 转换方案建议

### 小米 → 华为
- 步骤：<具体操作>
- 技术难点：<难点分析>

### 小米 → Apple
- 步骤：<具体操作>
- 关键依赖：FFmpeg MP4→MOV 转换
```

---

## 五、下一步行动

**用户需执行**：
1. 提供小米实况照片文件路径（例如：`D:\Photos\xiaomi\IMG_xxx.jpg`）

**收到路径后我将执行**：
1. 运行上述 ExifTool 分析命令
2. 生成小米格式分析报告
3. 更新002报告中的小米格式章节
4. 确定华为↔小米转换技术方案

---

*验证方案（格式分析类）*
*ExifTool 已就绪于 D:\Tools\exiftool*