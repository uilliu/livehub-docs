# 华为与Apple动态照片兼容性方案调研报告

> 文档类型：兼容性方案
> 调研日期：2026-05-19
> 适用场景：iOS动态照片克隆、跨平台迁移

---

## 1. iOS动态照片克隆机制

### 1.1 克隆流程

```
iOS Live Photo克隆到华为单框架流程：

1. 识别阶段：
   - 识别名称带dynamic的数据
   - 搬迁对应MOV格式数据到同一目录

2. 转换阶段：
   - 修改MOV后缀为MP4
   - 不插入MOV数据到数据库
   - 插入metadata数据（版本号+封面帧数）

3. extraData写入：
   - 格式：v7_f<封面帧>
   - v7标记能力不支持
   - 封面帧位置：前600ms位置
   - LIVE_<视频大小+20>
```

### 1.2 克隆后数据结构

```
克隆后的单框架存储结构：
/storage/cloud/100/files/.editData/Photo/
├── xxx.jpg         # 封面图片（从HEIC转换）
├── xxx.mp4         # 动态视频（从MOV转换）
└── extraData       # 元数据（v7_f<帧号>）
```

### 1.3 元数据转换规则

| iOS元数据 | 华为元数据 | 转换规则 |
|----------|-----------|---------|
| **ContentIdentifier** | - | 用于配对识别 |
| **still_image_time** | cover_position | 封面帧时间戳转换 |
| **HEIC格式** | JPEG格式 | 图片格式转换 |
| **MOV格式** | MP4格式 | 视频格式转换 |
| **无版本号** | v7版本号 | iOS克隆版本标记 |

**extraData写入格式**：
```
v7_f<封面帧号>
- v7：iOS克隆版本标记（能力不支持）
- f<帧号>：封面帧号（前600ms位置）
- LIVE_<视频大小+20>：视频长度标记
```

---

## 2. iOS动态照片存储细节

### 2.1 iOS Live Photo结构

iOS动态照片由两部分组成：
- **HEIC图片**：封面帧（HEIF编码）
- **MOV视频**：动态内容（前后各1.5秒）

### 2.2 核心API对比

| iOS类 | 接口 | 功能描述 |
|-------|-----|---------|
| PHLivePhotoView | `startPlayback(with:)` | 播放控制（full/hint/循环/来回/长曝光） |
| | `stopPlayback()` | 停止播放 |
| | `livePhotoBadgeImage(options:)` | 获取LivePhoto角标图标 |
| PHLivePhoto | `request(withResourceFileURLs:...)` | 从HEIC+MOV合成LivePhoto |
| | `cancelRequest(withRequestID:)` | 取消请求 |
| PHAssetResource | `assetResources(for:)` | 获取LivePhoto源文件（HEIC+MOV） |

### 2.3 华为与iOS存储对比

| 特性 | 华为双框架livePhoto | iOS Live Photo | 华为单框架movingPhoto |
|-----|-------------------|----------------|----------------------|
| 文件格式 | 单文件（JPG+MP4合并） | 双文件（HEIC+MOV） | 双文件（JPG+MP4分离） |
| 元数据位置 | 文件末尾 | 独立文件 | 数据库+extraData |
| 版本管理 | v1-v7版本标记 | 无版本概念 | v6/v7版本号 |
| CinemaGraph支持 | 支持（_c标记） | 不支持 | 不支持 |
| 封面帧标记 | f<帧号> | 无 | cover_position字段 |
| 视频编码 | H.264 | H.265(HEVC) | H.264 |
| 图片编码 | JPEG | HEIC | JPEG |

---

## 3. 克隆判重逻辑

### 3.1 判重依据

| 场景 | 判重依据 |
|-----|---------|
| 四元组判重 | displayName + fileSize + orientation |
| 移动到相册 | 视频不插入数据库 |
| 编辑另存 | 对封面图片判重 |
| 重命名 | 视频跟随重命名 |

### 3.2 判重实现

```python
def check_duplicate_ios_livephoto(heic_path, mov_path):
    """iOS Live Photo克隆判重"""
    # 读取HEIC元数据
    heic_metadata = {
        'display_name': get_display_name(heic_path),
        'file_size': get_file_size(heic_path),
        'orientation': get_orientation(heic_path)
    }
    
    # 查询数据库是否存在相同四元组
    existing = query_database(heic_metadata)
    
    if existing:
        # 不插入MOV数据到数据库
        return {'duplicate': True, 'existing_id': existing['file_id']}
    else:
        # 插入新记录
        return {'duplicate': False}
```

---

## 4. 克隆已知问题

### 4.1 问题列表

