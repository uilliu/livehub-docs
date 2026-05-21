# HarmonyOS 动态照片 API 技术规格

> 文档类型：API规格
> 适用系统：HarmonyOS NEXT（单框架）
> 更新日期：2026-05-19

---

## 1. MovingPhoto 对象结构

### 文件组成

MovingPhoto（动态照片）由两部分组成：
- **静态图片**：JPG/HEIC/HEIF 格式（封面帧）
- **动态视频**：MP4/MOV/TS 格式（约3-10秒，H.264/H.265编码）

### 关联方式

- **分离文件模式**：图库中显示为单一"动态照片"条目，但文件系统为两个物理文件
- 通过文件名前缀相同 + 数据库 `subtype = 3` 关联

---

## 2. PhotoAccessHelper API

### 核心类与方法

| 类/方法 | 说明 |
|--------|-----|
| `photoAccessHelper.MovingPhoto` | 动态照片对象 |
| `MediaAssetManager.requestMovingPhoto()` | 从媒体库请求动态照片 |
| `MediaAssetManager.loadMovingPhoto()` | 从应用沙箱加载动态照片 |
| `MovingPhoto.requestContent()` | 请求内容（图片/视频数据） |
| `MovingPhoto.getUri()` | 获取动态照片 URI |

### requestMovingPhoto API

```typescript
static requestMovingPhoto(
  context: Context,
  asset: PhotoAsset,
  requestOptions: RequestOptions,
  dataHandler: MediaAssetDataHandler<MovingPhoto>
): Promise<string>
```

**参数**：
- `context`: Ability 实例上下文
- `asset`: 媒体文件对象（PhotoAsset）
- `requestOptions`: 请求配置（含 `deliveryMode`）
- `dataHandler`: 数据回调处理器

**返回值**：
- `Promise<string>`: 请求ID（可用于 `cancelRequest`）

**权限**：
- `ohos.permission.READ_IMAGEVIDEO`（Picker调用时无需）

**RequestOptions定义**：

```typescript
interface RequestOptions {
  deliveryMode: DeliveryMode;  // 交付模式
  compatibleMode: CompatibleMode; // 兼容模式（HDR/HEIF转码）
  mediaAssetProgressHandler: MediaAssetProgressHandler; // 进度回调（云端、转码）
  sourceMode: SourceMode; // 来源模式（原始/编辑）
}

enum DeliveryMode {
  FAST_MODE,           // 快速访问：基于现有图片生成返回
  HIGH_QUALITY_MODE,   // 高质量访问：返回全质量图，若没有则立即触发生成
  BALANCE_MODE         // 均衡访问：若存在全质量图则返回，否则先返回低质量图再触发生成
}

enum CompatibleMode {
  COMPATIBLE_MODE,     // 兼容模式（默认）
  ORIGINAL_MODE        // 原始模式
}

enum SourceMode {
  ORIGINAL_MODE,       // 原始模式
  EDITED_MODE          // 编辑模式（默认）
}
```

### loadMovingPhoto API

```typescript
static loadMovingPhoto(
  context: Context,
  imageFileUri: string,
  videoFileUri: string
): Promise<MovingPhoto>
```

**示例代码**：

```typescript
let imageFileUri = 'file://' + context.filesDir + '/local_moving_photo.jpg';
let videoFileUri = 'file://' + context.filesDir + '/local_moving_photo.mp4';
let movingPhoto = await photoAccessHelper.MediaAssetManager.loadMovingPhoto(
  context, imageFileUri, videoFileUri
);
```

---

## 3. MovingPhoto.requestContent() 方法

### 导出到沙箱

```typescript
// 同时导出图片和视频
let imageFileUri = context.filesDir + '/request_moving_photo.jpg';
let videoFileUri = context.filesDir + '/request_moving_photo.mp4';
await movingPhoto.requestContent(imageFileUri, videoFileUri);
```

### 获取 ArrayBuffer

