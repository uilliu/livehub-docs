# LiveHub 版本控制指南

> **版本**：1.0
> **更新日期**：2026-05-20
> **适用范围**：所有LiveHub项目模块

---

## 一、Git忽略配置

### 1.1 根目录 .gitignore

```gitignore
# .gitignore

# HarmonyOS
livehub-harmonyos/entry/.preview/
livehub-harmonyos/entry/build/
livehub-harmonyos/oh_modules/
*.hap

# Android
livehub-android/.gradle/
livehub-android/app/build/
livehub-android/build/
*.apk
*.aab

# iOS
livehub-ios/Pods/
livehub-ios/*.xcuserdata
livehub-ios/DerivedData/
*.ipa

# Native
livehub-core/build/
livehub-core/libs/**/*.so
livehub-core/libs/**/*.a
livehub-core/libs/**/*.dylib
livehub-core/third_party/**/build/

# IDE
.idea/
.vscode/
*.iml

# Claude
.claude/projects/
.claude/scheduled_tasks.json

# Logs
*.log

# OS
.DS_Store
Thumbs.db
```

### 1.2 忽略规则说明

| 忽略项 | 原因 | 备注 |
|-------|------|------|
| `*.hap` | HarmonyOS应用包，编译产物 | 不提交编译产物 |
| `*.apk/*.aab` | Android应用包，编译产物 | 不提交编译产物 |
| `*.ipa` | iOS应用包，编译产物 | 不提交编译产物 |
| `*.so/*.a/*.dylib` | Native动态/静态库 | 预编译库不提交 |
| `oh_modules/` | HarmonyOS依赖模块 | 依赖通过oh-package.json5管理 |
| `.gradle/` | Gradle缓存 | 构建工具缓存 |
| `Pods/` | CocoaPods依赖 | 依赖通过Podfile管理 |
| `.idea/.vscode` | IDE配置 | 个人IDE配置不提交 |
| `.claude/projects/` | Claude项目配置 | 本地Claude配置 |

---

## 二、分支策略

### 2.1 分支结构

```
main (master)         ← 稳定发布版本
  │
  ├── develop         ← 开发主分支
  │   │
  │   ├── feature/    ← 功能开发分支
  │   │   ├── feature/detector-engine
  │   │   ├── feature/huawei-converter
  │   │   ├── feature/apple-converter
  │   │   ├── feature/ui-pages
  │   │   └── ...
  │   │
  │   ├── bugfix/     ← Bug修复分支
  │   │   ├── bugfix/ffmpeg-hevc-tag
  │   │   ├── bugfix/metadata-parsing
  │   │   └── ...
  │   │
  │   └── refactor/   ← 重构分支
  │       ├── refactor/converter-simplify
  │       └── ...
  │
  └── release/        ← 发布分支
      ├── release/v1.0.0
      ├── release/v1.1.0
      └── ...
```

### 2.2 分支命名规范

| 分支类型 | 命名格式 | 示例 | 说明 |
|---------|---------|------|------|
| **主分支** | `main` / `develop` | `main`, `develop` | 固定命名 |
| **功能分支** | `feature/<功能名>` | `feature/detector-engine` | 新功能开发 |
| **修复分支** | `bugfix/<问题描述>` | `bugfix/ffmpeg-hevc-tag` | Bug修复 |
| **重构分支** | `refactor/<重构范围>` | `refactor/converter-simplify` | 代码重构 |
| **发布分支** | `release/v<版本号>` | `release/v1.0.0` | 版本发布准备 |

### 2.3 分支工作流程

#### 功能开发流程

```
1. 从 develop 创建 feature 分支
   git checkout develop
   git checkout -b feature/detector-engine

2. 功能开发（多次提交）
   git commit -m "[livehub]feat(detector): implement format detection"
   git commit -m "[livehub]feat(detector): add Apple detector"
   ...

3. 功能完成后合并到 develop
   git checkout develop
   git merge feature/detector-engine

4. 删除 feature 分支
   git branch -d feature/detector-engine
```

#### Bug修复流程