| 问题 | 状态 | 原因 | 解决方案 |
|-----|------|------|---------|
| 旋转角度异常 | 待解决 | iOS元数据旋转角度处理差异 | 需特殊处理EXIF旋转标记 |
| 滤镜仅对封面生效 | 已确认 | 无法获取编辑后的视频 | iOS滤镜数据无法迁移 |
| 日期显示1970年 | 已确认 | 图库侧问题 | 需正确设置date_taken |
| 地理位置未更新 | 已确认 | 读取图片exif而非传递数据 | 需正确传递GPS数据 |
| HEIF格式上云变静态 | 已知问题 | 双框架版本较老 | HEIF需特殊处理 |

### 4.2 旋转角度异常定位

**问题原因**：
- iOS使用EXIF Orientation标记旋转角度
- 华为图库解析时可能未正确处理
- MOV视频的旋转元数据与HEIC不一致

**解决方案**：
```python
def fix_rotation_issue(heic_path, mov_path):
    """修复旋转角度异常"""
    # 读取HEIC EXIF旋转标记
    heic_orientation = get_exif_orientation(heic_path)
    
    # 读取MOV视频旋转元数据
    mov_rotation = get_video_rotation(mov_path)
    
    # 统一旋转角度
    if heic_orientation != mov_rotation:
        # 转换MOV视频旋转
        convert_video_rotation(mov_path, heic_orientation)
    
    # 写入华为数据库
    update_database_orientation(heic_orientation)
```

---

## 5. 跨平台兼容性对比

### 5.1 跨平台兼容性矩阵

| 特性 | 双框架livePhoto | 单框架movingPhoto | iOS Live Photo | 东湖livePhoto |
|-----|----------------|------------------|----------------|---------------|
| **文件格式** | JPG+MP4合并 | JPG+MP4分离 | HEIC+MOV分离 | JPG+MP4合并 |
| **元数据位置** | 文件末尾 | 数据库+extraData | 独立文件 | 文件末尾 |
| **版本管理** | v1-v4 | v7 | 无 | v1-v4 |
| **CinemaGraph** | 支持 | 不支持 | 不支持 | 支持 |
| **封面帧标记** | f<帧号> | extraData | 无 | f<帧号> |
| **视频编码** | H.264 | H.264 | H.265(HEVC) | H.264 |
| **图片编码** | JPEG | JPEG | HEIC | JPEG |
| **克隆支持** | - | 支持 | 支持（转MP4） | - |
| **分享兼容** | 需解封装 | 原生支持 | 需转换 | 需解封装 |
| **播放组件** | movingPhotoView | movingPhotoView | PHLivePhotoView | movingPhotoView |

### 5.2 转换方案对比

| 转换方向 | 转换方法 | 技术依赖 | 注意事项 |
|---------|---------|---------|---------|
| **iOS→华为单框架** | HEIC→JPEG + MOV→MP4 + extraData写入 | FFmpeg + ExifTool | v7版本标记 |
| **iOS→华为双框架** | HEIC→JPEG + MOV→MP4 + 末尾元数据 | FFmpeg + 文件写入 | 无CinemaGraph |
| **华为→iOS** | JPG→HEIC + MP4→MOV + ContentId写入 | FFmpeg + ExifTool | 需配对标识 |

---

## 6. iOS克隆实现方案

### 6.1 克隆代码示例

```python
def clone_ios_livephoto_to_huawei(heic_path, mov_path, output_dir):
    """iOS Live Photo克隆到华为单框架"""
    
    # 1. 转换图片格式
    jpeg_path = convert_heic_to_jpeg(heic_path, output_dir)
    
    # 2. 转换视频格式
    mp4_path = convert_mov_to_mp4(mov_path, output_dir)
    
    # 3. 计算封面帧位置
    still_image_time = get_still_image_time(mov_path)
    cover_frame = calculate_cover_frame(still_image_time)
    
    # 4. 构造extraData
    extra_data = {
        'version': 'v7',
        'cover_frame': cover_frame,
        'video_length': get_file_size(mp4_path) + 20,
        'sight_tremble': '0:0'
    }
    
    # 5. 写入extraData文件
    write_extra_data(jpeg_path, extra_data)
    
    # 6. 更新数据库
    insert_database_record({
        'subtype': 3,
        'cover_position': still_image_time,
        'moving_photo_effect_mode': 0,
        'media_extra_data': json.dumps(extra_data)
    })
    
    return {'image': jpeg_path, 'video': mp4_path}
```

### 6.2 HEIC转JPEG

