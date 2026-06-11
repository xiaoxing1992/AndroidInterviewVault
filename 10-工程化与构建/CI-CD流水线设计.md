---
tags: [android, 面试, 工程化, cicd, A]
difficulty: A
frequency: medium
---

# CI/CD 流水线设计

## 问题

> Android 项目的 CI/CD 流水线应该包含哪些阶段？如何设计高效的自动化构建和发布流程？常用的 CI 工具有哪些？

## 核心答案

Android CI/CD 流水线核心阶段：代码检查（lint/ktlint）→ 单元测试 → 构建 APK/AAB → UI 测试 → 安全扫描 → 分发（内测/灰度/正式）。设计原则是快速反馈（轻量检查前置）、并行执行（独立任务并行）、缓存复用（Gradle cache）。常用工具：GitHub Actions、GitLab CI、Jenkins、Bitrise、CircleCI。

## 深入解析

### 原理层

**标准流水线设计：**
```yaml
# .github/workflows/android.yml
name: Android CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  # 阶段1: 快速检查（2-3分钟）
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
      - name: Lint & Ktlint
        run: ./gradlew lintDebug ktlintCheck
      - name: Detekt
        run: ./gradlew detekt

  # 阶段2: 单元测试（3-5分钟）
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Unit Tests
        run: ./gradlew testDebugUnitTest
      - name: Upload Coverage
        uses: codecov/codecov-action@v3

  # 阶段3: 构建（5-10分钟）
  build:
    needs: [lint, unit-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build APK
        run: ./gradlew assembleRelease
      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: app/build/outputs/apk/release/

  # 阶段4: UI 测试（10-15分钟，可选）
  instrumented-test:
    needs: build
    runs-on: macos-latest  # 需要硬件加速
    steps:
      - name: Run Instrumented Tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          script: ./gradlew connectedDebugAndroidTest

  # 阶段5: 发布
  deploy:
    needs: [build, instrumented-test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Firebase App Distribution
        run: ./gradlew appDistributionUploadRelease
```

**流水线设计原则：**
```
1. 快速反馈: lint/test 前置，失败快速通知
2. 并行执行: lint 和 unit-test 并行
3. 缓存复用: Gradle cache + dependency cache
4. 环境一致: Docker 容器或固定 runner 版本
5. 安全: 签名密钥用 CI secrets，不入库
```

**Gradle 缓存配置：**
```yaml
- name: Gradle Cache
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: gradle-
```

### 实战层

**签名管理：**
```yaml
# CI 中安全处理签名
- name: Decode Keystore
  run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/release.keystore

- name: Build Signed APK
  env:
    KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
    KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
    KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
  run: ./gradlew assembleRelease
```

**发布渠道：**
- **内测**：Firebase App Distribution / 蒲公英 / fir.im
- **灰度**：Google Play 分阶段发布 / 自建灰度系统
- **正式**：Google Play Console API / 应用宝 / 各渠道市场

**版本号自动化：**
```kotlin
// build.gradle.kts
val versionCode = providers.exec { commandLine("git", "rev-list", "--count", "HEAD") }
    .standardOutput.asText.map { it.trim().toInt() }
```

**PR 检查清单：**
- 代码风格（ktlint/detekt）
- 单元测试通过 + 覆盖率不降
- APK 大小变化报告
- 依赖安全扫描（Dependabot/Snyk）
- Screenshot 测试（Paparazzi）

### 延伸问题

- [[Gradle构建加速]]
- [[多渠道打包方案]]
- [[APK签名机制-V1-V2-V3]]

## 记忆锚点

CI/CD = 快速反馈 + 自动化保障。流水线顺序：lint（秒级）→ test（分钟级）→ build → deploy。核心是让问题尽早暴露，让发布一键完成。
