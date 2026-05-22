# HarmonyOS 动态照片格式分析

> **文档类型**：格式规格（简版）
> **适用系统**：HarmonyOS NEXT（单框架）
> **更新日期**：2026-05-21

---

## 说明

**本文档为HarmonyOS单框架格式快速参考，完整规格请查阅：**
- **主文档**：`006-harmonyos-dual-single-framework-compatibility.md`（第3章：单框架分离文件模式）
- **API详情**：`013-harmonyos-movingphoto-api.md`

---

## 1. 格式概述

### 存储方式

**分离文件模式**：图库中显示为单一"动态照片"条目，但文件系统为两个物理文件

### 文件组成

- **静态图片**：JPG/HEIC/HEIF 格式（封面帧）
- **动态视频**：MP4/MOV/TS 格式（约3-10秒，H.264/H.265编码）

### 关联方式

通过文件名前缀相同 + 数据库 `subtype = 3` 关联

### 存储路径

```
单框架存储路径：/storage/cloud/{userid}/files/Photo/{bucket}/

完整路径示例：
/storage/media/{userid}/local/files/Photo/{bucket}/A.jpg
/storage/media/{userid}/local/files/Photo/{bucket}/A.mp4
```

---

## 2. 存储目录结构

### 完整目录规格

```
单框架文件存储：/storage/cloud/{userid}/files/Photo/{bucket}/

|--Photo
|  |--{bucket}
|  |  |--A.jpg（效果图）
|  |  |--A.mp4（效果图视频）
|  |--{bucket}
|--.editData
|  |--Photo
|  |  |--{bucket}
|  |  |  |--A.jpg
|  |  |  |  |--origin_source.jpg（原始图）
|  |  |  |  |--origin_source.mp4（原始视频）
|  |  |  |  |--editdata（编辑数据）
|  |  |  |  |--editdata_camera（相机水印、滤镜）
|  |  |  |  |--extraData（双框架来源的额外数据）
|  |--{bucket}
|--.thumbs
|  |--Photo
|  |  |--{bucket}
|  |  | |--A.jpeg（缩略图）
|  |  | |--LCD.jpg（LCD缩略图）
|  |  | |--MTH.jpg（MTH缩略图）
|  |  | |--AbbreviatedVideo.mp4（缩略视频）
|  |--{bucket}
|--.cache
|  |--Photo
|  |  |--{bucket}
|  |  |  |--livePhoto.xx（分享时的缓存文件）
```

### 存储路径详解

**完整路径示例**：
```
效果图路径：/storage/media/{userid}/local/files/Photo/{bucket}/A.jpg
效果图视频：/storage/media/{userid}/local/files/Photo/{bucket}/A.mp4
原始图路径：/.editData/Photo/{bucket}/A.jpg/origin_source.jpg
原始视频：/.editData/Photo/{bucket}/A.jpg/origin_source.mp4
缩略图：/.thumbs/Photo/{bucket}/A.jpeg
缩略视频：/.thumbs/Photo/{bucket}/AbbreviatedVideo.mp4
缓存文件：/.cache/Photo/{bucket}/livePhoto.xx
```

**路径参数说明**：
- `{userid}`：用户ID（如100）
- `{bucket}`：相册桶编号（如0、1、4等）
- `A.jpg`：文件名（图片和视频同名不同后缀）

### URI与路径映射

| 类型 | URI格式 | 绝对路径 | 用户视图 |
|-----|---------|---------|---------|
| 文管 | `file://docs/storage/Users/currentUser/test.txt` | `/data/service/el2/hmdfs/account/files/Docs/test.txt` | 我的手机/test.txt |
| 媒体 | `file://media/Photo/IMG_datetime_0001/displayName.jpg` | `/storage/cloud/files/Photo/<桶编号>/IMG_datetime_0001.jpg` | 我的手机/图库/相册名/displayName.jpg |

### file_source_type枚举

| 字段 | 值 | 含义 | 特征 |
|-----|---|------|------|
| MEDIA | 0 | 媒体库媒体文件 | 保存到媒体库沙箱，图库和三方均可见 |
| FILE_MANAGER | 1 | 文管本地媒体文件 | 保存到文管沙箱，图库和三方均可见 |
| PERIPHERAL | 2 | 外设媒体文件 | 外设，图库和三方均可见 |
| MEDIA_HO_LAKE | 3 | 湖内媒体文件 | 保存在东湖容器内，图库和三方均可见 |
单框架文件存储：/storage/cloud/{userid}/files

