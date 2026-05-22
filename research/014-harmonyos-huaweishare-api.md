# HarmonyOS 华为分享 API 技术文档总结

> 文档类型：API规格
> 适用系统：HarmonyOS NEXT（单框架）/ EMUI / HarmonyOS 4（双框架）
> 更新日期：2026-05-18

---

## 1. 华为分享跨端概述

### 功能说明

华为分享是华为设备的跨设备传输功能，支持：
- 近场传输（蓝牙 + Wi-Fi Direct）
- 远场传输（云端中转）
- 动态照片跨端兼容

### 跨端场景

| 场景 | 单框架设备 | 双框架设备 | 处理方式 |
|-----|-----------|-----------|---------|
| 单→双分享 | 发送方 | 接收方 | 媒体库融合为嵌入格式 |
| 双→单分享 | 接收方 | 发送方 | 媒体库拆分存储 |
| 单→单分享 | 发送方 | 接收方 | 保持分离格式 |

### 动态照片格式差异

| 系统类型 | 存储格式 | 分享格式 |
|---------|---------|---------|
| **单框架** (HarmonyOS NEXT) | 分离格式：JPG + MP4 + extraData | 融合为嵌入格式后分享 |
| **双框架** (EMUI/HarmonyOS 4) | 嵌入格式：单一JPG（含视频） | 直接分享单一JPG |

---

## 2. 单框架 API (HarmonyOS NEXT)

### 2.1 ShareKit API

**模块**：`@kit.ShareKit`
**核心类**：`systemShare.ShareController`, `systemShare.SharedData`

```typescript
import { systemShare } from '@kit.ShareKit';
import { uniformTypeDescriptor as utd } from '@kit.ArkData';

// 构建分享数据
let shareData = new systemShare.SharedData({
  utd: utd.UniformDataType.IMAGE,
  content: photoUri,      // 文件 URI
  title: '动态照片',
  description: '华为动态照片'
});

// 显示分享面板
let controller = new systemShare.ShareController(shareData);
let context = getContext(this) as common.UIAbilityContext;

await controller.show(context, {
  previewMode: systemShare.SharePreviewMode.DEFAULT,
  selectionMode: systemShare.SelectionMode.SINGLE
});
```

### 2.2 批量分享

```typescript
// 多文件分享
let shareData = new systemShare.SharedData({
  utd: utd.UniformDataType.IMAGE,
  content: photoUri1
});
shareData.addRecord({
  utd: utd.UniformDataType.IMAGE,
  content: photoUri2
});
```

### 2.3 媒体库融合方法

#### URI格式

```
公共目录媒体类URI格式：file://media/<mediaType>/IMG_DATATIME_ID/<displayName>

示例：file://media/Photo/1/IMG_1708594211_637/IMG_20240222_172911.jpg
```

### 融合方法（OS提供）

媒体库提供静态融合方法，将单框架分离格式转换为双框架嵌入格式：

```typescript
// 因为官方未公开这部分, 基于抓包观测, 仅作示意
function fuseMovingPhoto(
  imageUri: string,    // 图片URI
  videoUri: string,    // 视频URI
  extraDataPath?: string // 额外数据路径（双框架来源时存在）
): Promise<ArrayBuffer>
```

**融合流程**：

```
单框架手机 → 华为分享 → 双框架手机

1. 华为分享接收媒体库uri（file://media/Photo/xx/IMG_xx.jpg）

2. 媒体库提供静态融合方法：
   ├── 判断uri是否为媒体库uri（file://media/）
   ├── 判断动态照片来源（单框架/双框架）
   ├── 单框架来源：构造简单元数据（无CinemagraphInfo）
   ├── 双框架来源：从extraData读取原始元数据
   └── 融合为双框架格式的单一JPG文件

3. 发送融合后的文件到对端

4. 对端双框架手机：
   ├── 直接支持双框架格式
   └── 图库识别并播放动态照片

5. 对端单框架手机（若对端也为单框架）：
   ├── 媒体库扫描识别为动态照片
   └── 自动拆分为JPG + MP4 + extraData存储
```