```typescript
// 获取图片数据
let imageData = await movingPhoto.requestContent(
  photoAccessHelper.ResourceType.IMAGE_RESOURCE
);

// 获取视频数据
let videoData = await movingPhoto.requestContent(
  photoAccessHelper.ResourceType.VIDEO_RESOURCE
);
```

**ResourceType 枚举**：
- `PHOTO_PROXY` - 相机封装的图片&视频句柄
- `IMAGE_RESOURCE` - 图片资源
- `VIDEO_RESOURCE` - 视频资源

---

## 4. MovingPhotoView 播放组件

### 导入方式

```typescript
// API version 21及之前
import { photoAccessHelper, MovingPhotoView, MovingPhotoViewController, MovingPhotoViewAttribute } from '@kit.MediaLibraryKit';

// API version 22及之后
import { photoAccessHelper, MovingPhotoView, MovingPhotoViewController } from '@kit.MediaLibraryKit';
```

### 组件属性

| 属性 | 类型 | 默认值 | 说明 |
|-----|------|-------|-----|
| `movingPhoto` | `photoAccessHelper.MovingPhoto \| undefined` | - | 动态照片对象 |
| `controller` | `MovingPhotoViewController` | - | 播放控制器 |
| `muted` | `boolean` | `false` | 静音播放 |
| `objectFit` | `ImageFit` | `Cover` | 视频显示模式 |

### 事件回调

| 事件 | 说明 |
|-----|-----|
| `onStart()` | 播放开始触发 |
| `onFinish()` | 播放结束触发 |
| `onStop()` | 播放停止触发 |
| `onError()` | 错误触发 |

### 控制器方法

```typescript
controller: MovingPhotoViewController = new MovingPhotoViewController();

// 开始播放
controller.startPlayback();

// 停止播放
controller.stopPlayback();
```

### 完整示例

```typescript
@Entry
@Component
struct Index {
  @State src: photoAccessHelper.MovingPhoto | undefined = undefined
  @State isMuted: boolean = false
  controller: MovingPhotoViewController = new MovingPhotoViewController();

  build() {
    Column() {
      MovingPhotoView({
        movingPhoto: this.src,
        controller: this.controller
      })
        .muted(this.isMuted)
        .objectFit(ImageFit.Cover)
        .onStart(() => { console.info('onStart'); })
        .onFinish(() => { console.info('onFinish'); })
        .onStop(() => { console.info('onStop'); })
        .onError(() => { console.error('onError'); })

      Row() {
        Button('start').onClick(() => { this.controller.startPlayback(); })
        Button('stop').onClick(() => { this.controller.stopPlayback(); })
      }
    }
  }
}
```

---

## 5. 查询动态照片资产

### FetchOptions 配置

```typescript
import { dataSharePredicates } from '@kit.ArkData';

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
let asset = await assetResult.getFirstObject();
```

### PhotoViewPicker 选择

```typescript
let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.MOVING_PHOTO_IMAGE_TYPE;
photoSelectOptions.maxSelectNumber = 9;

let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
let photoSelectResult = await photoViewPicker.select(photoSelectOptions);
```

---

## 6. 保存动态照片

### MediaAssetChangeRequest 接口

```typescript
class MediaAssetChangeRequest {
  static createAssetRequest(
    context: Context,
    photoType: PhotoType,
    extension: string,
    options?: CreateOptions
  ): MediaAssetChangeRequest;

  addResource(type: ResourceType, fileUri: string): void;  // 通过fileUri添加资源
  addResource(type: ResourceType, data: ArrayBuffer): void; // 通过ArrayBuffer添加资源
  applyChanges(mediaChangeRequest: MediaChangeRequest): Promise<void>;
}

interface CreateOptions {
  title?: string;        // 图片或视频标题
  subtype?: PhotoSubtype; // 照片子类型（可选）
}
```

### 示例代码（基于安全控件）