```
1. 从 develop 创建 bugfix 分支
   git checkout develop
   git checkout -b bugfix/ffmpeg-hevc-tag

2. 修复Bug
   git commit -m "[livehub]fix(ffmpeg): resolve HEVC MOV playback issue"

3. 合合到 develop
   git checkout develop
   git merge bugfix/ffmpeg-hevc-tag

4. 删除 bugfix 分支
   git branch -d bugfix/ffmpeg-hevc-tag
```

#### 版本发布流程

```
1. 从 develop 创建 release 分支
   git checkout develop
   git checkout -b release/v1.0.0

2. 发布准备（版本号更新、文档完善）
   git commit -m "[livehub]chore(release): prepare v1.0.0 release"

3. 合并到 main 并打标签
   git checkout main
   git merge release/v1.0.0
   git tag -a v1.0.0 -m "Release v1.0.0"

4. 合并回 develop
   git checkout develop
   git merge release/v1.0.0

5. 删除 release 分支
   git branch -d release/v1.0.0
```

---

## 三、提交消息规范

### 3.1 提交消息格式

**格式模式**：
```
[livehub]<类型>(<范围>): <简短英文描述>

<主要内容分点陈述，每点20字以内>
```

**字数要求**：全部commit message控制在300字(word)以内

### 3.2 类型定义

| 类型 | 英文 | 用途 | 示例场景 |
|------|------|------|---------|
| **feat** | feature | 新功能开发 | 实现格式检测器、添加转换器 |
| **fix** | fix | Bug修复 | 修复MOV播放问题、修复元数据解析错误 |
| **docs** | documentation | 文档更新 | 更新需求规格、补充API文档 |
| **refactor** | refactor | 代码重构 | 简化转换流程、提取公共逻辑 |
| **test** | test | 测试相关 | 添加单元测试、集成测试 |
| **chore** | chore | 构建/配置/工具 | CMake配置、依赖管理 |

### 3.3 范围定义

| 范围 | 适用模块 | 示例 |
|------|---------|------|
| **detector** | 格式检测模块 | `feat(detector)` |
| **converter** | 格式转换模块 | `feat(converter)` |
| **metadata** | 元数据处理模块 | `fix(metadata)` |
| **ffmpeg** | FFmpeg封装 | `fix(ffmpeg)` |
| **ui** | UI界面 | `feat(ui)` |
| **docs** | 文档 | `docs(spec)` |
| **cmake** | 构建配置 | `chore(cmake)` |

### 3.4 提交消息示例

#### 新功能开发

```
[livehub]feat(detector): implement Apple Live Photo format detection

* Add ContentIdentifier UUID parsing
* Support HEIC+MOV file pair validation
* Unit test coverage 99%
```

#### Bug修复

```
[livehub]fix(ffmpeg): resolve HEVC MOV playback issue on iOS

* Add -tag:v hvc1 parameter
* Update codec_tag handling logic
* Verified on iPhone 14 Pro
```

#### 文档更新

```
[livehub]docs(glossary): update terminology and add vendor naming

* Add Live Photo vs Motion Photo distinction
* Define file mode terminology
* Include code naming examples
```

#### 代码重构

```
[livehub]refactor(converter): simplify format conversion workflow

* Extract common conversion logic
* Reduce code duplication by 30%
* Improve error handling
```

#### 测试相关

```
[livehub]test(detector): add integration tests for all formats

* Cover 5 format detection scenarios
* Add edge case test cases
* Achieve 95% accuracy target
```

#### 构建/配置

```
[livehub]chore(cmake): configure FFmpeg LGPL integration

* Enable LGPL-only components
* Setup dynamic linking
* Add license compliance files
```

---

## 四、版本标签规范

### 4.1 标签命名

**格式**：`v<主版本>.<次版本>.<修订号>`

| 版本类型 | 格式 | 示例 | 说明 |
|---------|------|------|------|
| **主版本** | `vX.0.0` | `v1.0.0`, `v2.0.0` | 重大版本更新、架构变更 |
| **次版本** | `vX.Y.0` | `v1.1.0`, `v1.2.0` | 功能增强、新功能添加 |
| **修订号** | `vX.Y.Z` | `v1.0.1`, `v1.0.2` | Bug修复、小改进 |

