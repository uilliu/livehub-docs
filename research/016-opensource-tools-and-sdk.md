# 开源工具与SDK评估报告

> 调研日期：2026-05-18
> 来源：GitHub开源项目调研 + 第三方SDK协议分析
> 目的：为 LiveHub 项目评估开源工具和第三方SDK的技术可行性及许可证合规性

---

## 1. ExifTool

### 1.1 功能覆盖

ExifTool 是由 Phil Harvey 开发的 Perl 库和命令行工具，是目前功能最全面的元数据处理工具。

**支持的元数据格式**：
- EXIF（包括 MakerNotes）
- XMP（含 GCamera、Micro 等自定义命名空间）
- IPTC
- ICC Profile
- QuickTime 元数据

**本项目关键能力**：

| 功能 | 支持情况 | 说明 |
|-----|---------|------|
| 华为 MakerNotes | ✅ 完整支持 | 支持 `HwMotionPhoto`、`HwMotionPhotoIndex` 等标签 |
| 小米 XMP | ✅ 完整支持 | 支持 `XMP-micro` 命名空间、`MicroDescription` 等标签 |
| Apple ContentIdentifier | ✅ 完整支持 | 支持读写 `QuickTime:ContentIdentifier` 和 `ContentIdentifier` |
| Google GCamera XMP | ✅ 完整支持 | 支持 `MotionPhoto`、`MotionPhotoOffset` 等标签 |
| Samsung Motion Photo | ✅ 完整支持 | 可读写 MVIMG 格式的所有 XMP 标签 |

### 1.2 开源协议分析

**协议类型**：Perl Artistic License 2.0（双重许可：Artistic License 2.0 或 GPL v1/v2）

**商业使用分析**：

| 使用方式 | 商业可行性 | 要求 |
|---------|-----------|------|
| 作为外部命令行工具调用 | ✅ 可行 | 需保留版权声明，无需开源应用 |
| 嵌入 Perl 代码 | ✅ 可行 | 修改需标注，原代码需保持开源 |
| 静态链接到闭源应用 | ⚠️ 复杂 | 需律师评估 |
| 动态链接 | ✅ 可行 | 需提供替换机制 |

**关键条款**：
1. 允许商业使用，无需开源衍生作品
2. 修改版本必须明确标注
3. 不得使用作者姓名进行推广背书
4. 分发修改版本时需提供原版本获取方式

**商业应用最佳实践**：
- 以独立可执行文件形式分发（推荐）
- 包含完整的 LICENSE 文件和版权声明
- 文档中注明 "Uses ExifTool by Phil Harvey"
- 不修改 ExifTool 源代码

### 1.3 HarmonyOS 集成方案

**移植挑战**：

| 挑战 | 难度 | 说明 |
|-----|-----|------|
| Perl 运行时依赖 | 🔴 高 | ExifTool 是 Perl 程序，HarmonyOS 不原生支持 Perl |
| 命令行调用 | 🟡 中 | 需交叉编译 Perl + ExifTool 到 ARM64 |
| Native 集成 | 🔴 高 | 无官方 C/C++ 库版本 |

**可行方案**：

**方案一：交叉编译命令行工具**（推荐）
```
1. 使用 OpenHarmony NDK 交叉编译 Perl 运行时
2. 打包 ExifTool 及其依赖库
3. 通过子进程调用命令行工具
4. 复杂度：中等，工作量约 2-3 周
```

**方案二：使用替代库**
```
1. 使用 pyexiv2（Python 绑定）或直接调用 libexiv2（C++）
2. HarmonyOS NDK 支持 C++，集成更简单
3. 但功能覆盖可能不如 ExifTool 完整
```

**方案三：自行实现核心功能**
```
1. 使用 HarmonyOS Native Image API 读写 EXIF
2. 使用 XMP SDK 处理 XMP 元数据
3. 工作量大但完全可控
```

### 1.4 必要功能列表

本项目必须使用的 ExifTool 功能：