|--Photo
|  |--0
|  |  |--A.jpg（效果图）
|  |  |--A.mp4（效果图）
|  |--1
|--.editData
|  |--Photo
|  |  |--0
|  |  |  |--A.jpg
|  |  |  |  |--origin_source.jpg（原始图）
|  |  |  |  |--origin_source.mp4（原始视频）
|  |  |  |  |--editdata
|  |  |  |  |--editdata_camera（相机水印、滤镜）
|  |  |  |  |--extraData（双框架来源的额外数据）
|  |--1
|--.thumbs
|  |--Photo
|  |  |--0
|  |  | |--A.jpeg
|  |  |  |--LCD.jpg
|  |  |  |--MTH.jpg
|  |  |--1
|--.cache
|  |--Photo
|  |  |--xx
|  |  |  |--xxxxxx
|  |  |  |  |--livePhoto.xx（分享时的缓存文件）
```

---

## 3. 关键目录说明

### 目录用途

| 目录路径 | 说明 |
|---------|------|
| `/Photo/{bucket}/A.jpg` | 效果图（编辑后的图片） |
| `/Photo/{bucket}/A.mp4` | 效果视频（编辑后的视频） |
| `/.editData/Photo/{bucket}/A.jpg/origin_source.jpg` | 原始图（未编辑的裸文件） |
| `/.editData/Photo/{bucket}/A.jpg/origin_source.mp4` | 原始视频（未编辑的裸文件） |
| `/.editData/Photo/{bucket}/A.jpg/editdata` | 编辑数据（JSON格式） |
| `/.editData/Photo/{bucket}/A.jpg/editdata_camera` | 相机水印、滤镜数据 |
| `/.editData/Photo/{bucket}/A.jpg/extraData` | **双框架来源的额外信息**（二进制文件，约51KB） |
| `/.thumbs/Photo/{bucket}/A.jpeg` | 缩略图（用于图库列表显示） |
| `/.thumbs/Photo/{bucket}/LCD.jpg` | LCD缩略图 |
| `/.thumbs/Photo/{bucket}/MTH.jpg` | MTH缩略图 |
| `/.thumbs/Photo/{bucket}/AbbreviatedVideo.mp4` | 缩略视频 |
| `/.cache/Photo/{bucket}/livePhoto.xx` | 分享时生成的双框架格式缓存文件 |

### 数据库表结构

**Photos表新增字段**：

| 字段 | 类型 | 说明 |
|-----|------|------|
| **file_source_type** | INT | 图片视频来源类型（MEDIA/FILE_MANAGER/PERIPHERAL/MEDIA_HO_LAKE） |
| **storage_path** | TEXT | 图片视频存储路径 |
| **cover_position** | BIGINT | 封面帧位置（时间戳us） |
| **moving_photo_effect_mode** | INT | 效果模式（循环/来回/长曝光/多曝光） |
| **original_subtype** | INT | 原始照片类型 |
| **media_extra_data** | TEXT | 双框架额外信息（JSON格式） |

**PhotoAlbum表变更**：

| 类型 | 子类型 | 值 | 说明 |
|-----|-------|---|------|
| Album_type | SOURCE | 2048 | 由应用创建的相册 |
| AlbumSubtype | SOURCE_GENERIC_FROM_FILEMANAGER | 2050 | 由文管创建的相册 |

---

## 4. extraData 文件内容

### 存储内容

```
extraData存储内容（用于单→双转换）：
├── CinemagraphInfo数据（90帧微动瞬间数据）
├── Version & Frame Num数据（版本号+封面帧号）
├── Sight tremble metadata数据（微动瞬间播放信息）
└── Video info metadata数据（视频长度信息）

存储位置：/.editData/Photo/xx/IMG_xx.jpg/extraData
用途：单框架来源的动态照片转回双框架格式时使用
文件大小：约51KB（二进制格式）
```

### 存在条件

仅当动态照片来源于双框架设备（数据克隆/分享接收）时存在

### CinemagraphInfo详解

**数据结构**：
- 内容：90帧的Cinemagraph数据（微动瞬间效果数据）
- 长度：可变长度，从前4字节读取
- 典型大小：60字节

**存储差异对比**：

| 对比项 | 双框架 | 单框架 |
|--------|--------|--------|
| **存储位置** | 视频数据段后方，内嵌于图片文件 | 视频文件独立track |
| **读取方式** | 从文件尾部解析Version & Frame Num是否含_c → 若有则向前读4字节获取长度 → 读取CinemagraphInfo | 从extraData文件读取，或从视频track获取 |
| **数据格式** | 二进制数据，追加在视频段后 | 跟随视频文件单独track |

---

## 5. 命名规则

### 文件命名规则

```
1. 图片和视频命名相同，仅后缀不同
   - 图片：IMG_XXXX.jpg
   - 视频：IMG_XXXX.mp4

