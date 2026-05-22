# EMUI 动态照片格式分析

> **文档类型**：格式规格（简版）
> **适用系统**：EMUI / HarmonyOS 4（双框架）
> **更新日期**：2026-05-21

---

## 说明

**本文档为EMUI双框架格式快速参考，完整规格请查阅：**
- **主文档**：`006-harmonyos-dual-single-framework-compatibility.md`（第2章：双框架嵌入文件模式）
- **Cinemagraph详情**：`005-harmonyos-cinemagraph.md`

---

## 1. 格式概述

### 存储方式

**嵌入文件模式**：单一JPG文件，包含静态图片和嵌入视频

### 文件组成

```
双框架动态照片文件结构（单一JPG文件）：
┌────────────────────────────────────────────────────────────┐
│ [0, m-n-40)        │ JPEG图片数据（封面帧）                 │
│ [m-n-40, m-p-60)   │ MP4视频数据                            │
│ [m-p-60, m-p-40)   │ CinemagraphInfo（可选，4字节长度标记）  │
│ [m-p-40, m-40)     │ Version & Frame Num（20字节）          │
│ [m-40, m-20)       │ Sight tremble metadata（20字节）       │
│ [m-20, m)          │ Video info metadata（20字节）          │
└────────────────────────────────────────────────────────────┘

参数说明：
m = 文件总长度
n = Video info metadata中的视频长度（包括mp4+Cinemagraph+VersionAndFrame）
p = CinemagraphInfo长度（若存在）
```

### 存储路径

```
双框架存储路径：/storage/emulated/0/DCIM/Camera/
```

---

## 2. 末尾元数据详细规格

### 元数据位置

| 元数据名称 | 位置 | 格式 | 说明 |
|-----------|------|------|------|
| **Video info metadata** | 末尾倒数20字节 | `LIVE_xxxx` | xxxx为视频文件长度（包括mp4+Cinemagraph+VersionAndFrame） |
| **Sight tremble metadata** | 末尾倒数20~40字节 | `xx:xx` | 微动瞬间播放信息，分号前为开始时间(ms)，分号后为结束时间(ms) |
| **Version & Frame Num** | 末尾倒数40~60字节 | `v3_f31_c` | vn为版本号，f31为封面帧号，_c表示支持CinemaGraph |
| **CinemagraphInfo** | 末尾倒数60字节前4字节 | 4字节长度标记 | 仅当Version & Frame Num以_c结尾时存在 |

### 元数据结构详解

```
双框架动态照片文件结构（从后往前解析）：
┌────────────────────────────────────────────────────────────┐
│ [m-20, m)          │ Video info metadata（20字节）         │
│                    │ 格式：LIVE_xxxx                        │
│                    │ xxxx为视频长度（含Cinemagraph+Version）│
│                    │ 示例：LIVE_1234                        │
│ [m-40, m-20)       │ Sight tremble metadata（20字节）      │
│                    │ 格式：xx:xx                            │
│                    │ 分号前：微动瞬间开始时间(ms)            │
│                    │ 分号后：微动瞬间结束时间(ms)            │
│                    │ 示例：0:0（无微动瞬间）                 │
│                    │ 示例：1500:3000（1.5s-3s）             │
│ [m-60, m-40)       │ Version & Frame Num（20字节）         │
│                    │ 格式：v3_f31_c                         │
│                    │ v3：版本号                             │
│                    │ f31：封面帧号（第31帧）                 │
│                    │ _c：CinemaGraph标识（可选）            │
│ [m-p-60, m-60)     │ CinemagraphInfo（可选）               │
│                    │ 仅当版本号以_c结尾时存在               │
│                    │ 前4字节为长度标记                       │
│                    │ 内容：90帧微动瞬间数据                  │
└────────────────────────────────────────────────────────────┘

参数说明：
m = 文件总长度
n = 视频长度（从LIVE_xxxx解析）
p = CinemagraphInfo长度（从前4字节读取）
```

### 元数据解析示例