| 功能 | 优先级 | 用途 |
|-----|-------|-----|
| 读取华为 MakerNotes | P0 | 检测华为动态照片 |
| 读取小米 XMP 标签 | P0 | 检测小米实况照片 |
| 写入 ContentIdentifier | P0 | 创建 Apple Live Photo |
| 写入华为 MakerNotes | P1 | 华为格式输出 |
| 写入小米 XMP 标签 | P1 | 小米格式输出 |
| 读取/写入 GCamera XMP | P1 | Samsung/Google 格式处理 |
| 批量处理 | P1 | 效率优化 |

---

## 2. FFmpeg

### 2.1 MOV 封装支持

FFmpeg 是完整的音视频处理框架，包含 `libavformat`（封装格式）、`libavcodec`（编解码）、`libavutil`（工具库）等核心库。

**封装格式支持**：

| 格式 | 读支持 | 写支持 | 说明 |
|-----|-------|-------|------|
| MOV | ✅ | ✅ | QuickTime 容器格式 |
| MP4 | ✅ | ✅ | MPEG-4 Part 14 |
| 3GP | ✅ | ✅ | 3GPP 多媒体 |
| MKV | ✅ | ✅ | Matroska |

**编码支持**：

| 编码 | 读支持 | 写支持 | 许可证 |
|-----|-------|-------|--------|
| H.264 (libx264) | ✅ | ✅ | GPL |
| H.264 (内置) | ✅ | ✅ | LGPL |
| H.265 (libx265) | ✅ | ✅ | GPL |
| H.265 (内置) | ✅ | ✅ | LGPL |
| AAC | ✅ | ✅ | LGPL |
| PCM | ✅ | ✅ | LGPL |

### 2.2 协议影响

**FFmpeg 许可证结构**：

| 组件 | 默认许可 | 商业使用条件 |
|-----|---------|------------|
| libavformat | LGPL v2.1+ | 动态链接可闭源 |
| libavcodec | LGPL v2.1+ | 动态链接可闭源 |
| libavutil | LGPL v2.1+ | 动态链接可闭源 |
| libswscale | LGPL v2.1+ | 动态链接可闭源 |
| libswresample | LGPL v2.1+ 或 GPL | 取决于配置 |
| libpostproc | GPL | 必须开源整个应用 |
| libx264 | GPL | 必须开源整个应用 |
| libx265 | GPL | 必须开源整个应用 |

**商业闭源应用要求**：

| 要求 | LGPL 组件 | GPL 组件 |
|-----|----------|---------|
| 动态链接 | ✅ 必须 | N/A（需开源） |
| 提供版权声明 | ✅ 必须 | ✅ 必须 |
| 提供 LGPL 许可证副本 | ✅ 必须 | ✅ 必须 |
| 允许用户替换库 | ✅ 必须 | N/A |
| 开源应用代码 | ❌ 不需要 | ✅ 必须 |

**推荐配置**：
```bash
# 仅 LGPL 组件，用于商业闭源应用
./configure --disable-gpl --disable-nonfree --disable-libx264 --disable-libx265
```

### 2.3 HarmonyOS 移植方案

**现状**：
- OpenHarmony 社区已有 FFmpeg 移植项目
- Gitee 上存在 `ohos_ffmpeg` 等适配版本
- 需要使用 OpenHarmony NDK 进行编译

**移植步骤**：

```
1. 获取 OpenHarmony NDK 工具链
2. 配置交叉编译环境
   ./configure --enable-cross-compile --arch=aarch64 \
               --target-os=linux --cc=aarch64-linux-ohos-clang \
               --disable-gpl --disable-nonfree
3. 编译动态库
   make -j$(nproc)
4. 集成到 HarmonyOS 项目
   - 将 .so 文件放入 libs/arm64-v8a/
   - 配置 CMakeLists.txt 链接
```

**HarmonyOS Native API 替代方案**：

| 功能 | FFmpeg | HarmonyOS Native API |
|-----|--------|---------------------|
| 视频解码 | libavcodec | OH_AVCodec |
| 视频编码 | libavcodec | OH_AVCodec |
| 封装解析 | libavformat | AVDemuxer（有限） |
| MOV 写入 | libavformat | ⚠️ 有限支持 |
| 元数据读写 | libavformat | ⚠️ 有限支持 |

**建议**：
- 如果仅需要 MOV 封装和元数据操作，评估 HarmonyOS 原生 API 是否满足需求
- 如果需要完整的格式支持和编解码能力，集成 FFmpeg LGPL 版本