#### 单框架来源融合（无extraData）

**构造简单元数据**

```python
def convert_movingphoto_to_livephoto_single_source(image_path, video_path):
    """
    单框架来源的动态照片 → 双框架格式
    
    特点：无CinemagraphInfo，Version & Frame Num无_c结尾
    """
    with open(image_path, 'rb') as img_f:
        image_data = img_f.read()
    
    with open(video_path, 'rb') as vid_f:
        video_data = vid_f.read()
    
    # 构造元数据
    sight_tremble = "0:0"  # 单框架来源无微动瞬间信息
    version_frame = "v3_f31"  # 无_c结尾
    video_length = len(video_data) + 40  # 视频长度 + 元数据长度
    video_info = f"LIVE_{video_length:04d}"
    
    # 组合文件
    combined_data = image_data + video_data
    combined_data += sight_tremble.encode()  # 20字节
    combined_data += version_frame.encode()  # 20字节
    combined_data += video_info.encode()  # 20字节
    
    return combined_data
```

#### 双框架来源融合（有extraData）

**从extraData读取元数据**

```python
def convert_movingphoto_to_livephoto_dual_source(image_path, video_path, extra_data_path):
    """
    双框架来源的动态照片（在单框架中） → 双框架格式
    
    特点：从extraData读取原始元数据信息
    """
    with open(image_path, 'rb') as img_f:
        image_data = img_f.read()
    
    with open(video_path, 'rb') as vid_f:
        video_data = vid_f.read()
    
    with open(extra_data_path, 'rb') as extra_f:
        extra_data = extra_f.read()
    
    # 从extraData解析元数据
    # extraData包含：CinemagraphInfo、Version & Frame Num、Sight tremble、Video info
    
    # 需要修改的信息：
    # 1. CinemagraphInfo末尾4字节长度需依据当前视频长度刷新
    # 2. Version & Frame Num封面帧需重新计算（依赖视频引擎：时间戳→帧号）
    # 3. Video info metadata中的长度需实时计算
    
    # 组合文件（类似双框架原始格式）
    combined_data = image_data + video_data
    combined_data += extra_data  # 包含所有元数据
    
    return combined_data
```

### 2.4 缓存文件位置

```
分享时生成的缓存文件位置：
/storage/cloud/100/files/.cache/Photo/xx/xxxxxx/livePhoto.xx

用途：华为分享发送融合后的双框架格式文件
```
---

## 3. 双框架 API (EMUI / HarmonyOS 4)

### 3.1 Android 分享 API

#### Intent 分享

```java
Intent shareIntent = new Intent(Intent.ACTION_SEND);
shareIntent.setType("image/jpeg");
shareIntent.putExtra(Intent.EXTRA_STREAM, Uri.parse(filePath));
Intent chooser = Intent.createChooser(shareIntent, "分享动态照片");
context.startActivity(chooser);
```

#### 多文件分享

```java
Intent shareIntent = new Intent(Intent.ACTION_SEND_MULTIPLE);
shareIntent.setType("image/jpeg");
ArrayList<Uri> uris = new ArrayList<>();
uris.add(Uri.parse(file1Path));
uris.add(Uri.parse(file2Path));
shareIntent.putExtra(Intent.EXTRA_STREAM, uris);
context.startActivity(Intent.createChooser(shareIntent, "分享多张照片"));
```

### 3.2 HiShare SDK（可选）

```java
import com.huawei.hishare.HiShare;
import com.huawei.hishare.HiShareConfig;

HiShareConfig config = new HiShareConfig.Builder()
    .setAppId("your_app_id")
    .build();
HiShare.getInstance().init(context, config);

HiShare.getInstance().shareFile(context, filePath, new HiShare.Callback() {
    @Override
    public void onSuccess(ShareResult result) {}
    @Override
    public void onFailure(int errorCode, String errorMsg) {}
});
```

