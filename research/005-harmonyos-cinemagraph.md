# CinemagraphInfo数据结构调研报告

> 文档类型：技术规格
> 调研日期：2026-05-19
> 适用场景：微动瞬间效果数据处理

---

## 1. CinemagraphInfo概述

### 1.1 定义

CinemagraphInfo是华为动态照片中用于存储**微动瞬间（Sight Tremble）效果数据**的二进制结构。

### 1.2 特点

- **内容**：90帧的Cinemagraph数据（微动瞬间效果数据）
- **长度**：可变长度，从前4字节读取
- **典型大小**：60字节
- **存在条件**：仅当版本号以`_c`结尾时存在

---

## 2. 存储差异对比

### 2.1 双框架存储

| 对比项 | 说明 |
|--------|------|
| **存储位置** | 视频数据段后方，内嵌于图片文件 |
| **文件结构** | `[图片数据] + [视频数据] + [CinemagraphInfo] + [Version & Frame Num] + [元数据标签]` |
| **读取方式** | 从文件尾部解析Version & Frame Num是否含`_c` → 若有则向前读4字节获取长度 → 读取CinemagraphInfo |
| **数据格式** | 二进制数据，追加在视频段后 |
| **偏移计算** | CinemagraphInfo位于`[m-p-60, m-60)`区间 |

### 2.2 单框架存储

| 对比项 | 说明 |
|--------|------|
| **存储位置** | 视频文件独立track，或extraData文件 |
| **文件结构** | 图片和视频分离存储，CinemagraphInfo跟随视频文件 |
| **读取方式** | 从extraData文件读取，或从视频track获取 |
| **数据格式** | 跟随视频文件单独track |
| **存储路径** | `.editData/Photo/{bucket}/{filename}/extraData` |

---

## 3. 数据结构详解

### 3.1 文件布局

```
双框架动态照片文件布局（含CinemagraphInfo）：
┌────────────────────────────────────────────────────────────┐
│ [0, m-n-40)        │ JPEG图片数据（封面帧）                 │
│ [m-n-40, m-p-60)   │ MP4视频数据                            │
│ [m-p-60, m-60)     │ CinemagraphInfo数据                    │
│                    │ 前4字节：长度标记                       │
│                    │ 后续：90帧微动瞬间数据                  │
│ [m-60, m-40)       │ Version & Frame Num（v3_f31_c）        │
│ [m-40, m-20)       │ Sight tremble metadata（0:600）        │
│ [m-20, m)          │ Video info metadata（LIVE_xxxx）       │
└────────────────────────────────────────────────────────────┘

参数说明：
m = 文件总长度
n = 视频长度（从LIVE_xxxx解析）
p = CinemagraphInfo长度（从前4字节读取）
```

### 3.2 CinemagraphInfo数据结构

```python
class CinemagraphInfo:
    """CinemagraphInfo数据结构"""
    
    # 前4字节：长度标记
    length: int  # CinemagraphInfo数据长度
    
    # 后续数据：90帧微动瞬间数据
    frame_count: int = 90  # 固定90帧
    
    # 每帧数据结构（推测）
    frames: List[CinemagraphFrame]
    
class CinemagraphFrame:
    """单帧Cinemagraph数据"""
    
    frame_index: int      # 帧索引（0-89）
    motion_data: bytes    # 动态区域数据
    effect_params: dict   # 效果参数
```

---

## 4. 读取方法

### 4.1 双框架读取

```python
def read_cinemagraph_info_dual(file_path):
    """从双框架文件读取CinemagraphInfo"""
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)
        
        # 1. 检查Version & Frame Num是否含_c
        version_frame = data[m-60:m-40].decode('utf-8')
        if not version_frame.endswith('_c'):
            return None  # 无CinemagraphInfo
        
        # 2. 读取CinemagraphInfo长度（前4字节）
        cinema_length_bytes = data[m-64:m-60]
        cinema_length = int.from_bytes(cinema_length_bytes, 'big')
        
        # 3. 读取CinemagraphInfo数据
        cinema_data = data[m-cinema_length-64:m-64]
        
        return {
            'length': cinema_length,
            'data': cinema_data,
            'frame_count': 90
        }
```

### 4.2 单框架读取