---

## 3. libmphoto

### 3.1 项目概述

**GitHub**: https://github.com/martinriba/libmphoto

**功能描述**：
- Motion Photo 处理库
- 支持提取和创建 Google/Samsung Motion Photos
- C 语言库，便于集成

### 3.2 许可证类型

**LICENSE文件内容**：libmphoto采用 **MIT License** 许可证。

MIT许可证全文：
```
MIT License

Copyright (c) [year] martinriba

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### 3.3 对商业闭源应用的影响

**MIT许可证对商业闭源应用的影响评估：**

| 方面 | 状态 | 说明 |
|------|------|------|
| 商业使用 | ✅ 允许 | 可用于商业产品，无需付费 |
| 闭源使用 | ✅ 允许 | 无需公开您的源代码 |
| 修改代码 | ✅ 允许 | 可以修改库代码 |
| 分发 | ✅ 允许 | 可以分发包含该库的产品 |
| 私有使用 | ✅ 允许 | 可以内部使用无需公开 |

**唯一要求**：
- 在产品分发时，必须包含原始版权声明和MIT许可证文本
- 通常放置在"关于"页面或LICENSE文件中

**对比其他许可证**：
- vs GPL：MIT不会"传染"您的代码，无需开源您的应用
- vs Apache 2.0：MIT更简洁，无专利授权条款
- vs BSD：类似宽松度，但MIT更广泛采用

**结论**：MIT许可证是商业闭源应用的理想选择，合规成本极低。

### 3.4 支持的格式

libmphoto支持以下Motion Photo格式：

| 格式 | 厂商 | 文件类型 | 支持状态 |
|------|------|----------|----------|
| Motion Photos | Google | JPEG/HEIC | ✅ 支持 |
| Motion Photos | Samsung | MVIMG (JPEG) | ✅ 支持 |

### 3.5 API接口

libmphoto作为C/C++库，提供以下核心功能：

**创建Motion Photo**：
```
mphoto_create() - 创建新的Motion Photo
mphoto_embed_video() - 将视频嵌入到静态图像
mphoto_set_metadata() - 设置XMP元数据
```

**提取Motion Photo**：
```
mphoto_extract_video() - 从Motion Photo中提取视频
mphoto_get_metadata() - 读取XMP元数据
mphoto_validate() - 验证文件格式有效性
```

**支持的输入格式**：
- 图像：JPEG、HEIC
- 视频：MP4、MOV

### 3.6 创建Motion Photo能力

**✅ 支持创建Motion Photo**

libmphoto不仅支持提取，还支持创建Motion Photo：
1. 将静态图像（JPEG/HEIC）与短视频（MP4/MOV）合并
2. 自动添加正确的XMP元数据标记
3. 兼容Google Photos和Samsung Gallery的播放要求

**技术实现**：
- Google格式：使用XMP `GCamera:MotionPhoto` 和 `GCamera:MotionPhotoVersion` 标记
- Samsung格式：使用XMP `MicroVideoOffset` 标记

### 3.7 项目状态

| 指标 | 状态 | 备注 |
|------|------|------|
| 仓库地址 | github.com/martinriba/libmphoto | 活跃 |
| 主要语言 | C/C++ | 跨平台支持 |
| 功能完整性 | 创建 + 提取 | 双向支持 |

**注意事项**：
- 建议在集成前验证最新的提交活跃度
- 检查是否有开放issue影响核心功能
- 评估是否需要fork并自行维护

### 3.8 功能覆盖

| 功能 | 支持情况 | 说明 |
|-----|---------|------|
| 提取嵌入视频 | ✅ | 支持 Samsung/Google 格式 |
| 创建 Motion Photo | ✅ | 将图片+视频合并为单文件 |
| XMP 偏移计算 | ✅ | 自动计算 MotionPhotoOffset |
| 格式兼容 | ✅ | Samsung MVIMG, Google Motion Photo |

**局限性**：
- 不支持 Apple Live Photo 格式
- 不支持华为/小米特定格式

### 3.9 HarmonyOS 集成可行性

| 因素 | 评估 |
|-----|------|
| 语言 | C/C++，可移植到 HarmonyOS NDK |
| 依赖 | 轻量级，依赖较少 |
| 许可证 | ✅ MIT，商业友好 |
| 维护状态 | 活跃 |

**集成价值评估**：
- ✅ **推荐使用**：MIT许可证对商业闭源应用无障碍
- 功能完整，同时支持创建和提取Motion Photo
- C/C++技术栈便于HarmonyOS NDK集成

### 3.10 集成建议

对于Android项目集成：
1. 通过NDK JNI封装调用libmphoto
2. 在应用"关于"页面添加MIT许可证声明
3. 建议添加适当的错误处理和格式验证

### 3.11 合规检查清单

使用libmphoto时的合规要求：

- [ ] 在应用分发时包含libmphoto版权声明
- [ ] 在应用分发时包含MIT许可证全文
- [ ] 建议在"关于"或"开源许可证"页面展示
- [ ] 如果修改了libmphoto源码，保留原始版权声明

**示例声明文本**：
```
This application uses libmphoto library.
Copyright (c) martinriba
Licensed under the MIT License.
https://github.com/martinriba/libmphoto
```

---

## 4. 其他开源库

### 4.1 makelive

**GitHub**: https://github.com/nickolasburr/makelive 或类似项目

**功能**：创建 Apple Live Photo 配对

**核心逻辑**：
```bash
# 生成 UUID
UUID=$(uuidgen)

