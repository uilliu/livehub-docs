# Apple Live Photos 技术规格

> 调研日期：2026-05-18
> 来源：Apple 官方文档 + Web搜索整合
> 官方文档：PHLivePhoto, LivePhotosKitJS, Human Interface Guidelines

---

## 1. 文件结构

### 基本组成

Live Photos 由**两个独立文件**组成：

| 组件 | 格式 | 说明 |
|-----|-----|-----|
| **静态图片** | `.heic` (默认) 或 `.jpg` | HEIF/JPEG 格式，主帧 |
| **动态视频** | `.mov` | QuickTime 容器，约3秒 |

**分离文件模式**：
- 文件系统中为两个物理文件
- 通过 `ContentIdentifier` UUID 关联
- 相册显示为单一"Live Photo"条目

### 文件命名规则

```
示例：
IMG_0001.HEIC  # 静态图片
IMG_0001.MOV   # 关联视频（共享相同文件名前缀）
```

---

## 2. ContentIdentifier（核心元数据）

### UUID 格式要求

```
格式：UUID v4 字符串
示例：A5E2B1D8-4C9A-4F3E-A8E7-123456789ABC
```

**关键规则**：
- 图片和视频必须使用**完全相同的 UUID**
- UUID 在两个文件中分别存储于不同位置

### HEIC 文件存储

**位置**：EXIF MakerNotes Apple Dictionary

| Key | 说明 |
|-----|-----|
| `17` | ContentIdentifier UUID |

**ExifTool 读取命令**：
```bash
exiftool -ContentIdentifier image.heic
exiftool -QuickTime:ContentIdentifier image.heic
```

### MOV 文件存储

**位置**：QuickTime 元数据轨道

| 元数据键 | 说明 |
|---------|-----|
| `com.apple.quicktime.content.identifier` | ContentIdentifier UUID |
| `com.apple.quicktime.still-image-time` | 静态帧时间点（通常为 0） |

**写入命令**：
```bash
UUID="A5E2B1D8-4C9A-4F3E-A8E7-123456789ABC"

# 为 HEIC 写入 ContentIdentifier
exiftool "-ContentIdentifier=$UUID" image.heic

# 为 MOV 写入 ContentIdentifier
exiftool "-QuickTime:ContentIdentifier=$UUID" video.mov
```

---

## 3. StillImageTime 元数据

### 定义

`StillImageTime` 标记视频中的"静态帧"时间点：
- 用户按下快门时，相机同时记录一个静态帧
- 该帧作为"主要照片"展示在相册中
- 长按播放动态效果时，从此帧开始播放

**存储位置**：
- MOV 文件 QuickTime 元数据
- 元数据键：`com.apple.quicktime.still-image-time`
- 常见值：`0`（视频起始帧）或负数（表示主帧位置）

### ExifTool 操作

```bash
# 读取 StillImageTime
exiftool -QuickTime:StillImageTime video.mov

# 写入 StillImageTime
exiftool "-QuickTime:StillImageTime=0" video.mov
```

---

## 4. 视频编码要求

### MOV 容器要求

| 参数 | 要求 | 说明 |
|-----|-----|-----|
| **容器格式** | MOV (QuickTime) | 必须使用 MOV，不接受 MP4 |
| **视频编码** | H.264 (AVC) 或 H.265 (HEVC) | iOS 11+ 支持 HEVC |
| **音频编码** | AAC | 必须包含音轨（可为静音） |
| **视频时长** | ~3秒 | 标准 Live Photo 时长 |

### FFmpeg 转换命令

**HEVC + Apple 兼容性关键参数**：
```bash
# 转换为 Live Photo 兼容 MOV
ffmpeg -i input.mp4 -c:v libx265 -crf 28 \
       -c:a aac -b:a 128k \
       -tag:v hvc1 \  # ⚠️ 关键：Apple 要求 hvc1 标签
       -f mov output.mov

# 仅重封装（不重编码）
ffmpeg -i input.mp4 -c copy -tag:v hvc1 output.mov
```

**`-tag:v hvc1` 的作用**：
- Apple 软件（Photos、QuickTime）要求 HEVC 使用 `hvc1` 标签
- 默认的 `hev1` 标签会导致 Live Photo 无法识别
- 该参数将参数集存储在容器层而非比特流

---

## 5. iOS API 使用

### PHLivePhoto（iOS 本地框架）

**创建 Live Photo**：
```swift
import Photos

// 从文件创建 Live Photo
let imageURL = URL(fileURLWithPath: "image.HEIC")
let videoURL = URL(fileURLWithPath: "video.MOV")

PHLivePhoto.request(withResourceFileURLs: [imageURL, videoURL],
                    placeholderImage: nil,
                    targetSize: CGSize.zero,
                    contentMode: PHVideoRequestOptionsContentMode.aspectFit) { (livePhoto, info) in
    if let livePhoto = livePhoto {
        // 显示 Live Photo
        livePhotoView.livePhoto = livePhoto
    }
}
```

**保存到相册**：
```swift
PHPhotoLibrary.shared().performChanges({
    let creationRequest = PHAssetCreationRequest.forAsset()
    creationRequest.addResource(with: .photo, fileURL: imageURL, options: nil)
    creationRequest.addResource(with: .pairedVideo, fileURL: videoURL, options: nil)
}, completionHandler: { success, error in
    // 处理结果
})
```

### PHLivePhotoView（显示组件）

```swift
let livePhotoView = PHLivePhotoView(frame: frame)
livePhotoView.livePhoto = livePhoto

// 开始播放
livePhotoView.startPlayback(with: .full)

// 停止播放
livePhotoView.stopPlayback()
```

---

## 6. LivePhotosKitJS（Web 框架）

### CDN 引入

```html
<script src="https://cdn.apple.com/livephotoskit/1.5.3/livephotoskit.js"></script>
```

### Player 类

```javascript
// 创建 Live Photo 播放器
const player = new LivePhotosKit.Player();

// 设置图片和视频源
player.photoSrc = "image.HEIC";
player.videoSrc = "video.MOV";

// 播放控制
player.play();    // 开始播放
player.pause();   // 暂停
player.stop();    // 停止

// 事件监听
player.addEventListener('play', () => console.log('playing'));
player.addEventListener('ended', () => console.log('ended'));
```

---

## 7. Live Photo 创建流程

### 完整步骤

1. **准备素材**：
   - 静态图片：HEIC 或 JPEG
   - 动态视频：MOV（H.264/HEVC + AAC）

2. **生成 UUID**：
   ```bash
   UUID=$(uuidgen)  # macOS/Linux
   # 或在线工具生成
   ```

3. **写入 ContentIdentifier**：
   ```bash
   exiftool "-ContentIdentifier=$UUID" image.HEIC
   exiftool "-ContentIdentifier=$UUID" video.MOV
   ```

4. **写入 StillImageTime**：
   ```bash
   exiftool "-QuickTime:StillImageTime=0" video.MOV
   ```

5. **导入 iOS 设备验证**：
   - 将 HEIC + MOV 传输到 iOS
   - 在相册中长按检查是否为 Live Photo

---

## 8. 从其他格式转换为 Live Photo

### 华为/小米 → Apple

```bash
# 步骤1：提取 JPG 和 MP4
# （华为：已有 JPG + MP4 文件）

# 步骤2：MP4 转 MOV（FFmpeg）
ffmpeg -i video.mp4 -c:v libx265 -crf 28 \
       -c:a aac -b:a 128k -tag:v hvc1 \
       -f mov output.mov

# 步骤3：生成 UUID
UUID=$(uuidgen)

# 步骤4：写入 ContentIdentifier
exiftool "-ContentIdentifier=$UUID" image.jpg
exiftool "-ContentIdentifier=$UUID" output.mov

# 步骤5：写入 StillImageTime
exiftool "-QuickTime:StillImageTime=0" output.mov
```

### Google Motion Photo → Apple

```bash
# 步骤1：提取嵌入视频
exiftool -GCamera:MotionPhotoOffset motion.jpg
# 计算：video_start = file_size - MotionPhotoOffset

dd if=motion.jpg bs=1 skip=$video_start of=video.mp4

# 步骤2：MP4 转 MOV
ffmpeg -i video.mp4 -c:v libx265 -tag:v hvc1 output.mov

# 步骤3：写入 ContentIdentifier（同上）
UUID=$(uuidgen)
exiftool "-ContentIdentifier=$UUID" extracted.jpg
exiftool "-ContentIdentifier=$UUID" output.mov
```

---

## 9. 常见问题与解决方案

### PVT 文件问题

**问题**：iPhone AirDrop 到 Windows 后文件扩展名为 `.pvt`

**原因**：Windows 无法识别 HEIC + MOV 配对，错误命名

**解决**：
```bash
# 重命名 .pvt 为 .heic
mv IMG_0001.pvt IMG_0001.heic
```

### HEVC 不兼容问题

**症状**：MOV 文件无法在 iOS Photos 中识别为 Live Photo

**原因**：缺少 `hvc1` 标签

**解决**：
```bash
ffmpeg -i video.mov -c copy -tag:v hvc1 compatible.mov
```

### ContentIdentifier 不匹配

**症状**：图片和视频未配对显示

**检查**：
```bash
exiftool -ContentIdentifier image.heic
exiftool -QuickTime:ContentIdentifier video.mov

# 两值必须完全相同
```

---

## 10. 参考资料

### 官方文档

- [Apple Human Interface Guidelines - Live Photos](https://developer.apple.com/design/human-interface-guidelines/live-photos)
- [PHLivePhoto Framework Reference](https://developer.apple.com/documentation/Photos/PHLivePhoto)
- [LivePhotosKitJS Reference](https://developer.apple.com/documentation/LivePhotosKitJS)

### 开源工具

- [LimitPoint/LivePhoto](https://github.com/LimitPoint/LivePhoto) - Swift Live Photo 工具
- [mihir-io/MotionPhotoMuxer](https://github.com/mihir-io/MotionPhotoMuxer) - 格式转换工具
- [ExifTool](https://exiftool.org/) - 元数据操作

### 技术博客

- [Limit Point Blog - Working with Live Photos](https://www.limit-point.com/blog/2018/live-photos/)
- [Aaron.cc - FFmpeg HEVC for Apple Devices](https://aaron.cc/ffmpeg-hevc-apple-devices/)

---

*Apple Live Photos 技术规格*
*调研于 2026-05-18*