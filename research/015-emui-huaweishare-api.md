# EMUI 华为分享 API 技术规格

> 文档类型：API规格
> 适用系统：EMUI / HarmonyOS 4（双框架）
> 更新日期：2026-05-18

---

## 1. 华为分享概述

### 功能说明

华为分享（Huawei Share）是华为设备的跨设备传输功能，支持：
- 近场传输（蓝牙 + Wi-Fi Direct）
- 远场传输（云端中转）
- 动态照片跨端兼容

### 双框架特点

- 动态照片使用**嵌入文件模式**
- 分享时直接传输单一JPG文件（含嵌入视频）
- 接收方自动识别并播放

---

## 2. Android 分享 API

### Intent 分享

```java
// Android Intent 分享（双框架）
Intent shareIntent = new Intent(Intent.ACTION_SEND);
shareIntent.setType("image/jpeg");
shareIntent.putExtra(Intent.EXTRA_STREAM, Uri.parse(filePath));

// 创建选择器
Intent chooser = Intent.createChooser(shareIntent, "分享动态照片");

// 启动分享
context.startActivity(chooser);
```

### 多文件分享

```java
// 批量分享
Intent shareIntent = new Intent(Intent.ACTION_SEND_MULTIPLE);
shareIntent.setType("image/jpeg");

ArrayList<Uri> uris = new ArrayList<>();
uris.add(Uri.parse(file1Path));
uris.add(Uri.parse(file2Path));

shareIntent.putExtra(Intent.EXTRA_STREAM, uris);
context.startActivity(Intent.createChooser(shareIntent, "分享多张照片"));
```

---

## 3. 华为分享专用 API

### HiShare SDK（可选）

华为提供 HiShare SDK 用于增强分享功能：

```java
// 华为 HiShare SDK（需集成 HMS Core）
import com.huawei.hishare.HiShare;
import com.huawei.hishare.HiShareConfig;
import com.huawei.hishare.ShareResult;

// 初始化
HiShareConfig config = new HiShareConfig.Builder()
    .setAppId("your_app_id")
    .build();
HiShare.getInstance().init(context, config);

// 分享文件
HiShare.getInstance().shareFile(
    context,
    filePath,
    new HiShare.Callback() {
        @Override
        public void onSuccess(ShareResult result) {
            // 分享成功
        }
        
        @Override
        public void onFailure(int errorCode, String errorMsg) {
            // 分享失败
        }
    }
);
```

---

## 4. 动态照片分享流程

### 双框架 → 双框架

```
发送方（双框架）：
1. 选择动态照片（单一JPG文件）
2. 使用 Intent 或 HiShare SDK 分享
3. 传输单一JPG文件（含嵌入视频）

接收方（双框架）：
1. 接收单一JPG文件
2. 图库扫描识别末尾元数据（LIVE_xxxx）
3. 直接播放动态照片
```

### 双框架 → 单框架

```
发送方（双框架）：
1. 选择动态照片（单一JPG文件）
2. 使用 Intent 或 HiShare SDK 分享
3. 传输单一JPG文件（含嵌入视频）

接收方（单框架）：
1. 接收单一JPG文件
2. 媒体库扫描识别末尾元数据（LIVE_xxxx）
3. 执行 ConvertToMovingPhoto 拆分逻辑
4. 存储为分离格式：JPG + MP4 + extraData
```

---

## 5. 分享兼容性

### 跨品牌分享

| 目标设备 | 动态照片体验 | 说明 |
|---------|------------|-----|
| 华为设备（双框架） | ✅ 全功能 | 直接播放 |
| 华为设备（单框架） | ✅ 全功能 | 自动拆分存储 |
| 其他品牌设备 | ⚠️ 仅静态图 | 无法识别嵌入视频 |

### 格式兼容性矩阵

| 原始格式 | 华为双框架 | 华为单框架 | 其他品牌 |
|---------|-----------|-----------|---------|
| **华为嵌入格式** | ✅ 全功能 | ✅ 全功能 | ⚠️ 仅静态 |
| **华为分离格式** | ⚠️ 仅静态 | ✅ 全功能 | ⚠️ 仅静态 |
| **Apple Live Photo** | ⚠️ 仅静态 | ⚠️ 仅静态 | ⚠️ 仅静态（iOS除外） |
| **Google Motion Photo** | ⚠️ 仅静态 | ⚠️ 仅静态 | ⚠️ 有限 |

---

## 6. 分享文件处理

### 文件路径获取

```java
// 从 MediaStore 获取文件路径
Cursor cursor = resolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    new String[] { MediaStore.Images.Media.DATA },
    MediaStore.Images.Media._ID + " = ?",
    new String[] { String.valueOf(imageId) },
    null
);

if (cursor.moveToFirst()) {
    String filePath = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
    cursor.close();
}
```

### URI 转换

```java
// 文件路径转 URI
Uri fileUri = Uri.fromFile(new File(filePath));

// 或使用 FileProvider（Android 7.0+）
Uri contentUri = FileProvider.getUriForFile(
    context,
    "com.example.app.fileprovider",
    new File(filePath)
);
```

---

## 7. 分享权限

### 文件读取权限

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

### FileProvider 配置

```xml
<!-- AndroidManifest.xml -->
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

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <external-path name="external_files" path="." />
</paths>
```

---

## 8. 分享接收处理

### 接收 Intent

```java
// 接收分享的文件
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    
    Intent intent = getIntent();
    String action = intent.getAction();
    
    if (Intent.ACTION_SEND.equals(action)) {
        Uri uri = intent.getParcelableExtra(Intent.EXTRA_STREAM);
        if (uri != null) {
            // 处理接收的文件
            handleReceivedFile(uri);
        }
    }
}

private void handleReceivedFile(Uri uri) {
    // 检查是否为华为动态照片
    String filePath = getFilePathFromUri(uri);
    if (isHuaweiLivePhoto(filePath)) {
        // 动态照片处理逻辑
    } else {
        // 普通图片处理逻辑
    }
}
```

---

## 9. 分享优化建议

### 文件大小优化

```java
// 分享前检查文件大小
long fileSize = new File(filePath).length();
if (fileSize > 10 * 1024 * 1024) {  // 10MB
    // 提示用户文件较大
    // 或压缩后分享
}
```

### 分享进度提示

```java
// 使用 Notification 显示分享进度
NotificationManager notificationManager = 
    (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);

NotificationCompat.Builder builder = new NotificationCompat.Builder(context, "share_channel")
    .setContentTitle("正在分享")
    .setContentText("传输中...")
    .setSmallIcon(R.drawable.ic_share)
    .setProgress(100, 0, false);

notificationManager.notify(1, builder.build());
```

---

## 参考资料

### Android官方文档
- Intent.ACTION_SEND: https://developer.android.com/reference/android/content/Intent#ACTION_SEND
- FileProvider: https://developer.android.com/reference/android/support/v4/content/FileProvider

### 华为开发者文档
- HMS Core HiShare: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/hishare-guidelines
