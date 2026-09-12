# 安卓青龙面板

在 Android 上复刻 [青龙面板](https://github.com/whyour/qinglong) v2.21.0：运行 JavaScript / Python 脚本、定时任务、环境变量、订阅管理、本地 HTTP API、网页面板。

## 功能

- **双脚本运行时**：nodejs-mobile (Node 18) + Chaquopy (Python 3.11)，进程内执行、支持超时强停
- **定时任务**：cron 调度、失败自动重试、实时状态、批量操作、标签、置顶/排序、左滑操作
- **环境变量**：批量启停、掩码显示、修改时间、拖拽排序语义（position 中点法）
- **脚本管理**：文件树、语法高亮编辑器（撤销/重做/查找替换）、URL 导入、本地文件导入（SAF）、分享、移动/复制
- **订阅**：GitHub / Gitee / GitLab 仓库定时拉取（zip 归档），白黑名单筛选，自动创建任务
- **依赖**：Python（纯 wheel 解压）/ Node.js（npmmirror tarball）
- **通知**：任务结果推送（成功/失败分开开关），通知栏常驻统计面板
- **API**：Ktor 本地服务（默认 `127.0.0.1:5700`），路径/响应结构与青龙 v2.21.0 一致，支持 OpenAPI 令牌
- **网页面板**：`http://<ip>:5700/panel`，深色模式，任务/变量/脚本/订阅完整管理
- **数据互通**：导出/导入青龙服务器格式的 tar.gz 备份（含 database.sqlite）

## 构建

要求：JDK 17、Android SDK（platform 35 + build-tools 35 + NDK 27 + CMake 3.22）。

1. **下载 libnode.so**（约 180MB，不进 git）：
   从 [nodejs-mobile v18.20.4](https://github.com/nodejs-mobile/nodejs-mobile/releases/download/v18.20.4/nodejs-mobile-v18.20.4-android.zip)
   解压 `bin/<abi>/libnode.so`（arm64-v8a / armeabi-v7a / x86_64）到 `app/src/main/jniLibs/<abi>/`。
2. 写 `local.properties` 指向 SDK 路径。
3. `./gradlew assembleDebug`，产物在 `app/build/outputs/apk/debug/`。

Windows + 中文路径：CMake 会崩，建议镜像到 ASCII 路径构建（参考 `build.sh` 的 robocopy 方案）。

## 文档

- [`docs/API_COMPAT.md`](docs/API_COMPAT.md) —— 与青龙 v2.21.0 的接口对照表
- [`安卓脚本运行管理界面原型.html`](安卓脚本运行管理界面原型.html) —— UI 设计原型