### 3.3 分享权限

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="com.example.app.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

---

## 4. 跨端兼容性

### 4.1 跨品牌分享兼容性

| 目标设备 | 动态照片体验 | 说明 |
|---------|------------|-----|
| 华为设备（双框架） | 全功能 | 直接播放 |
| 华为设备（单框架） | 全功能 | 自动拆分存储 |
| 其他品牌设备 | 仅静态图 | 无法识别嵌入视频 |

### 4.2 格式兼容性矩阵

| 原始格式 | 华为双框架 | 华为单框架 | 其他品牌 |
|---------|-----------|-----------|---------|
| 华为嵌入格式 | 全功能 | 全功能 | 仅静态 |
| 华为分离格式 | 仅静态 | 全功能 | 仅静态 |
| Apple Live Photo | 仅静态 | 仅静态 | 仅静态（iOS除外） |
| Google Motion Photo | 仅静态 | 仅静态 | 有限 |

### 4.3 跨端处理流程总览

**单框架发送 → 双框架接收**：
- 单框架：媒体库融合分离格式为嵌入格式 → 发送单一JPG
- 双框架：接收单一JPG → 直接播放

**双框架发送 → 单框架接收**：
- 双框架：直接发送单一JPG（含嵌入视频）
- 单框架：接收 → 媒体库拆分为分离格式存储

**单框架发送 → 单框架接收**：
- 单框架发送：融合为嵌入格式发送
- 单框架接收：拆分为分离格式存储

---

## 7. 媒体库扫描识别

### 保存时自动识别

```python
def scan_and_parse_livephoto(file_path):
    """
    应用保存图片时，媒体库扫描解析是否为动态照片
    
    来源为双框架的动态照片，扫描时识别并拆分存储
    """
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)
        
        # 检查末尾20字节
        video_info = data[m-20:m]
        
        if video_info.startswith('LIVE_'):
            # 识别为双框架动态照片
            # 执行ConvertToMovingPhoto拆分逻辑
            result = convert_livephoto_to_movingphoto(file_path)
            
            # 存储拆分后的文件
            save_movingphoto_files(result)
            
            return True
        
        return False  # 普通图片
```

---

## 8. 华为分享跨端流程完整示例

### 发送方（单框架设备）

```typescript
// 1. 获取动态照片URI
let assetUri = 'file://media/Photo/1/IMG_1708594211_637/IMG_20240222_172911.jpg';

// 2. 使用系统分享面板
let shareData = new systemShare.SharedData({
  utd: utd.UniformDataType.IMAGE,
  content: assetUri,
  title: '动态照片'
});

let controller = new systemShare.ShareController(shareData);
await controller.show(context, {
  previewMode: systemShare.SharePreviewMode.DEFAULT,
  selectionMode: systemShare.SelectionMode.SINGLE
});

// 3. 华为分享内部处理：
// - 媒体库判断URI是否为动态照片（subtype == 3）
// - 若为动态照片，调用融合方法生成双框架格式文件
// - 发送融合后的单一JPG文件
```

### 接收方（双框架设备）

```
1. 华为分享接收单一JPG文件
2. 图库扫描识别末尾元数据（LIVE_xxxx）
3. 直接播放动态照片（无需拆分）
```

### 接收方（单框架设备）

```
1. 华为分享接收单一JPG文件
2. 媒体库扫描识别末尾元数据（LIVE_xxxx）
3. 执行ConvertToMovingPhoto拆分逻辑
4. 存储为分离格式：JPG + MP4 + extraData
```

---

## 参考资料

### 官方文档
- HarmonyOS ShareKit: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/share-utd-link
- Android Intent.ACTION_SEND: https://developer.android.com/reference/android/content/Intent#ACTION_SEND
- Android FileProvider: https://developer.android.com/reference/android/support/v4/content/FileProvider
- HMS Core HiShare: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/hishare-guidelines
