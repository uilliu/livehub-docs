# 东湖(卓易通)LivePhoto识别机制调研报告

> 文档类型：技术方案
> 调研日期：2026-05-19
> 适用场景：湖内/湖外动态照片识别与处理

---

## 1. 东湖概述

### 1.1 定义

东湖（卓易通）是华为内部的容器化存储方案，用于隔离和管理特定应用数据。

### 1.2 特点

- 文件目录结构与文管相同
- 支持湖内/湖外双视角访问
- 媒体库需要特殊处理才能识别动态照片

---

## 2. 现状问题

### 2.1 问题描述

| 问题 | 说明 |
|-----|------|
| **湖内动态照片识别** | 东湖双框架的livePhoto，目前图库仅识别为静态照片 |
| **湖外视角解析** | 湖外看湖内时，媒体库将livePhoto当做.jpg解析，不支持动态照片 |
| **subtype未标记** | 数据库未正确标记subtype=3 |
| **动态效果丢失** | 用户无法播放动态照片的动态效果 |

### 2.2 问题原因

```
问题根因分析：

1. 扫描机制不完整
   - 媒体库扫描时未识别LIVE_TAG标识
   - 未正确解析文件末尾元数据

2. 数据库标记缺失
   - subtype字段未设置为3（MOVING_PHOTO）
   - 缺少动态照片相关字段

3. 文件路径映射问题
   - 湖内路径与湖外视角路径不一致
   - 媒体库无法正确关联文件
```

---

## 3. 解决方案

### 3.1 功能上线后方案

东湖文件目录结构与文管相同，功能上线后：

```
解决方案流程：

1. 扫描阶段：
   ├── 识别LIVE_TAG标识（文件末尾20字节）
   ├── 解析Version & Frame Num
   ├── 解析CinemagraphInfo（可选）
   └── 提取视频数据

2. 数据库标记：
   ├── subtype = 3（MOVING_PHOTO）
   ├── cover_position = 封面帧时间戳
   ├── moving_photo_effect_mode = 效果模式
   └── media_extra_data = JSON格式额外信息

3. 动态照片支持：
   ├── 图库显示动态照片标识
   ├── 支持动态照片播放
   └── 支持编辑和分享
```

### 3.2 识别逻辑优先级

```
1. 数据库识别（满足任一）：
   - subtype == 3
   - moving_photo_effect_mode = 10 && subtype = 0

2. 文件存在性检查：
   - 若存在mp4文件 → movingPhoto

3. 文件格式判断：
   - MovingPhotoFileUtils::IsLivePhoto(path)
   - 检查文件末尾LIVE_TAG标识
```

### 3.3 LivePhoto识别函数

```cpp
bool MovingPhotoFileUtils::IsLivePhoto(const string& path)
// 判断依据：
// 1. 检查文件末尾倒数20字节是否存在LIVE_TAG
// 2. 解析视频长度信息
// 3. 验证版本号和帧号标记
```

---

## 4. 存储路径映射

### 4.1 湖内路径

```
湖内路径：
/storage/cloud/files/Photo/<桶编号>/xxx.jpg
/storage/cloud/files/Photo/<桶编号>/xxx.mp4
```

### 4.2 湖外视角

```
湖外视角：
文管沙箱路径（具体路径取决于应用配置）

映射关系：
湖内路径 → 湖外视角路径
/storage/cloud/files → 文管沙箱路径
```

### 4.3 URI与路径映射

| 类型 | URI格式 | 绝对路径 | 用户视图 |
|-----|---------|---------|---------|
| 湖内媒体 | `file://media/Photo/IMG_datetime_0001/displayName.jpg` | `/storage/cloud/files/Photo/<桶编号>/IMG_datetime_0001.jpg` | 我的手机/图库/相册名/displayName.jpg |
| 湖外视角 | 文管沙箱URI | 文管沙箱路径 | 我的手机/文档/displayName.jpg |

### 4.4 file_source_type枚举

| 字段 | 值 | 含义 | 特征 |
|-----|---|------|------|
| MEDIA_HO_LAKE | 3 | 湖内媒体文件 | 保存在东湖容器内，图库和三方均可见 |

---

## 5. 实现方案

### 5.1 扫描识别实现