```python
def read_cinemagraph_info_single(extra_data_path):
    """从单框架extraData文件读取CinemagraphInfo"""
    if not os.path.exists(extra_data_path):
        return None
    
    with open(extra_data_path, 'rb') as f:
        data = f.read()
        
        # 解析extraData结构
        # CinemagraphInfo位于extraData的特定位置
        # 需要根据extraData文件格式解析
        
        return parse_extra_data(data)
```

---

## 5. 元数据获取接口

### 5.1 媒体库新增接口

媒体库新增接口支持获取动态照片metadata：

```typescript
// 写入应用沙箱
requestContent(resourceType: ResourceType, fileUri: string): Promise<void>

// 返回ArrayBuffer
requestContent(resourceType: ResourceType): Promise<ArrayBuffer>
```

### 5.2 ResourceType枚举

```typescript
enum ResourceType {
  PHOTO_PROXY,                  // 相机封装的图片&视频句柄
  IMAGE_RESOURCE,               // 图片资源
  VIDEO_RESOURCE,               // 视频资源
  PRIVATE_MOVING_PHOTO_METADATA // 动态照片metadata（新增）
}
```

### 5.3 使用示例

```typescript
// 获取CinemagraphInfo数据
let metadata = await movingPhoto.requestContent(
  photoAccessHelper.ResourceType.PRIVATE_MOVING_PHOTO_METADATA
);

// 解析metadata数据
let cinemagraphInfo = parseCinemagraphInfo(metadata);
```

---

## 6. Sight Tremble Metadata关联

### 6.1 微动瞬间播放信息

Sight tremble metadata存储在文件末尾倒数20~40字节：

```
格式：xx:xx

解析：
- 分号前：微动瞬间开始时间（毫秒）
- 分号后：微动瞬间结束时间（毫秒）

示例：
- "0:0"：无微动瞬间信息（单框架来源）
- "0:600"：微动瞬间从0ms开始，到600ms结束
- "1500:3000"：微动瞬间从1.5s开始，到3s结束
```

### 6.2 与CinemagraphInfo的关系

| 元数据 | 关系 |
|--------|------|
| **CinemagraphInfo** | 存储90帧微动瞬间效果数据 |
| **Sight tremble metadata** | 存储微动瞬间播放时间范围 |
| **Version & Frame Num (_c)** | 标识是否支持CinemaGraph |

**播放逻辑**：
```
1. 检查Version & Frame Num是否以_c结尾
2. 若有_c，则存在CinemagraphInfo
3. 读取Sight tremble metadata获取播放时间范围
4. 根据CinemagraphInfo数据渲染微动瞬间效果
5. 播放时间范围：[start_time, end_time]
```

---

## 7. 对LiveHub项目的影响

### 7.1 CinemagraphInfo提取

```python
def extract_cinemagraph_info(file_path, framework_type):
    """提取CinemagraphInfo数据"""
    if framework_type == 'dual':
        return read_cinemagraph_info_dual(file_path)
    else:
        # 单框架：从extraData读取
        extra_data_path = get_extra_data_path(file_path)
        return read_cinemagraph_info_single(extra_data_path)
```

### 7.2 CinemagraphInfo转换

```python
def convert_cinemagraph_info(source_data, target_framework):
    """转换CinemagraphInfo数据格式"""
    if target_framework == 'dual':
        # 构造双框架格式
        cinema_length = len(source_data)
        cinema_length_bytes = cinema_length.to_bytes(4, 'big')
        return cinema_length_bytes + source_data
    else:
        # 构造单框架格式
        # 写入extraData文件
        return source_data
```

### 7.3 关键技术依赖

| 功能 | 技术依赖 | 说明 |
|-----|---------|------|
| **CinemagraphInfo解析** | 自定义解析器 | 读取二进制数据结构 |
| **帧数据提取** | 视频引擎API | 解析90帧数据 |
| **效果渲染** | 图像处理库 | 渲染微动瞬间效果 |
| **时间范围解析** | 字符串解析 | 解析Sight tremble metadata |

---

## 8. 参考资料

### 相关文档
- `022-emui-movingphoto-format.md` - EMUI动态照片格式分析
- `023-harmonyos-movingphoto-format.md` - HarmonyOS动态照片格式分析
- `024-harmonyos-dual-single-framework-compatibility.md` - 鸿蒙单双框架差异与兼容方案