2. 若图片重名加后缀，视频同样加后缀
   - 图片：IMG_XXXX(1).jpg
   - 视频：IMG_XXXX(1).mp4

3. 数据库存储图片路径，视频路径通过替换后缀获取
```

---

## 6. 视频格式规格

### 支持的格式

| 资源类型 | 支持格式 | 说明 |
|---------|---------|------|
| 图片 | `jpg`, `heic`, `heif` | 封面帧图片 |
| 视频 | `mp4`, `mov`, `ts`（不区分大小写） | 动态视频片段 |
| 编码 | H.264 (AVC), H.265 (HEVC) | 视频编码格式 |

### 视频时长规格变更

| 版本 | 原规格 | 新规格 | 变更原因 |
|-----|--------|--------|---------|
| 初版 | 2~3秒 | (0, 3]秒 | 用户拍完照片就退出相机，无法获取拍照后1.5s视频流 |
| 当前 | (0, 3]秒 | **(0, 10]秒** | 三方应用（微博）及双框架存在超过3s的动态照片视频 |

---

## 7. 数据库字段规格

### Photos 表字段（单框架）

| 字段 | 含义 | 类型 | 动态照片规格 |
|-----|------|------|-------------|
| **file_id** | 唯一标识 | INTEGER | 与图片一致 |
| **data** | 资产存储绝对路径 | TEXT | `xx/Photo/x/IMG_xx_xx.jpg` |
| **size** | 资产大小 | BIGINT | **image文件 + video文件 + extraData总大小**（与双框架保持一致） |
| **title** | 显示名 | TEXT | 与图片一致 |
| **display_name** | 文件名含后缀 | TEXT | 与图片一致 |
| **media_type** | 媒体类型 | INT | 值为1（IMAGE） |
| **mime_type** | MIME类型 | TEXT | `image/jpeg` |
| **subtype** | 照片子类型 | INT | **值为3（MOVING_PHOTO）** |
| **duration** | 间隔时长 | INT | 先定义为0 |
| **time_pending** | 临时文件 | BIGINT | image和video都写入完成后为0，其它状态非0 |
| **dirty** | 端云同步状态 | INT | **当前不上云，设置为-1** |
| **orientation** | 图片方向 | INT | 图片视频的EXIF都要改 |
| **latitude** | 纬度 | DOUBLE | 以图片EXIF中的纬度一致 |
| **longitude** | 经度 | DOUBLE | 以图片EXIF中的经度一致 |
| **height** | 高度 | INT | 与图片一致 |
| **width** | 宽度 | INT | 与图片一致 |
| **photo_id** | 分段式拍照主键ID | TEXT | 与图片一致 |
| **photo_quality** | 是否为高质量图 | INT | 与图片一致 |
| **deferred_proc_type** | 分段式任务类型 | INT | 与图片一致 |
| **+cover_position** | 封面帧位置 | BIGINT | **动态照片封面帧对应的视频时间戳(us)** |
| **+moving_photo_effect_mode** | 效果模式 | INT | **表示效果模式（循环、来回、长曝光、多曝光）** |
| **+original_subtype** | 原始照片类型 | INT | 表示原始照片类型 |
| **+media_extra_data** | 双框架额外信息 | TEXT | **JSON格式存储** |

---

## 8. size 字段计算规则

### 计算公式

```
单框架动态照片size计算（与双框架保持一致）：

size = image文件大小 + video文件大小 + extraData文件大小

目的：确保单框架与双框架手机显示的大小一致
场景：克隆判断文件是否重复时，size需在1M误差以内
```

---

## 9. cover_position 字段说明

### 字段含义

```
含义：动态照片封面帧对应的视频时间戳（单位：微秒us）

来源：
1. 双框架动态照片：解析Version & Frame Num中的帧号，通过pts转时间戳
2. 单框架动态照片：默认为0（无帧号信息）