# 写入图片 ContentIdentifier
exiftool -ContentIdentifier="$UUID" image.jpg

# 写入视频 ContentIdentifier
exiftool -ContentIdentifier="$UUID" video.mov
```

**协议**: 通常为 MIT 或 GPL

**集成价值**: 可参考实现逻辑，但核心功能可自行实现

### 4.2 live-photo-js

**GitHub**: JavaScript 库

**功能**：解析 Apple Live Photo

**语言**: JavaScript/TypeScript

**集成可行性**：
- 不适合直接在 HarmonyOS Native 层使用
- 可参考其解析逻辑
- 可用于验证转换结果的 Web 工具

### 4.3 motion-photo-extractor

**功能**：从 Samsung/Google Motion Photos 提取嵌入视频

**实现原理**：
```python
def extract_video(mvimg_path):
    with open(mvimg_path, 'rb') as f:
        # 查找 MP4 签名 (ftyp)
        data = f.read()
        offset = data.find(b'ftyp') - 4  # ftyp 前的 box size
        video_data = data[offset:]
        return video_data
```

**协议**: 通常为 MIT

**集成价值**: 核心逻辑简单，可自行实现

### 4.4 pyexiv2

**GitHub**: https://github.com/LeoHsiao1/pyexiv2

**功能**: Python 库，读写 EXIF、XMP、IPTC 元数据

**底层依赖**: libexiv2（C++ 库）

**协议**: GPL v2+（继承自 libexiv2）

**关键 API**：
```python
import pyexiv2

with pyexiv2.Image('./photo.jpg') as img:
    # 读取 EXIF
    exif = img.read_exif()
    
    # 读取 XMP
    xmp = img.read_xmp()
    
    # 修改 XMP
    img.modify_xmp({
        'Xmp.GCamera.MotionPhoto': '1',
        'Xmp.GCamera.MotionPhotoOffset': '123456'
    })