```python
def convert_heic_to_jpeg(heic_path, output_dir):
    """HEIC格式转换为JPEG"""
    # 使用FFmpeg或ImageMagick
    import subprocess
    
    jpeg_path = os.path.join(output_dir, 
        os.path.splitext(os.path.basename(heic_path))[0] + '.jpg')
    
    # FFmpeg转换
    subprocess.run([
        'ffmpeg', '-i', heic_path,
        '-q:v', '2', jpeg_path
    ])
    
    # 保留EXIF元数据
    copy_exif_metadata(heic_path, jpeg_path)
    
    return jpeg_path
```

### 6.3 MOV转MP4

```python
def convert_mov_to_mp4(mov_path, output_dir):
    """MOV格式转换为MP4"""
    import subprocess
    
    mp4_path = os.path.join(output_dir,
        os.path.splitext(os.path.basename(mov_path))[0] + '.mp4')
    
    # FFmpeg转换（保持视频质量）
    subprocess.run([
        'ffmpeg', '-i', mov_path,
        '-c:v', 'libx264', '-c:a', 'aac',
        '-q:v', '0', '-q:a', '0',
        mp4_path
    ])
    
    return mp4_path
```

---

## 7. 对LiveHub项目的影响

### 7.1 iOS动态照片导入处理

```python
def import_ios_livephoto(livephoto_pair):
    """导入iOS Live Photo到LiveHub"""
    heic_path = livephoto_pair['image']
    mov_path = livephoto_pair['video']
    
    # 1. 解析iOS元数据
    content_id = get_content_identifier(heic_path)
    still_image_time = get_still_image_time(mov_path)
    
    # 2. 转换为华为格式
    result = clone_ios_livephoto_to_huawei(heic_path, mov_path, temp_dir)
    
    # 3. 提取图片和视频数据
    with open(result['image'], 'rb') as f:
        image_data = f.read()
    with open(result['video'], 'rb') as f:
        video_data = f.read()
    
    # 4. 转换为目标格式（如Apple Live Photo）
    # MOV格式保持不变（iOS原生支持）
    # 或转换为其他厂商格式
    
    return {
        'image': image_data,
        'video': video_data,
        'metadata': {
            'content_id': content_id,
            'cover_time': still_image_time,
            'source': 'ios'
        }
    }
```

### 7.2 关键技术依赖

| 功能 | 技术依赖 | 说明 |
|-----|---------|------|
| **HEIC解析** | ExifTool / libheif | HEIF格式解析库 |
| **MOV解析** | FFmpeg | QuickTime格式解析 |
| **格式转换** | FFmpeg | HEIC→JPEG, MOV→MP4 |
| **元数据提取** | ExifTool | Content Identifier提取 |
| **封面帧定位** | FFmpeg | still_image_time解析 |

---

## 8. 最佳实践建议

### 8.1 克隆流程建议

| 建议 | 具体措施 | 预期效果 |
|-----|---------|---------|
| **保留原始元数据** | 正确提取iOS EXIF和QuickTime元数据 | 确保信息完整 |
| **统一旋转角度** | 处理HEIC和MOV的旋转差异 | 避免旋转异常 |
| **正确设置日期** | 从iOS元数据提取拍摄时间 | 避免1970年问题 |
| **传递GPS数据** | 从HEIC EXIF提取GPS信息 | 确保地理位置正确 |
| **版本号标记** | 使用v7标记iOS来源 | 识别能力限制 |

### 8.2 错误处理建议

```python
def handle_clone_errors(heic_path, mov_path):
    """处理克隆过程中的错误"""
    errors = []
    
    # 检查文件是否存在
    if not os.path.exists(heic_path):
        errors.append('HEIC文件不存在')
    if not os.path.exists(mov_path):
        errors.append('MOV文件不存在')
    
    # 检查Content Identifier是否配对
    heic_content_id = get_content_identifier(heic_path)
    mov_content_id = get_content_identifier(mov_path)
    if heic_content_id != mov_content_id:
        errors.append('Content Identifier不匹配')
    
    # 检查视频时长
    duration = get_video_duration(mov_path)
    if duration > 10:
        errors.append('视频时长超过10秒限制')
    
    return errors
```

---

## 9. 参考资料

### Apple官方文档
- [Apple Live Photo Technical Format Specification](https://support.apple.com/en-us/HT207022)
- [Apple Photos Framework - Understanding Live Photos](https://developer.apple.com/documentation/photos/understanding_live_photos)
- [PHLivePhoto Class Reference](https://developer.apple.com/documentation/photos/phlivephoto)

### 相关文档
- `024-harmonyos-dual-single-framework-compatibility.md` - 鸿蒙单双框架差异与兼容方案
- `015-apple-livephoto-spec.md` - Apple Live Photo技术规格