```python
def parse_livephoto_metadata(file_path):
    """解析双框架动态照片末尾元数据"""
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)
        
        # 1. Video info metadata（末尾20字节）
        video_info = data[m-20:m].decode('utf-8')
        video_length = int(video_info[5:])  # LIVE_xxxx
        
        # 2. Sight tremble metadata（倒数20~40字节）
        sight_tremble = data[m-40:m-20].decode('utf-8')
        start_time, end_time = sight_tremble.split(':')
        
        # 3. Version & Frame Num（倒数40~60字节）
        version_frame = data[m-60:m-40].decode('utf-8')
        has_cinema = version_frame.endswith('_c')
        
        # 4. CinemagraphInfo（可选）
        if has_cinema:
            cinema_length_bytes = data[m-64:m-60]
            cinema_length = int.from_bytes(cinema_length_bytes, 'big')
            cinema_data = data[m-cinema_length-64:m-64]
        
        return {
            'video_length': video_length,
            'sight_tremble': {'start': start_time, 'end': end_time},
            'version': version_frame,
            'has_cinema': has_cinema
        }
```

---

## 3. 版本号详细规格

### 版本类型

| 版本 | 规格说明 | 是否有TAG | 特点 |
|-----|---------|----------|------|
| **v1** | 720P老版本动态照片 | ❌ 无版本号及封面帧号tag | 视频分辨率720P |
| **v2** | 4K版本 | ✅ 有版本号及封面帧号tag | 封面和视频均为4K，版本号=2 |
| **v3** | 封面4K，视频1080P 75帧（最新版本） | ✅ 有版本号及封面帧号tag | _c结尾表示CinemaGraph |
| **v4** | 无TAG的4K版本（少量存在） | ❌ 无版本号及封面帧号tag | 需通过其他方式识别 |
| **v5** | 4.3 cepi动态照片 | ✅ 有版本号及封面帧号tag | 特殊版本 |
| **v6** | 单框架拍摄的动态照片 | ✅ 有版本号及封面帧号tag | 单框架默认版本 |
| **v7** | iOS克隆转换版本 | ✅ 有版本号及封面帧号tag | 能力不支持标记 |

### Version & Frame Num 格式解析

```
格式：v3_f31_c

解析：
- v3：版本号（v1/v2/v3/v4/v5/v6/v7）
- f31：封面帧号（帧号31）
- _c：CinemaGraph标识（可选）

用途：
- 版本号：确定视频分辨率和编码规格
- 封面帧号：封面帧对应的视频帧号
- CinemaGraph标识：是否支持动态图效果

版本号对应逻辑：
- v1：无TAG的720P版本
- v2：4K（有版本号和封面帧号tag，版本号==2）
- v3：封面4K，视频1080P 75帧（版本号==3，_c结尾支持CinemaGraph）
- v4：无TAG的4K版本
- v5：4.3 cepi动态照片
- v6：单框架拍摄的动态照片
- v7：iOS克隆转换版本（能力不支持标记）
```

### 版本号识别策略

```python
def detect_version(file_path):
    """检测双框架动态照片版本号"""
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)
        
        # 检查是否有TAG（末尾60字节）
        version_frame = data[m-60:m-40].decode('utf-8')
        
        if version_frame.startswith('v'):
            # 有TAG版本
            version_num = int(version_frame[1:].split('_')[0])
            has_cinema = version_frame.endswith('_c')
            return {
                'version': version_num,
                'has_tag': True,
                'has_cinema': has_cinema
            }
        else:
            # 无TAG版本（v1或v4）
            # 需通过视频分辨率判断
            video_info = data[m-20:m].decode('utf-8')
            video_length = int(video_info[5:])
            # 进一步分析视频分辨率...
            return {
                'version': 'unknown',
                'has_tag': False,
                'has_cinema': False
            }
```
格式：v3_f31_c

解析：
- v3：版本号（v1/v2/v3/v4）
- f31：封面帧号（帧号31）
- _c：CinemaGraph标识（可选）