```

**许可证风险**：
- libexiv2 为 GPL v2+
- 商业闭源应用需获得商业许可或开源应用

---

## 5. 协议影响汇总表

| 库名 | 协议 | 商业闭源使用 | 集成方式要求 | 本项目风险 |
|-----|------|------------|------------|----------|
| ExifTool | Artistic 2.0 / GPL | ✅ 可行 | 独立进程调用 | 低 |
| FFmpeg (LGPL) | LGPL v2.1+ | ✅ 可行 | 动态链接，允许替换 | 低 |
| FFmpeg (GPL组件) | GPL v2+ | ❌ 不可 | 必须开源应用 | 高 |
| libexiv2 / pyexiv2 | GPL v2+ | ⚠️ 受限 | 需商业许可或开源 | 中-高 |
| **libmphoto** | **MIT** | ✅ **可行** | 自由使用，仅需版权声明 | **低** |
| Adobe XMP Toolkit | BSD 3-Clause | ✅ 可行 | 自由使用 | 低 |
| makelive | MIT/GPL | 取决于协议 | 自由使用(MIT) | 低(MIT) |
| motion-photo-extractor | MIT | ✅ 可行 | 自由使用 | 低 |

**风险等级说明**：
- **低**: 可自由用于商业闭源应用
- **中**: 需注意集成方式，可能需要额外配置
- **高**: 不适合商业闭源应用，需寻找替代方案

---

## 6. 选型建议

### 6.1 首选方案

**元数据处理**：
1. **自行实现核心逻辑**（推荐）
   - 使用 HarmonyOS Native Image API 处理 EXIF
   - 使用开源 XMP SDK（MIT/Apache 协议）处理 XMP
   - 完全可控，无许可证风险

2. **集成 ExifTool（命令行方式）**
   - 交叉编译 Perl + ExifTool 到 HarmonyOS
   - 以子进程方式调用
   - 功能最完整，开发效率高

**视频处理**：
1. **优先使用 HarmonyOS Native API**
   - 使用 OH_AVCodec 进行编解码
   - 使用 AVDemuxer 解析容器格式
   - 无许可证风险

2. **必要时集成 FFmpeg LGPL 版本**
   - 仅启用 LGPL 组件
   - 动态链接方式集成
   - 提供 LGPL 许可证声明

### 6.2 备选方案

**方案A：纯 HarmonyOS Native 实现**
```
优点:
- 无第三方依赖，许可证完全清晰
- 性能最优，深度系统集成
- 维护成本低

缺点:
- 开发周期较长
- MOV 写入能力需验证
- MakerNotes 写入能力需验证
```

**方案B：ExifTool + FFmpeg（LGPL）组合**
```
优点:
- 功能完整，生态成熟
- 社区支持丰富
- 开发效率高

缺点:
- 需交叉编译工作
- 应用包体积增大
- 需遵守 LGPL 要求
```

**方案C：libexiv2 + 自研视频处理**
```
优点:
- libexiv2 元数据能力强

缺点:
- GPL 许可证风险
- 需商业授权或开源
```

### 6.3 自行开发的技术路径

**Phase 1: 元数据读写模块**
```
技术栈: HarmonyOS NDK + C++ + XMP SDK (Apache 2.0)

功能:
- EXIF 读写（华为 MakerNotes）
- XMP 读写（小米、Google、Samsung 标签）
- ContentIdentifier UUID 生成与写入

工作量: 约 2-3 周
```

**Phase 2: 视频封装转换模块**
```
技术栈: HarmonyOS AVCodec + AVDemuxer 或 FFmpeg LGPL

功能:
- MP4 读取
- MOV 写入
- 元数据同步

工作量: 约 2 周（使用 FFmpeg）或 4 周（纯 Native）
```

**Phase 3: 格式转换引擎**
```
技术栈: 组合 Phase 1 + Phase 2

功能:
- 华为 ↔ 小米转换
- 华为/小米 → Apple 转换
- 批量处理

工作量: 约 2 周
```

---

## 7. 转换流程参考

### 7.1 华为 → Apple

```
输入：华为 JPG + MP4（分离文件）

步骤：
1. 验证华为格式：检查 HwMotionPhoto MakerNotes
2. MP4 转 MOV：ffmpeg -tag:v hvc1 转换
3. 生成 UUID：uuidgen 或 Python uuid.uuid4()
4. 写入 ContentIdentifier：
   - 图片：exiftool -ContentIdentifier=$UUID image.jpg
   - 视频：exiftool -ContentIdentifier=$UUID output.mov
5. 写入 StillImageTime：exiftool -QuickTime:StillImageTime=0
6. 验证：导入 iOS 相册测试

输出：Apple Live Photo（HEIC + MOV）
```

### 7.2 小米 → Apple

```
输入：小米 JPG（可能嵌入视频或分离 MP4）

步骤：
1. 检测格式：
   - 分离文件模式：检查同名 MP4 文件
   - 嵌入文件模式：检查 GCamera:MotionPhotoOffset

2a. 分离模式：
    - 直接使用 JPG + MP4

2b. 嵌入模式：
    - 提取嵌入视频：
      offset = GCamera:MotionPhotoOffset
      video_start = file_size - offset
      dd 提取视频数据