```typescript
SaveButton().onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
  if (result == SaveButtonOnClickResult.SUCCESS) {
    const context = getContext(this);
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
    
    let changeRequest = photoAccessHelper.MediaAssetChangeRequest.createAssetRequest(
      context,
      photoAccessHelper.PhotoType.IMAGE,
      "jpg",
      {
        title: "moving_photo",
        subtype: photoAccessHelper.PhotoSubtype.MOVING_PHOTO
      }
    );
    
    changeRequest.addResource(
      photoAccessHelper.ResourceType.IMAGE_RESOURCE,
      "file://com.example/data/storage/el2/base/haps/entry/files/test.jpg"
    );
    
    changeRequest.addResource(
      photoAccessHelper.ResourceType.VIDEO_RESOURCE,
      "file://com.example/data/storage/el2/base/haps/entry/files/test.mp4"
    );
    
    await phAccessHelper.applyChanges(changeRequest);
  }
})
```

**注意**：不支持通过 `getWriteCacheHandler` 接口写入动态照片内容（报错 14000016）

---

## 7. 编辑动态照片

### 设置封面帧

```typescript
class MediaAssetChangeRequest {
  setCover(position: string, data?: ArrayBuffer): void;  // 动态照片封面帧
}

// 示例
let changeRequest = new MediaAssetChangeRequest(asset);
changeRequest.setCover(position, data);  // 需要传递原图封面帧
await phAccessHelper.applyChanges(changeRequest);
```

### 设置效果模式（system API）

```typescript
enum MovingPhotoEffectMode {
  DEFAULT,        // 默认
  LOOP,           // 循环播放
  BACK_AND_FORTH, // 来回播放
  LONG_EXPOSURE,  // 长曝光
  MULTI_EXPOSURE  // 多曝光
}

class MediaAssetChangeRequest {
  setEffectMode(mode: MovingPhotoEffectMode): void;  // 设置效果（system API）
}

// 示例
let changeRequest = new MediaAssetChangeRequest(asset);
changeRequest.setEffectMode(photoAccessHelper.MovingPhotoEffectMode.LONG_EXPOSURE);
changeRequest.addResource(ResourceType.VIDEO_RESOURCE, data);
await phAccessHelper.applyChanges(changeRequest);
```

### 编辑可回退

```typescript
class MediaAssetEditData {
  compatibleFormat: string;  // system表示标准编辑文件，支持应用自定义写
  formatVersion: string;     // 应用自定义版本号
  data: string;              // 编辑数据

  constructor(compatibleFormat: string, formatVersion: string);
}

class MediaAssetChangeRequest {
  setEditData(mediaAssetEditData: MediaAssetEditData): void;
  addResource(ResourceType.IMAGE_RESOURCE, data);  // 编辑后图
  addResource(ResourceType.VIDEO_RESOURCE, data);  // 编辑后视频
}

// 还原到初始设置
photoAsset.revertToOriginal();
```

---

## 8. MediaAssetDataHandler 回调

### Handler 实现

```typescript
class MovingPhotoHandler implements photoAccessHelper.MediaAssetDataHandler<photoAccessHelper.MovingPhoto> {
  async onDataPrepared(movingPhoto: photoAccessHelper.MovingPhoto) {
    if (movingPhoto === undefined) {
      console.error('Error occurred when preparing data');
      return;
    }
    console.info("moving photo acquired successfully, uri: " + movingPhoto.getUri());
    // 处理动态照片对象...
  }
}
```

---

## 9. PhotoSubtype 枚举

```typescript
enum PhotoSubtype {
  DEFAULT = 0,        // 默认照片类型（public API）
  SCREENSHOT = 1,     // 截屏录屏文件类型（system API）
  MOVING_PHOTO = 3,   // 动态照片文件类型（public API）
  GIF = 3             // GIF文件类型（注：与MOVING_PHOTO值相同，需区分）
}
```

---

## 10. PhotoKeys 枚举新增