```python
def scan_donghu_livephoto(directory):
    """扫描东湖目录识别动态照片"""
    livephotos = []
    
    for file_path in os.listdir(directory):
        if file_path.endswith('.jpg'):
            # 检查是否为livePhoto
            full_path = os.path.join(directory, file_path)
            
            if is_livephoto(full_path):
                # 解析元数据
                metadata = parse_livephoto_metadata(full_path)
                
                # 构造动态照片记录
                livephoto = {
                    'file_path': full_path,
                    'subtype': 3,
                    'cover_position': metadata['cover_position'],
                    'effect_mode': metadata['effect_mode'],
                    'extra_data': metadata['extra_data']
                }
                
                livephotos.append(livephoto)
    
    return livephotos

def is_livephoto(file_path):
    """检查是否为livePhoto"""
    with open(file_path, 'rb') as f:
        f.seek(-20, 2)  # 定位到文件末尾倒数20字节
        tail_bytes = f.read(20)
        
        # 检查LIVE_TAG
        return tail_bytes.startswith('LIVE_')
```

### 5.2 数据库更新实现

```python
def update_database_for_livephoto(livephoto):
    """更新数据库标记动态照片"""
    # 构造数据库记录
    db_record = {
        'subtype': 3,  # MOVING_PHOTO
        'cover_position': livephoto['cover_position'],
        'moving_photo_effect_mode': livephoto['effect_mode'],
        'media_extra_data': json.dumps(livephoto['extra_data']),
        'file_source_type': 3,  # MEDIA_HO_LAKE
        'storage_path': livephoto['file_path']
    }
    
    # 插入或更新数据库
    insert_or_update_photos_table(db_record)
```

### 5.3 湖外视角适配

```python
def adapt_for_external_view(livephoto, external_path):
    """适配湖外视角"""
    # 映射路径
    mapped_path = map_path_to_external_view(livephoto['file_path'])
    
    # 更新数据库路径
    update_database_path(livephoto['file_id'], mapped_path)
    
    # 确保视频文件可见
    video_path = get_video_path(livephoto['file_path'])
    mapped_video_path = map_path_to_external_view(video_path)
    
    return {
        'image_path': mapped_path,
        'video_path': mapped_video_path
    }
```

---

## 6. 对LiveHub项目的影响

### 6.1 东湖动态照片导入

```python
def import_donghu_livephoto(file_path):
    """导入东湖动态照片到LiveHub"""
    # 1. 检查是否为livePhoto
    if not is_livephoto(file_path):
        return None
    
    # 2. 解析元数据
    metadata = parse_livephoto_metadata(file_path)
    
    # 3. 提取图片和视频数据
    image_data = extract_image_data(file_path)
    video_data = extract_video_data(file_path)
    
    # 4. 提取CinemagraphInfo（可选）
    cinemagraph_info = extract_cinemagraph_info(file_path)
    
    return {
        'image': image_data,
        'video': video_data,
        'cinemagraph_info': cinemagraph_info,
        'metadata': metadata,
        'source': 'donghu'
    }
```

### 6.2 关键技术依赖

| 功能 | 技术依赖 | 说明 |
|-----|---------|------|
| **LIVE_TAG识别** | 文件末尾读取 | 检查倒数20字节 |
| **元数据解析** | 自定义解析器 | 解析Version & Frame Num等 |
| **路径映射** | 文管API | 湖内路径→湖外视角 |
| **数据库更新** | 媒体库API | 更新subtype等字段 |

---

## 7. 测试验证方案

### 7.1 测试用例

| 测试项 | 测试方法 | 验证标准 |
|--------|---------|---------|
| **湖内识别** | 放置livePhoto文件到湖内目录 | 图库显示动态照片标识 |
| **湖外视角** | 湖外应用访问湖内动态照片 | 正确显示动态效果 |
| **subtype标记** | 查询数据库subtype字段 | subtype=3 |
| **播放功能** | 长按动态照片播放 | 正常播放动态效果 |
| **编辑功能** | 编辑动态照片 | 保留动态形态 |
| **分享功能** | 分享动态照片 | 接收方可播放 |

### 7.2 验证流程

```
验证流程：

1. 准备测试数据
   ├── 双框架livePhoto文件
   ├── 单框架movingPhoto文件
   └── 放置到湖内目录

2. 触发媒体库扫描
   ├── 手动触发扫描
   └── 等待自动扫描

3. 检查数据库记录
   ├── subtype字段是否为3
   ├── cover_position是否正确
   └── media_extra_data是否完整

4. 验证图库显示
   ├── 是否显示动态照片标识
   ├── 是否可播放动态效果
   └── 是否可编辑和分享

5. 验证湖外视角
   ├── 湖外应用是否可访问
   ├── 是否正确显示动态效果
   └── 是否可正常操作
```

---

## 8. 参考资料

### 相关文档
- `022-emui-movingphoto-format.md` - EMUI动态照片格式分析
- `023-harmonyos-movingphoto-format.md` - HarmonyOS动态照片格式分析
- `024-harmonyos-dual-single-framework-compatibility.md` - 鸿蒙单双框架差异与兼容方案

---

*报告创建于 2026-05-21*