### 4.2 标签创建

```bash
# 创建带注释的标签
git tag -a v1.0.0 -m "Release v1.0.0 - MVP版本

核心功能：
* 5格式检测引擎
* 主流转换路径
* 批量转换支持
* 元数据保留

验收标准：
* 检测准确率 ≥ 95%
* 转换成功率 ≥ 95%
* 单张转换 ≤ 5秒"
```

---

## 五、合并请求(MR/PR)规范

### 5.1 MR标题格式

**格式**：`[livehub]<类型>(<范围>): <简短描述>`

示例：
- `[livehub]feat(detector): implement Apple detector`
- `[livehub]fix(ffmpeg): resolve HEVC MOV playback issue`

### 5.2 MR描述模板

```markdown
## 变更概述
<!-- 简述本次MR的目的 -->

## 变更内容
<!-- 详细列出变更项 -->
- [ ] 变更项1
- [ ] 变更项2
- [ ] 变更项3

## 测试验证
<!-- 说明如何验证变更 -->
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 真机测试通过

## 相关任务
<!-- 关联的任务ID -->
- 任务ID: T011

## 相关文档
<!-- 关联的文档 -->
- 文档: docs/superpowers/specs/017-mvp-architecture-design.md
```

---

## 六、最佳实践

### 6.1 提交频率

| 场景 | 建议 |
|------|------|
| 功能开发 | 每完成一个子功能提交一次 |
| Bug修复 | 修复完成后立即提交 |
| 重构 | 每完成一个重构点提交一次 |
| 文档更新 | 文档完成后提交 |

### 6.2 提交粒度

| 粒度 | 说明 | 示例 |
|------|------|------|
| **太小** | 单行修改单独提交 | ❌ 不推荐 |
| **适中** | 一个逻辑完整的变更 | ✅ 推荐 |
| **太大** | 多个功能混合提交 | ❌ 不推荐 |

### 6.3 分支管理

| 规则 | 说明 |
|------|------|
| **及时删除** | 功能完成后删除feature分支 |
| **保持干净** | 定期清理已合并的分支 |
| **避免长期分支** | feature分支存活时间 ≤ 1周 |
| **定期同步** | feature分支定期从develop同步更新 |

---

## 七、常见问题

### 7.1 如何撤销提交？

```bash
# 撤销最近一次提交（保留修改）
git reset --soft HEAD~1

# 撤销最近一次提交（丢弃修改）
git reset --hard HEAD~1

# 修改最近一次提交消息
git commit --amend
```

### 7.2 如何解决合并冲突？

```bash
# 1. 查看冲突文件
git status

# 2. 手动解决冲突（编辑冲突文件）

# 3. 标记冲突已解决
git add <冲突文件>

# 4. 继续合并
git commit
```

### 7.3 如何同步远程分支？

```bash
# 拉取远程更新
git fetch origin

# 合并远程develop到本地
git checkout develop
git merge origin/develop

# 或使用rebase保持提交历史整洁
git rebase origin/develop
```

---

## 八、工具配置

### 8.1 Git配置建议

```bash
# 配置用户信息
git config user.name "Your Name"
git config user.email "your.email@example.com"

# 配置默认分支名
git config init.defaultBranch main

# 配置提交消息模板
git config commit.template .git/commit-template.txt

# 配置合并工具
git config merge.tool vscode
git config mergetool.vscode.cmd 'code --wait $MERGED'
```

### 8.2 提交消息模板文件

创建 `.git/commit-template.txt`：
```
[livehub]<类型>(<范围>): <简短英文描述>

<主要内容分点陈述，每点20字以内>

# 类型：feat/fix/docs/refactor/test/chore
# 范围：detector/converter/metadata/ffmpeg/ui/docs/cmake
# 字数要求：≤ 300字
```

---

*版本控制指南 v1.0*
*2026-05-20*