```typescript
enum PhotoKeys {
  PHOTO_SUBTYPE = 'subtype';          // 照片子类型
  COVER_POSITION = 'cover_position';  // 封面帧位置
  MOVING_PHOTO_EFFECT_MODE = 'moving_photo_effect_mode'; // 效果模式
}
```

---

## 11. 错误码

| 错误码 | 说明 |
|-------|-----|
| 201 | 权限拒绝 |
| 401 | 参数错误（必填参数缺失、类型错误、验证失败） |
| 801 | 能力不支持 |
| 14000011 | 系统内部错误 |
| 14000016 | 不支持通过getWriteCacheHandler写入动态照片 |

---

## 12. 系统能力

**必需能力**：`SystemCapability.FileManagement.PhotoAccessHelper.Core`

---

## 13. SDK分层解析优化

### 优化背景

**优化前**：
- 启动时一次性解析所有Metadata
- 大量数据通过Parcel序列化传输
- 内存占用大、加载慢

**优化后**：
- 按需分层解析Metadata
- 使用共享内存优化Parcel传输
- 收益：应用依赖ROM减小1M，加载时延优化200ms

### Metadata分层结构

```
Metadata结构：
├── 设备基础信息（小数据量）
│   ├── Camera ID
│   ├── Sensor信息
│   └── 基础能力
├── 扩展能力配置（中等数据量）
│   ├── 支持模式
│   ├── 输出能力
│   └── Zoom能力
└── 动态运行数据（大数据量）
    ├── 实时帧数据
    ├── 人脸信息
    └── 3A参数
```

### 分层解析API

**CameraManager API**：

| API | 类型 | 说明 |
|-----|------|------|
| GetSupportedCameras | 修改 | 获取支持的相机设备（分层解析优化） |
| GetSupportedCapability | 修改 | 获取设备能力（分层解析优化） |
| GetSupportedModes | 修改 | 获取支持的模式（分层解析优化） |
| IsTorchSupported | 修改 | 是否支持闪光灯 |
| IsCameraMuteSupported | 修改 | 是否支持静音 |
| IsPrelaunchSupported | 修改 | 是否支持预启动 |

**CameraDevice API（新增）**：

| API | 类型 | 说明 |
|-----|------|------|
| CameraDevice | 新增 | 设备实例构造 |
| GetSupportedModes | 新增 | 获取设备支持的模式 |
| GetObjectTypes | 新增 | 获取对象类型 |
| ResetMetadata | 新增 | 重置Metadata |
| AddMetadata | 新增 | 添加Metadata |
| IsPrelaunch | 新增 | 是否预启动 |

---

## 14. API分层设计

### 普通应用层API（公开API）

**查询动态照片**：
```typescript
// 使用PhotoSubtype过滤
let predicates = new dataSharePredicates.DataSharePredicates();
predicates.equalTo('subtype', photoAccessHelper.PhotoSubtype.MOVING_PHOTO);
let fetchOptions = { fetchColumns: [], predicates: predicates };
let assetResult = await phAccessHelper.getAssets(fetchOptions);
```

**PhotoViewPicker选择**：
```typescript
let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.MOVING_PHOTO_IMAGE_TYPE;
photoSelectOptions.maxSelectNumber = 9;

let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
let photoSelectResult = await photoViewPicker.select(photoSelectOptions);
```

**读取动态照片**：
```typescript
// 请求MovingPhoto对象
let requestId = await MediaAssetManager.requestMovingPhoto(
  context, asset, requestOptions, handler
);

// 获取图片和视频数据
let imageData = await movingPhoto.requestContent(ResourceType.IMAGE_RESOURCE);
let videoData = await movingPhoto.requestContent(ResourceType.VIDEO_RESOURCE);
```

