# EMUI 动态照片 API 技术规格

> 文档类型：API规格
> 适用系统：EMUI / HarmonyOS 4（双框架）
> 更新日期：2026-05-18

---

## 1. 双框架系统概述

### 系统架构

**双框架**：Android + HarmonyOS 双系统架构
- 兼容 Android 应用
- 文件系统沿用 Android 模式（使用 `adb push` 命令）
- 动态照片采用**嵌入文件模式**（类似 Google/Samsung Motion Photo）

### 存储路径

```
双框架存储路径：/storage/emulated/0/DCIM/Camera/
```

---

## 2. Android MediaStore API

### 核心类

| 类 | 说明 |
|---|-----|
| `MediaStore.Images.Media` | 图片媒体库 |
| `ContentResolver` | 内容解析器 |
| `Cursor` | 查询结果游标 |

### 查询动态照片

```java
// Android MediaStore API（双框架）
ContentResolver resolver = context.getContentResolver();
Cursor cursor = resolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    new String[] {
        MediaStore.Images.Media._ID,
        MediaStore.Images.Media.DATA,
        MediaStore.Images.Media.DISPLAY_NAME,
        MediaStore.Images.Media.SIZE,
        MediaStore.Images.Media.DATE_TAKEN
    },
    null, null,
    MediaStore.Images.Media.DATE_TAKEN + " DESC"
);
```

### 检测嵌入视频

```java
// 方式1：读取文件末尾查找 LIVE_xxxx 标记
public boolean isHuaweiLivePhoto(String filePath) {
    try {
        RandomAccessFile file = new RandomAccessFile(filePath, "r");
        file.seek(file.length() - 20);
        byte[] tailBytes = new byte[20];
        file.read(tailBytes);
        String tailString = new String(tailBytes, "UTF-8");
        file.close();
        
        return tailString.startsWith("LIVE_");
    } catch (Exception e) {
        return false;
    }
}

// 方式2：查找 ftypmp4 标记（视频嵌入位置）
public long findEmbeddedVideoOffset(String filePath) {
    try {
        RandomAccessFile file = new RandomAccessFile(filePath, "r");
        byte[] buffer = new byte[1024];
        long offset = -1;
        
        // 从文件末尾向前搜索
        long fileSize = file.length();
        long searchStart = fileSize - 1024 * 1024; // 最后1MB
        
        file.seek(searchStart);
        while (file.getFilePointer() < fileSize) {
            int bytesRead = file.read(buffer);
            String chunk = new String(buffer, 0, bytesRead, "UTF-8");
            
            int ftypIndex = chunk.indexOf("ftypmp4");
            if (ftypIndex != -1) {
                offset = file.getFilePointer() - bytesRead + ftypIndex;
                break;
            }
        }
        
        file.close();
        return offset;
    } catch (Exception e) {
        return -1;
    }
}
```

---

## 3. 读取嵌入视频

### 解析末尾元数据

```java
public LivePhotoMetadata parseLivePhotoMetadata(String filePath) {
    try {
        RandomAccessFile file = new RandomAccessFile(filePath, "r");
        long fileSize = file.length();
        
        // 读取末尾60字节元数据
        file.seek(fileSize - 60);
        byte[] metadata = new byte[60];
        file.read(metadata);
        
        // 解析 Video info metadata（末尾20字节）
        String videoInfo = new String(metadata, 40, 20, "UTF-8");
        // 格式：LIVE_xxxx
        
        // 解析 Sight tremble metadata（倒数20~40字节）
        String sightTremble = new String(metadata, 20, 20, "UTF-8");
        // 格式：xx:xx
        
        // 解析 Version & Frame Num（倒数40~60字节）
        String versionFrame = new String(metadata, 0, 20, "UTF-8");
        // 格式：v3_f31_c
        
        file.close();
        
        // 提取视频长度
        int videoLength = Integer.parseInt(videoInfo.substring(5));
        
        return new LivePhotoMetadata(videoLength, sightTremble, versionFrame);
    } catch (Exception e) {
        return null;
    }
}
```

### 提取嵌入视频

```java
public byte[] extractEmbeddedVideo(String filePath, int videoLength) {
    try {
        RandomAccessFile file = new RandomAccessFile(filePath, "r");
        long fileSize = file.length();
        
        // 计算视频起始位置
        long videoStart = fileSize - videoLength - 40;
        
        // 读取视频数据
        byte[] videoData = new byte[videoLength - 40];
        file.seek(videoStart);
        file.read(videoData);
        
        file.close();
        return videoData;
    } catch (Exception e) {
        return null;
    }
}
```

---

## 4. 保存动态照片

### 双框架相机自动生成

双框架相机拍摄时自动生成嵌入格式的动态照片：
- 文件名：`IMG_XXXX.jpg`
- 格式：JPEG + 嵌入MP4 + 末尾元数据

### 第三方应用限制

第三方应用无法直接创建华为原生动态照片格式：
- 需要写入 MakerNotes（`HwMotionPhoto`）
- 需要嵌入视频数据并写入末尾元数据
- Android MediaStore API 不支持 MakerNotes 写入

### 替代方案

```java
// 方案A：使用 EXIF 写入自定义标记
ExifInterface exif = new ExifInterface(filePath);
exif.setAttribute("UserComment", "LiveHubMotionPhoto=1");
exif.saveAttributes();

// 方案B：使用 XMP 自定义命名空间
// 需要集成第三方 XMP 库（如 Adobe XMP SDK）
```

---

## 5. EXIF 元数据

### MakerNotes 结构（华为专用）

| 字段 | 说明 | 值 |
|-----|-----|---|
| `HwMotionPhoto` | 动态照片标识 | "yes" / "1" |
| `HwMotionPhotoIndex` | 视频文件索引 | 整数 |
| `HwMotionPhotoPresents` | 播放参数 | 字符串 |

### EXIF 读取示例

```java
ExifInterface exif = new ExifInterface(filePath);

// 检查是否为华为动态照片
String hwMotionPhoto = exif.getAttribute("HwMotionPhoto");
if (hwMotionPhoto != null && hwMotionPhoto.equals("yes")) {
    // 华为动态照片
}

// 读取其他 EXIF 信息
String orientation = exif.getAttribute(ExifInterface.TAG_ORIENTATION);
String dateTime = exif.getAttribute(ExifInterface.TAG_DATETIME);
```

---

## 6. 文件导入

### adb 命令

```bash
# 双框架导入（adb push）
adb push IMG_XXXX.jpg /storage/emulated/0/DCIM/Camera/

# 触发媒体库扫描
adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE \
    -d file:///storage/emulated/0/DCIM/Camera/IMG_XXXX.jpg
```

### 媒体库扫描识别

```
媒体库扫描识别逻辑：

1. 扫描新文件
2. 检查文件末尾20字节
3. 若为 "LIVE_xxxx"：
   ├── 识别为动态照片
   ├── 记录 subtype = 3（数据库字段）
   └── 图库显示动态照片标识
4. 若非 "LIVE_xxxx"：
   └── 识别为普通图片
```

---

## 7. 版本号差异

### 双框架动态照片版本规格

| 版本 | 规格说明 | 是否有TAG |
|-----|---------|----------|
| **v1** | 720P老版本动态照片 | ❌ 无版本号及封面帧号tag |
| **v2** | 4K版本 | ✅ 有版本号及封面帧号tag，版本号==2 |
| **v3** | 封面4K，视频1080P 75帧（最新版本） | ✅ 有版本号及封面帧号tag，版本号==3，_c结尾表示CinemaGraph |
| **v4** | 无TAG的4K版本（少量存在） | ❌ 无版本号及封面帧号tag |

---

## 8. 数据库字段

### Photos 表字段（双框架）

| 字段 | 含义 | 类型 | 动态照片规格 |
|-----|------|------|-------------|
| `_id` | 唯一标识 | INTEGER | 与图片一致 |
| `_data` | 资产存储路径 | TEXT | `/storage/emulated/0/DCIM/Camera/IMG_xx.jpg` |
| `_size` | 资产大小 | BIGINT | 文件总大小（含嵌入视频） |
| `_display_name` | 文件名含后缀 | TEXT | `IMG_XXXX.jpg` |
| `mime_type` | MIME类型 | TEXT | `image/jpeg` |
| `date_taken` | 拍摄时间 | INTEGER | 时间戳 |
| `orientation` | 图片方向 | INTEGER | EXIF方向值 |

**注意**：双框架数据库无 `subtype` 字段，需通过文件末尾元数据识别

---

## 9. 与单框架 API 对比

| 功能 | 双框架 API | 单框架 API |
|-----|-----------|-----------|
| 查询资产 | MediaStore | PhotoAccessHelper |
| 检测动态照片 | 文件末尾元数据 | PhotoSubtype.MOVING_PHOTO |
| 读取内容 | RandomAccessFile | MovingPhoto.requestContent() |
| 播放组件 | 无专用组件 | MovingPhotoView |
| 保存资产 | 相机自动生成 | MediaAssetChangeRequest |

---

## 参考资料

### Android官方文档
- MediaStore API: https://developer.android.com/reference/android/provider/MediaStore
- ExifInterface: https://developer.android.com/reference/android/media/ExifInterface