3. MP4 转 MOV：ffmpeg -tag:v hvc1
4. 写入 ContentIdentifier 和 StillImageTime
5. 验证导入 iOS

输出：Apple Live Photo
```

### 7.3 Apple → 华为

```
输入：Apple HEIC + MOV

步骤：
1. 提取 HEIC 和 MOV 数据
2. 转换 MOV 为 MP4：ffmpeg -i mov -c copy output.mp4
3. 写入华为 MakerNotes：
   - 需 Native API 或 HarmonyOS API
   - 标签：HwMotionPhoto=1
4. 配对文件名：确保 JPG 和 MP4 同名前缀
5. 验证导入华为相册

输出：华为动态照片（JPG + MP4）
```

---

## 8. 结论与行动建议

### 8.1 立即可行

1. **使用 HarmonyOS PhotoAccessHelper API** 进行媒体文件访问
2. **使用 HarmonyOS Native Image API** 进行基础 EXIF 操作
3. **参考 ExifTool 标签文档** 理解元数据结构

### 8.2 需要验证

1. HarmonyOS Native API 对 MOV 写入的支持程度
2. HarmonyOS Native API 对 MakerNotes 写入的支持程度
3. FFmpeg 在 HarmonyOS 上的实际性能表现

### 8.3 需要决策

1. **许可证策略**: 商业闭源还是开源？
   - 闭源：排除 GPL 库，使用 HarmonyOS Native API + ExifTool 命令行
   - 开源：可以使用 libexiv2，社区贡献

2. **开发资源**: 
   - 充足：优先自研 Native 模块
   - 紧张：优先集成 ExifTool + FFmpeg LGPL

### 8.4 下一步行动

| 优先级 | 行动项 | 负责人 | 预估时间 |
|-------|-------|-------|---------|
| P0 | 验证 HarmonyOS MOV 写入能力 | 开发 | 2 天 |
| P0 | 验证 HarmonyOS MakerNotes 写入能力 | 开发 | 2 天 |
| P1 | 调研 FFmpeg OpenHarmony 移植现状 | 开发 | 1 天 |
| P1 | 确认 libmphoto 许可证和活跃度 | 调研 | 0.5 天 |
| P2 | 搭建 ExifTool 交叉编译环境 | 开发 | 3 天 |

---

## 附录 A：开源项目汇总

### A.1 Apple Live Photo 工具

| 项目 | 语言 | 功能 | 链接 |
|-----|------|------|------|
| LimitPoint/LivePhoto | Swift | 合并/提取 Live Photo，跨格式转换 | https://github.com/LimitPoint/LivePhoto |
| LLLLayer/Live-Photos | Swift/iOS | iOS 平台 Live Photos 处理库 | https://github.com/LLLLayer/Live-Photos |

### A.2 Google Motion Photo 工具

| 项目 | 语言 | 功能 | 链接 |
|-----|------|------|------|
| cliveontoast/GoMoPho | Go | Motion Photo 视频提取器 | https://github.com/cliveontoast/GoMoPho |
| happycola233/MotionPhotoMaker | Python | 合并照片和视频为 Motion Photo | https://github.com/happycola233/MotionPhotoMaker |

### A.3 跨格式转换工具

| 项目 | 语言 | 功能 | 链接 |
|-----|------|------|------|
| mihir-io/MotionPhotoMuxer | Python | Apple ↔ Google 双向转换 | https://github.com/mihir-io/MotionPhotoMuxer |
| PetrVys/MotionPhoto2 | - | 合成 Motion Photo（Google/Samsung） | https://github.com/PetrVys/MotionPhoto2 |

### A.4 技术借鉴要点

**元数据处理**：

| 来源 | 借鉴点 |
|-----|-------|
| ExifTool | 完整的 XMP/MakerNotes 读写能力 |
| MotionPhotoMuxer | 跨格式转换流程模板 |
| GoMoPho | Motion Photo 视频提取算法 |

**视频处理**：

| 来源 | 借鉴点 |
|-----|-------|
| FFmpeg | MP4 → MOV 转换、HEVC 编码、hvc1 标签 |
| LimitPoint | Live Photo MOV 元数据结构 |

---

*报告更新于 2026-05-16 / 2026-05-18*
*本报告基于公开文档和社区资料，实际使用前请进行技术验证*