**保存动态照片**：
```typescript
// 创建动态照片资产
let changeRequest = MediaAssetChangeRequest.createAssetRequest(
  context, PhotoType.IMAGE, "jpg", { subtype: PhotoSubtype.MOVING_PHOTO }
);
changeRequest.addResource(ResourceType.IMAGE_RESOURCE, imageUri);
changeRequest.addResource(ResourceType.VIDEO_RESOURCE, videoUri);
await phAccessHelper.applyChanges(changeRequest);
```

### 高级开发者层API（系统API）

**分段式拍照**：
- 一阶段：80分图 + 全质量视频（临时存储24h）
- 二阶段：择机生成100分图 → 替换80分图
- 触发机制：高质量访问/后台空闲时触发

**编辑回退**：
- 存储三层文件：原图、编辑数据、效果图
- 支持滤镜/裁剪等全局编辑（保留动态形态）
- 涂鸦等局部编辑（降级为静态图）

**效果设置**：
```typescript
enum MovingPhotoEffectMode {
  DEFAULT,        // 默认
  LOOP,           // 循环播放
  BACK_AND_FORTH, // 来回播放
  LONG_EXPOSURE,  // 长曝光
  MULTI_EXPOSURE  // 多曝光
}

changeRequest.setEffectMode(MovingPhotoEffectMode.LONG_EXPOSURE);
changeRequest.setCover(position, data);
```

---

## 15. 第三方应用适配案例

### 拍摄美化垂域适配

| 应用名称 | 适配内容 | 适配效果 |
|---------|---------|---------|
| **抖音** | 动态照片拍摄、HDR Vivid、分段式拍照 | 完整动态照片功能支持 |
| **快手** | 动态照片拍摄、安全Picker | 系统相机能力集成 |
| **剪映** | 动态照片拍摄、HDR支持 | 编辑和拍摄能力 |
| **美图秀秀** | 动态照片拍摄、HDR Vivid | 拍照美化能力完整支持 |
| **快影** | 动态照片拍摄 | 视频编辑应用适配 |
| **醒图** | 动态照片拍摄、HDR | 图片编辑应用适配 |
| **今日水印相机** | 动态照片拍摄 | 水印相机应用适配 |
| **马克水印相机** | 安全Picker、动态照片 | 解决方案阶段性交流 |

### 关键适配能力

| 适配能力 | 适配要点 |
|---------|---------|
| **安全Picker** | 必选picker与可选picker配置 |
| **拍照一致性** | 统一拍照算法和流程 |
| **分段式拍照** | 优化拍照响应时延 |
| **HDR Vivid** | 支持HDR照片/视频拍摄 |
| **动态照片** | 提供三方动态照片拍摄 |
| **统一系统分享** | 全接和半接两种接入模式 |

---

## 16. 公开API文档链接

### HarmonyOS官方文档

| 文档类别 | 文档链接 | 重要程度 |
|---------|---------|---------|
| **动态照片拍摄** | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-movingphoto | ⭐⭐⭐⭐⭐ |
| **动态照片访问** | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/photoaccesshelper-movingphoto | ⭐⭐⭐⭐⭐ |
| **MovingPhotoView组件** | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/movingphotoview | ⭐⭐⭐⭐⭐ |
| **Camera Kit总览** | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-kit | ⭐⭐⭐⭐ |
| **Media Library Kit总览** | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/media-library-kit | ⭐⭐⭐⭐ |
| **PhotoAccessHelper API Reference** | https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetmanager | ⭐⭐⭐⭐ |

### EMUI动态照片

**重要说明**：EMUI版本未提供独立的公开API文档，动态照片功能主要通过以下方式提供：
- 系统相机应用内置动态照片模式
- Android MediaStore部分功能可用
- 建议迁移到HarmonyOS API

---

## 17. 系统能力

**必需能力**：`SystemCapability.FileManagement.PhotoAccessHelper.Core`

---

## 参考资料

### 华为官方文档
- PhotoAccessHelper MovingPhoto: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/photoaccesshelper-movingphoto
- MovingPhotoView Guidelines: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/movingphotoview-guidelines
- PhotoAccessHelper API Reference: https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetmanager
