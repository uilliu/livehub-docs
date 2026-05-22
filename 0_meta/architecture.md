# LiveHub 项目架构规划

> 版本：1.0
> 更新日期：2026-05-18
> 目的：定义项目目录结构，包括文档架构和工程架构

---

## 一、项目根目录结构

```
livehub/
├── docs/                           # 文档库（归档后的最终产品文档）
│   ├── 0_meta/                     # 元级定义
│   ├── 1_product/                  # 产品与项目管理
│   ├── 2_design/                   # 设计产出
│   ├── 3_development/              # 开发规约与指引
│   └── superpowers/                # Superpowers 技能相关文档
│       └── specs/                  # 技术调研报告
│
├── livehub-harmonyos/              # 鸿蒙版应用
├── livehub-core/                   # 跨平台核心模块（Native C++）
├── livehub-android/                # Android 版应用（可选扩展）
├── livehub-ios/                    # iOS 版应用（可选扩展）
├── livehub-web/                    # Web 版工具（可选扩展）
│
├── .claude/                        # Claude Code 配置
├── CLAUDE.md                       # 项目开发指令
├── AI_CONTEXT.md                   # 仓库级索引入口
├── README.md                       # 项目说明
└── LICENSE                         # 许可证文件
```

---

## 二、文档架构（docs/）

### 2.1 目录结构详情

```
docs/
├── 0_meta/                         # 元级定义
│   ├── vision.md                   # 愿景&目标
│   ├── glossary.md                 # 统一语言术语表 ✅ 已创建
│   ├── architecture.md             # 总体架构图（C4/六边形）
│   └── doc_arch.md                 # 文档架构说明 ✅ 已存在
│
├── 1_product/                      # 产品与项目管理（轻量）
│   ├── roadmap.md                  # 各里程碑功能规划
│   ├── spec.md                     # 需求规格说明书
│   ├── tasks.md                    # 任务清单
│   └── releases/                   # 外部发布说明
│       └── v1.0.0.md               # 版本发布文档
│
├── 2_design/                       # 设计产出
│   ├── domain/                     # DDD领域模型
│   │   ├── event-storming.md       # 事件风暴记录
│   │   └── aggregates.md           # 聚合设计说明
│   ├── ui/                         # UI/UX设计
│   │   ├── ui-design.md            # UI设计文档
│   │   └── wireframes/             # 原型图目录
│   └── security/                   # 安全设计
│       └── security-design.md      # 安全设计文档
│
└── 3_development/                  # 开发规约与指引
    ├── coding-conventions.md       # 编码规范
    ├── tdd-guide.md                # TDD流程说明
    ├── api-reference.md            # API参考文档
    └── troubleshooting.md          # 问题排查指南
```

### 2.2 文档命名规范

| 文档类型 | 命名规则           | 示例                             |
|------|----------------|--------------------------------|
| 调研报告 | `NNN-主题英文名.md` | `001-harmonyos-api-research.md` |
| 审核报告 | `RNN-主题英文名.md` | `R01-research-review.md` |
| 设计文档 | `主题英文名.md`     | `architecture.md`, `spec.md`   |
| 发布文档 | `v版本号.md`      | `v1.0.0.md`                    |

---

## 三、工程架构

### 3.1 模块职责划分

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         LiveHub 项目模块架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐                                                   │
│   │ livehub-harmonyos │ ← 首个MVP开发目标                                │
│   │ (鸿蒙应用层)      │                                                   │
│   │                  │                                                   │
│   │ • UI界面 (ArkTS) │                                                   │
│   │ • 文件管理       │                                                   │
│   │ • 任务调度       │                                                   │
│   │ • 调用Core模块   │                                                   │
│   └───────┬─────────┘                                                   │
│           │                                                             │
│           │ NAPI 调用                                                    │
│           │                                                             │
│   ┌───────▼─────────┐                                                   │
│   │  livehub-core   │ ← 跨平台核心，所有平台共用                          │
│   │  (Native C++)   │                                                   │
│   │                 │                                                   │
│   │ • FormatDetector│ 格式检测                                           │
│   │ • Converter     │ 格式转换                                           │
│   │ • Metadata      │ 元数据读写                                         │
│   │ • FFmpegWrapper │ FFmpeg封装                                         │
│   │ • LibmphotoWrap │ libmphoto封装                                       │
│   └───────┬─────────┘                                                   │
│           │                                                             │
│           │ 依赖                                                         │
│           │                                                             │
│   ┌───────▼─────────┐                                                   │
│   │  Third-Party    │ ← 第三方库集成                                     │
│   │                 │                                                   │
│   │ • FFmpeg LGPL   │ 视频封装转换                                       │
│   │ • libmphoto MIT │ Google/Samsung格式处理                            │
│   │ • Adobe XMP BSD │ XMP元数据处理                                      │
│   │ • libexif LGPL  │ EXIF处理（可选）                                   │
│   └─────────────────┘                                                   │
│                                                                         │
│   ┌─────────────────┐                                                   │
│   │ livehub-android │ ← 后续扩展（可选）                                  │
│   │ (Android版)     │                                                   │
│   │                 │                                                   │
│   │ • UI界面 (Kotlin│                                                   │
│   │ • JNI调用Core   │                                                   │
│   └─────────────────┘                                                   │
│                                                                         │
│   ┌─────────────────┐                                                   │
│   │ livehub-ios     │ ← 后续扩展（可选）                                  │
│   │ (iOS版)         │                                                   │
│   │                 │                                                   │
│   │ • UI界面 (Swift │                                                   │
│   │ • Swift调用Core │                                                   │
│   └─────────────────┘                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 livehub-core（跨平台核心模块）目录结构

```
livehub-core/
├── CMakeLists.txt                  # CMake构建配置
├── include/                        # 公共头文件
│   ├── livehub/                    # LiveHub公共API
│   │   ├── format_detector.h      # 格式检测接口
│   │   ├── converter.h            # 转换接口
│   │   ├── metadata.h             # 元数据接口
│   │   ├── types.h                # 类型定义（枚举、结构体）
│   │   └── error_codes.h          # 错误码定义
│   │
│   ├── ffmpeg/                    # FFmpeg封装头文件
│   │   └── remux_wrapper.h        # Remux封装接口
│   │
│   └── libmphoto/                 # libmphoto封装头文件
│       └── motion_photo_wrapper.h # Motion Photo封装接口
│
├── src/                            # 源代码实现
│   ├── detector/                   # 格式检测模块
│   │   ├── format_detector.cpp    # 格式检测实现
│   │   ├── apple_detector.cpp     # Apple检测器
│   │   ├── huawei_detector.cpp    # 华为检测器
│   │   ├── xiaomi_detector.cpp    # 小米检测器
│   │   ├── google_detector.cpp    # Google检测器
│   │   └── samsung_detector.cpp   # Samsung检测器
│   │
│   ├── converter/                  # 格式转换模块
│   │   ├── converter.cpp           # 转换引擎实现
│   │   ├── apple_converter.cpp    # Apple转换器
│   │   ├── huawei_converter.cpp   # 华为转换器
│   │   ├── xiaomi_converter.cpp   # 小米转换器
│   │   ├── google_converter.cpp   # Google转换器
│   │   ├── samsung_converter.cpp  # Samsung转换器
│   │   └── dual_output_converter.cpp # 双输出兜底转换器
│   │
│   ├── metadata/                   # 元数据处理模块
│   │   ├── metadata_reader.cpp    # 元数据读取
│   │   ├── metadata_writer.cpp    # 元数据写入
│   │   ├── xmp_processor.cpp      # XMP处理器
│   │   ├── exif_processor.cpp     # EXIF处理器
│   │   └── quicktime_processor.cpp # QuickTime处理器
│   │
│   ├── ffmpeg/                     # FFmpeg封装
│   │   ├── remux_wrapper.cpp      # Remux封装实现
│   │   └── metadata_writer.cpp    # MOV元数据写入
│   │
│   └── libmphoto/                  # libmphoto封装
│       ├── motion_photo_wrapper.cpp # Motion Photo封装实现
│       └── extract_video.cpp       # 视频提取
│
├── libs/                           # 第三方库预编译
│   ├── ffmpeg/                     # FFmpeg库
│   │   ├── arm64-v8a/              # ARM64架构
│   │   │   ├── libavcodec.so
│   │   │   ├── libavformat.so
│   │   │   ├── libavutil.so
│   │   │   └── libswresample.so
│   │   └── x86_64/                 # x86_64架构（模拟器）
│   │
│   └── libmphoto/                  # libmphoto库
│       ├── arm64-v8a/
│       └── x86_64/
│
├── tests/                          # 单元测试
│   ├── detector_tests.cpp
│   ├── converter_tests.cpp
│   └── metadata_tests.cpp
│
├── third_party/                    # 第三方库源码（用于编译）
│   ├── ffmpeg/                     # FFmpeg源码（可选）
│   ├── libmphoto/                  # libmphoto源码
│   └── xmp-toolkit/                # Adobe XMP Toolkit源码
│
└── README.md                       # 核心模块说明
```

### 3.3 livehub-harmonyos（鸿蒙应用）目录结构

```
livehub-harmonyos/
├── entry/                          # 主模块
│   ├── src/
│   │   ├── main/
│   │   │   ├── ets/                # ArkTS源码
│   │   │   │   ├── entryability/
│   │   │   │   │   └── EntryAbility.ets  # 入口Ability
│   │   │   │   ├── pages/
│   │   │   │   │   ├── Index.ets         # 主页面
│   │   │   │   │   ├── SelectPage.ets    # 照片选择页
│   │   │   │   │   ├── ConvertPage.ets   # 转换页
│   │   │   │   │   ├── ResultPage.ets    # 结果页
│   │   │   │   │   └── SettingsPage.ets  # 设置页
│   │   │   │   ├── components/
│   │   │   │   │   ├── PhotoGrid.ets     # 照片网格组件
│   │   │   │   │   ├── ProgressIndicator.ets # 进度指示器
│   │   │   │   │   ├── FormatSelector.ets # 格式选择器
│   │   │   │   │   └── MotionPhotoPreview.ets # 动态照片预览
│   │   │   │   ├── services/
│   │   │   │   │   ├── ConversionService.ets # 转换服务
│   │   │   │   │   ├── FileService.ets    # 文件服务
│   │   │   │   │   ├── NotificationService.ets # 通知服务
│   │   │   │   │   └── BackgroundService.ets # 后台任务服务
│   │   │   │   ├── models/
│   │   │   │   │   ├── PhotoItem.ets      # 照片数据模型
│   │   │   │   │   ├── ConversionTask.ets # 转换任务模型
│   │   │   │   │   └── ConversionResult.ets # 转换结果模型
│   │   │   │   └── utils/
│   │   │   │   │   ├── Constants.ets      # 常量定义
│   │   │   │   │   ├── Logger.ets         # 日志工具
│   │   │   │   │   └── PermissionHelper.ets # 权限助手
│   │   │   │   └── native/
│   │   │   │       └── LiveHubNative.ets  # Native模块封装
│   │   │   ├── cpp/                # Native C++源码
│   │   │   │   ├── CMakeLists.txt
│   │   │   │   ├── napi_init.cpp          # NAPI模块初始化
│   │   │   │   ├── napi_bridge.cpp        # NAPI接口桥接
│   │   │   │   └── harmony_os_wrapper.cpp # HarmonyOS特定封装
│   │   │   ├── resources/          # 资源文件
│   │   │   │   ├── base/
│   │   │   │   │   ├── element/
│   │   │   │   │   │   ├── string.json    # 字符串资源
│   │   │   │   │   │   ├── color.json     # 颜色资源
│   │   │   │   │   └── media/
│   │   │   │   │       ├── icon.png       # 图标
│   │   │   │   │       └── logo.png       # Logo
│   │   │   │   └── rawfile/
│   │   │   │       └── license/           # 许可证文件
│   │   │   │           ├── ffmpeg_license.txt
│   │   │   │           ├── libmphoto_license.txt
│   │   │   │           └── xmp_license.txt
│   │   │   └── module.json5       # 模块配置
│   │   └── ohosTest/              # 测试代码
│   │       └── ets/
│   │           └── test/
│   │               └── Ability.test.ets
│   │
│   ├── build-profile.json5        # 构建配置
│   ├── hvigorfile.ts              # Hvigor构建脚本
│   ├── oh-package.json5           # 包配置
│   └── oh_modules/                # 依赖模块
│
├── feature/                       # 功能模块（可选拆分）
│   ├── conversion/                # 转换功能模块
│   ├── picker/                    # 照片选择模块
│   └── settings/                  # 设置模块
│
├── AppScope/                      # 应用全局配置
│   ├── app.json5                  # 应用配置
│   └── resources/
│
├── build-profile.json5            # 项目构建配置
├── hvigorw                        # Hvigor包装器
├── hvigorw.bat                    # Windows Hvigor包装器
└── oh-package.json5               # 项目包配置
```

### 3.4 livehub-android（可选扩展）目录结构

```
livehub-android/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/livehub/app/
│   │   │   │   ├── ui/
│   │   │   │   │   ├── MainActivity.kt
│   │   │   │   │   ├── SelectFragment.kt
│   │   │   │   │   ├── ConvertFragment.kt
│   │   │   │   │   └── SettingsFragment.kt
│   │   │   │   ├── service/
│   │   │   │   │   ├── ConversionService.kt
│   │   │   │   │   └ NotificationService.kt
│   │   │   │   └── jni/
│   │   │   │       └── LiveHubJni.kt       # JNI封装
│   │   │   ├── jni/
│   │   │   │   ├── Android.mk
│   │   │   │   ├── Application.mk
│   │   │   │   └── livehub_jni.cpp         # JNI实现
│   │   │   ├── res/
│   │   │   │   ├── layout/
│   │   │   │   ├── values/
│   │   │   │   └── drawable/
│   │   │   └── AndroidManifest.xml
│   │   └── test/
│   │   └── androidTest/
│   │
│   ├── build.gradle.kts
│   └── proguard-rules.pro
│
├── gradle/
├── build.gradle.kts
├── settings.gradle.kts
└── gradle.properties
```

### 3.5 livehub-ios（可选扩展）目录结构

```
livehub-ios/
├── LiveHub/
│   ├── App/
│   │   ├── AppDelegate.swift
│   │   ├── SceneDelegate.swift
│   │   └── ContentView.swift
│   ├── Views/
│   │   ├── SelectView.swift
│   │   ├── ConvertView.swift
│   │   ├── ResultView.swift
│   │   └── SettingsView.swift
│   ├── Services/
│   │   ├── ConversionService.swift
│   │   ├── FileService.swift
│   │   └── NotificationService.swift
│   ├── Models/
│   │   ├── PhotoItem.swift
│   │   ├── ConversionTask.swift
│   │   └── ConversionResult.swift
│   └── CoreWrapper/
│   │   ├── LiveHubCoreWrapper.swift    # Swift封装
│   │   └── livehub_core_bridge.c       # C桥接
│   │
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   ├── Localizable.strings
│   │   └── Licenses/
│   │
│   └── Info.plist
│
├── LiveHubTests/
├── LiveHubUITests/
│
├── CoreModule/                    # 集成livehub-core
│   ├── include/
│   └── libs/
│       ├── liblivehub.a           # 静态库
│       ├── ffmpeg.framework
│       └── libmphoto.framework
│
├── Podfile                        # CocoaPods依赖
├── LiveHub.xcodeproj
└── LiveHub.xcworkspace
```

---

## 四、构建配置规范

### 4.1 livehub-core CMakeLists.txt 示例

```cmake
cmake_minimum_required(VERSION 3.10)
project(livehub-core VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 配置选项
option(LIVEHUB_ENABLE_FFMPEG "Enable FFmpeg integration" ON)
option(LIVEHUB_ENABLE_LIBMPHOTO "Enable libmphoto integration" ON)
option(LIVEHUB_ENABLE_XMP_TOOLKIT "Enable Adobe XMP Toolkit" ON)

# 平台判断
if(CMAKE_SYSTEM_NAME STREQUAL "OHOS")
    set(LIVEHUB_PLATFORM "harmonyos")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(LIVEHUB_PLATFORM "android")
elseif(CMAKE_SYSTEM_NAME STREQUAL "iOS")
    set(LIVEHUB_PLATFORM "ios")
else()
    set(LIVEHUB_PLATFORM "desktop")
endif()

# 公共头文件
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

# 源文件列表
set(LIVEHUB_CORE_SOURCES
    src/detector/format_detector.cpp
    src/detector/apple_detector.cpp
    src/detector/huawei_detector.cpp
    src/detector/xiaomi_detector.cpp
    src/detector/google_detector.cpp
    src/detector/samsung_detector.cpp
    
    src/converter/converter.cpp
    src/converter/apple_converter.cpp
    src/converter/huawei_converter.cpp
    src/converter/xiaomi_converter.cpp
    src/converter/google_converter.cpp
    src/converter/samsung_converter.cpp
    src/converter/dual_output_converter.cpp
    
    src/metadata/metadata_reader.cpp
    src/metadata/metadata_writer.cpp
    src/metadata/xmp_processor.cpp
    src/metadata/exif_processor.cpp
    src/metadata/quicktime_processor.cpp
)

# FFmpeg集成
if(LIVEHUB_ENABLE_FFMPEG)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/third_party/ffmpeg/include)
    list(APPEND LIVEHUB_CORE_SOURCES
        src/ffmpeg/remux_wrapper.cpp
        src/ffmpeg/metadata_writer.cpp
    )
    link_directories(${CMAKE_CURRENT_SOURCE_DIR}/libs/ffmpeg/${LIVEHUB_PLATFORM})
endif()

# libmphoto集成
if(LIVEHUB_ENABLE_LIBMPHOTO)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/third_party/libmphoto/include)
    list(APPEND LIVEHUB_CORE_SOURCES
        src/libmphoto/motion_photo_wrapper.cpp
        src/libmphoto/extract_video.cpp
    )
    link_directories(${CMAKE_CURRENT_SOURCE_DIR}/libs/libmphoto/${LIVEHUB_PLATFORM})
endif()

# 构建动态库
add_library(livehub_core SHARED ${LIVEHUB_CORE_SOURCES})

# 链接第三方库
if(LIVEHUB_ENABLE_FFMPEG)
    target_link_libraries(livehub_core
        avformat avcodec avutil swresample
    )
endif()

if(LIVEHUB_ENABLE_LIBMPHOTO)
    target_link_libraries(livehub_core mphoto)
endif()

# 平台特定配置
if(LIVEHUB_PLATFORM STREQUAL "harmonyos")
    # HarmonyOS NDK链接
    target_link_libraries(livehub_core
        libace_napi.z.so
        libhilog_ndk.z.so
    )
elseif(LIVEHUB_PLATFORM STREQUAL "android")
    # Android JNI链接
    target_link_libraries(livehub_core log)
endif()
```

### 4.2 HarmonyOS module.json5 配置示例

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "description": "$string:module_desc",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "description": "$string:EntryAbility_desc",
        "icon": "$media:icon",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          {
            "entities": ["entity.system.home"],
            "actions": ["action.system.home"]
          }
        ],
        "backgroundModes": ["dataProcessing"]
      }
    ],
    "requestPermissions": [
      {
        "name": "ohos.permission.READ_IMAGEVIDEO",
        "reason": "$string:permission_read_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "inuse"
        }
      },
      {
        "name": "ohos.permission.WRITE_IMAGEVIDEO",
        "reason": "$string:permission_write_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "inuse"
        }
      }
    ]
  }
}
```

---

## 五、依赖管理

### 5.1 第三方库版本锁定

| 库名 | 版本 | 许可证 | 来源 | 说明 |
|------|------|-------|------|------|
| FFmpeg | 5.1.x (LGPL) | LGPL v2.1+ | 官方源码编译 | 仅启用LGPL组件 |
| libmphoto | 1.x | MIT | GitHub | Motion Photo处理 |
| Adobe XMP Toolkit | 202x.x | BSD 3-Clause | Adobe官方 | XMP元数据处理 |
| libexif | 0.6.x | LGPL | 官方源码 | EXIF处理（可选） |

### 5.2 许可证合规文件位置

```
livehub-harmonyos/entry/src/main/resources/rawfile/license/
├── ffmpeg_license.txt             # FFmpeg LGPL许可证
├── libmphoto_license.txt          # libmphoto MIT许可证
├── xmp_license.txt                # Adobe XMP BSD许可证
├── third_party_notice.txt         # 第三方库声明汇总
└── about_open_source.txt          # 关于开源许可证页面内容
```

---

## 六、版本控制与开发规范

版本控制策略、Git配置、提交规范等开发规范详见：

**文档路径**：`docs/3_development/version-control-guide.md`

**主要内容**：
- Git忽略配置
- 分支策略与命名规范
- 提交消息规范
- 版本标签规范
- 合并请求(MR/PR)规范
- 最佳实践与常见问题

---

## 七、分支策略概览

采用 `main/develop/feature` 三层分支结构：

```
main ← 稳定发布版本
  └─ develop ← 开发主分支
      ├─ feature/* ← 功能开发分支
      ├─ bugfix/* ← Bug修复分支
      └─ release/* ← 发布分支
```

详细规范见：`docs/3_development/version-control-guide.md`

---

*项目架构规划 v1.1*
*2026-05-20*