用途：
- 图库显示封面帧
- 编辑时重选封面帧
- 视频播放定位
```

---

## 10. media_extra_data 字段内容

### JSON 格式

```json
{
  "CinemagraphInfo": "data",      // CinemagraphInfo数据（字符串或需改为blob）
  "VersionAndFrameNum": "v3_f31_c", // 版本号和封面帧号
  "SightTrembleMetadata": "0:0"    // 微动瞬间播放信息
}
```

### 版本号完整规格

| 版本 | 规格说明 | TAG状态 | 特点 |
|-----|---------|--------|------|
| **v1** | 720P老版本 | ❌ 无TAG | 视频分辨率720P |
| **v2** | 4K版本 | ✅ 有TAG | 封面和视频均为4K，版本号=2 |
| **v3** | 封面4K，视频1080P 75帧 | ✅ 有TAG，_c结尾 | 支持CinemaGraph |
| **v4** | 无TAG的4K版本 | ❌ 无TAG | 需通过其他方式识别 |
| **v5** | 4.3 cepi动态照片 | ✅ 有TAG | 特殊版本 |
| **v6** | 单框架拍摄的动态照片 | ✅ 有TAG | 单框架默认版本 |
| **v7** | iOS克隆转换版本 | ✅ 有TAG | 能力不支持标记 |

**版本号格式解析**：
```
格式：v3_f31_c

解析：
- v3：版本号
- f31：封面帧号（第31帧为封面）
- _c：CinemaGraph标识（支持微动瞬间效果）

用途：
- 版本号：确定视频分辨率和编码规格
- 封面帧号：封面帧对应的视频帧号
- CinemaGraph标识：是否支持动态图效果
```

---

## 11. attachments_data 字段

### 字段说明

| 字段 | 类型 | 说明 |
|-----|------|------|
| **attachments_data** | INT | 1表示为附件（查询资产时屏蔽），0为资产标志，后续可拓展到连拍照片 |

---

## 12. 编辑数据存储格式

### editdata 文件内容（JSON）

```json
{
  "time": "x",            // 媒体库自行添加
  "compatibleFormat": "xx",
  "formatVersion": "xxx",
  "data": "xxxx"
}
```

### 编辑数据目录结构

```
编辑数据存储位置：/.editData/Photo/xx/IMG_xx.jpg/

编辑数据内容：
├── origin_source.jpg（原始图）
├── origin_source.mp4（原始视频）
├── editdata（编辑数据）
├── editdata_camera（相机水印、滤镜）
└── extraData（双框架来源的额外数据）
```

---


### 动态照片内存计算
动态照片迁移需要关注迁移前后照片的大小相同
动态照片分为三部分组成 ：照片+视频+额外的存储信息（metadata）
在桶内输入 ls -la | grep +文件名称（不带尾缀），列出动态照片的图片文件+视频文件 的详情信息和文件大小 
metadata信息存储在 .editData目录中，相同方法搜索该动态照片的 metadata信息大小 再把三个文件的大小相加，得出动态照片的大小总和，在数据库内进行对比，大小应该相同。

## 13. 文件导入方式

### hdc + mediatool 命令

```bash
# 单框架导入（hdc + mediatool）
hdc file send IMG_XXXX.jpg /storage/media/100/local/files/Pictures/
hdc file send IMG_XXXX.mp4 /storage/media/100/local/files/Pictures/
hdc shell mediatool send /storage/media/100/local/files/Pictures/
```

---

## 14. 分段式拍照支持

### 分段式拍照流程

```
相机分段式拍摄动态照片流程：

1. 一阶段：相机保存低质量图和视频
   ├── 使用PhotoProxy保存低质量图
   ├── 使用应用沙盒内的视频uri写入视频文件
   └── 媒体库记录photo_id、deferred_proc_type

2. 二阶段：相机框架生成高质量图
   ├── HAL完成二段式照片处理
   ├── 通过延时子服务传递给媒体库
   └── 媒体库完成二阶段图片对一阶段图片的替换

3. 触发时机：
   ├── 图库访问新拍摄照片（高优先级触发）
   ├── 三方应用访问（高优先级触发）
   └── 系统资源允许时（适时触发）
```

---

## 参考资料

### 相关文档
- `harmonyos-movingphoto-api.md` - HarmonyOS动态照片API规格
- `harmonyos-emui-movingphoto-difference.md` - HarmonyOS与EMUI差异对比