用途：
- 版本号：确定视频分辨率和编码规格
- 封面帧号：封面帧对应的视频帧号
- CinemaGraph标识：是否支持动态图效果
```

---

## 4. 视频格式规格

### 支持的编码格式

| 资源类型 | 支持格式 | 说明 |
|---------|---------|------|
| 视频编码 | H.264 (AVC), H.265 (HEVC) | 视频编码格式 |
| 视频容器 | MP4 | 嵌入的视频容器格式 |

### 视频时长规格

| 版本 | 原规格 | 新规格 | 变更原因 |
|-----|--------|--------|---------|
| 初版 | 2~3秒 | (0, 3]秒 | 用户拍完照片就退出相机，无法获取拍照后1.5s视频流 |
| 当前 | (0, 3]秒 | **(0, 10]秒** | 三方应用（微博）及双框架存在超过3s的动态照片视频 |

---

## 5. 文件拆分计算公式

### 拆分逻辑

```python
def split_livephoto(file_path):
    """
    双框架动态照片 → 分离格式
    
    输入：单一JPG文件（含嵌入视频和末尾元数据）
    输出：JPG + MP4 + extraData三个文件
    """
    with open(file_path, 'rb') as f:
        data = f.read()
        m = len(data)  # 文件总长度
        
        # 解析末尾元数据
        video_info = data[m-20:m]  # "LIVE_xxxx"
        video_length = int(video_info[5:])  # xxxx
        
        # Sight tremble metadata
        sight_tremble = data[m-40:m-20]  # "xx:xx"
        
        # Version & Frame Num
        version_frame = data[m-60:m-40]  # "v3_f31_c"
        
        # 检查是否有CinemagraphInfo
        has_cinema = version_frame.endswith('_c')
        if has_cinema:
            cinema_length_bytes = data[m-64:m-60]
            cinema_length = int.from_bytes(cinema_length_bytes, 'big')
            p = cinema_length
        else:
            p = 0
        
        # 计算边界
        n = video_length
        image_end = m - n - 40
        video_start = m - n - 40
        video_end = m - p - 60
        
        # 提取内容
        image_data = data[0:image_end]
        video_data = data[video_start:video_end]
        
        # 提取额外信息（用于后续单→双转换）
        extra_data = {
            'video_info': video_info,
            'sight_tremble': sight_tremble,
            'version_frame': version_frame,
            'cinema_data': data[m-p-60:m-60] if has_cinema else None
        }
        
        return {
            'image': image_data,
            'video': video_data,
            'extra_data': extra_data,
            'cover_position': parse_cover_position(version_frame)
        }
```

### 边界计算公式

```
参数：
- m：文件总长度
- n：视频长度（从Video info metadata解析）
- p：CinemagraphInfo长度（若存在）

边界计算：
- 图片结束位置：image_end = m - n - 40
- 视频起始位置：video_start = m - n - 40
- 视频结束位置：video_end = m - p - 60

提取范围：
- 图片数据：[0, image_end)
- 视频数据：[video_start, video_end)
- 元数据：[video_end, m)
```

---

## 6. CinemagraphInfo 规格

### 存在条件

仅当 Version & Frame Num 以 `_c` 结尾时存在

### 存储位置

末尾倒数60字节前4字节为长度标记

### 内容说明

CinemagraphInfo 包含动态图效果的详细信息：
- 动态区域坐标
- 动态效果参数
- 其他特效数据

---

## 7. Sight Tremble Metadata 规格

### 格式

```
格式：xx:xx

解析：
- 分号前：开始时间（毫秒）
- 分号后：结束时间（毫秒）

用途：
- 微动瞬间播放信息
- 确定动态效果的播放时间范围
```

### 示例

```
"0:0"：无微动瞬间信息（单框架来源）
"1500:3000"：微动瞬间从1.5秒开始，到3秒结束
```

---

## 8. 封面帧时间戳转换

### 帧号转时间戳

```python
def parse_cover_position(version_frame):
    """
    解析封面帧时间戳
    
    输入：version_frame格式如 "v3_f31_c"
    输出：封面帧对应的视频时间戳（微秒us）
    """
    if not version_frame.startswith('v'):
        return 0  # 老版本无tag
    
    # 解析帧号
    parts = version_frame.split('_')
    frame_num = int(parts[1][1:])  # f31 -> 31
    
    # 需要调用视频引擎接口：pts与帧号互转
    # SR20240306642753: demuxer支持通过pts和帧号互转
    # SR20240307667648: MP4文件支持根据帧号查询时间戳
    
    # 暂时返回帧号（实际需要调用视频引擎）
    return frame_num
```

### 外部依赖

| 需求号 | 功能 | 交付版本 |
|-------|------|---------|
| SR20240306642753 | demuxer支持通过pts和帧号互转 | IT8\|OpenHarmony 5.0.0.38(C00) |
| SR20240307667648 | MP4文件支持根据帧号查询时间戳 | IT8\|OpenHarmony 5.0.0.39(C00) |

---

## 9. 数据库字段规格

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

## 10. 文件导入方式

### adb 命令

```bash
# 双框架导入（adb push）
adb push IMG_XXXX.jpg /storage/emulated/0/DCIM/Camera/

# 触发媒体库扫描
adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE \
    -d file:///storage/emulated/0/DCIM/Camera/IMG_XXXX.jpg
```

---

## 参考资料

### 相关文档
- `emui-movingphoto-api.md` - EMUI动态照片API规格
- `harmonyos-emui-movingphoto-difference.md` - HarmonyOS与EMUI差异对比
