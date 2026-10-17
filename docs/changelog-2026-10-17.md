# motion-craft 技能更新日志

日期：2026-10-17 ｜ 仓库：xianhetao/agent-motion ｜ 依据：test-01 全片制作与修订轮实战复盘

## 概览

本轮将一次完整影片制作（142.5s / 4276 帧竖版口播包装片）及一轮人工反馈修订中验证过的工程经验，沉淀为技能文档，共 3 次迭代、新增约 70 条实战规则，全部通过 read-stage fail-closed 校验。

| 迭代 | 提交 | 内容 |
| --- | --- | --- |
| v1 全片制作复盘 | `dc56533` | 新增 build-pipeline 模块（6 节经验） |
| v2 修订轮复盘 | `e45da72` | 补充 7 条遮挡/退场/流程规则 |
| v3 跨平台 | `fce1869` | 新增 macOS ↔ Windows 章节 |

## v1 新增模块：build-pipeline（按需读取）

登记于 `stage-reading.json`，触发条件：**搭建 Three.js 渲染管线、准备本机环境或编排批量渲染前**。SKILL.md 模块示例同步提示。读取方式：

```sh
node scripts/read-stage.mjs --module build-pipeline
```

### 环境与依赖安装
- Node 原生 fetch 不读 `HTTPS_PROXY`：用 undici `ProxyAgent` shim 注入 dispatcher
- Playwright Chromium 卡死 → `PLAYWRIGHT_DOWNLOAD_HOST=npmmirror`；pip 走清华源
- Node 用受管版本写绝对路径，不依赖裸 `node`（系统 nvm 可能被清理）
- 中文 zip 乱码：cp437 → utf-8/gbk 解码重命名后解包
- 受管 Node 安全删除护栏会拦渲染器的批量 `rmSync`：渲染前手动清帧目录

### 渲染管线与确定性
- 本地 server + 无头 Chromium 逐帧截图 + ffmpeg 编码，帧号即真相
- `window.film` 契约：`ready` / `duration` / `fps` / `renderAt(t)` / `pickAt(t)`
- 相机一次性标定（竖版 720×1280 ↔ z=1600、fov 43.60°），舞台元素直接用设计像素
- 人物层用图程序列（非视频纹理）保证逐帧确定性；框内二次居中用离线锚点跟踪（±8 帧平滑）
- 锚点数据流式计算，防大 N 硬编码漏帧/内存被杀
- 分语义段推进：样段验收 → 逐段检查点 → 全片，不整片盲渲

### 人物层与取景
- RVM 抠像约 0.14s/帧；输出前景+alpha 双轨
- 竖版人物卡比例 0.882（11:16）而非 4:5，按素材实测最大人形高度定
- 二次取景锚定脸部而非头顶；取景窗设底部硬限防烧录字幕入前景
- 原景禁描边、换背景才描边，缩窗不改变判断

### 实现陷阱清单
- 常驻网格显隐 = 透明度乘法（opacity 与 visible 同步），不只设一次 visible
- 叠放卡片显式排 z，被接管者主动退后降透明度
- canvas 纹理先算坐标再落笔，越界绘制静默裁切无报错
- grep 对 ▸✓ 等特殊字符可能静默失败：批量改源码用脚本精确替换 + diff 验证
- 跨模块舞台上下文显式传参，作用域错误渲染时才暴露

### 音效合成与重混
- numpy 直接合成三层：动作声/提示声/强重音，事件表从语义时间线生成
- 增益/衰减/音色集中在脚本顶部常量区，改听感秒级重混
- 混音检查真峰值（≤ -1 dBTP 量级），不只看采样削波

## v2 修订轮补充（7 条）

来自 7 处人工报障（标题遮挡/文字叠字/残影/画面乱）的根因复盘：

- **盒体前脸遮挡**：BoxGeometry 有厚度，遮挡判断比的是前脸 z（center + depth/2），文字面片 z 必须大于盒体前脸
- **完全退场**：对象退场必须衰减到 0，不留 10-20% 残影底；同列纵向空间分带预约（标题带/剧名条/数据区）
- **overlay 约束**：装饰乱线/问号锁在焦点卡范围内并跟随其位置，强调符号 ≤3 个
- **安全区钳制**：常驻小窗与轮换文字共存时，排布函数内置窗体矩形钳制 + 超宽行等比缩小
- **修订流程**：报障时间点 → 渲染检查帧 → 定位根因 → 修复 → 复验 → 才整片重渲，不盲改后全片重渲

## v3 跨平台（macOS ↔ Windows）

- 工程本体已适配 Windows：`read-stage.py` 内置 PowerShell 引号分支、`start.bat`、`render.mjs` 支持 `FFMPEG_PATH` 兜底；Playwright / ONNX RVM / numpy 跨平台
- 平台绑定的只有装配层：`~/.workbuddy/skills/` symlink 是 macOS 做法，Windows 上在 WorkBuddy 内重装技能或复制 `.agents/skills/` 技能包
- Windows 注意：ffmpeg 用 `FFMPEG_PATH` 指向 exe；工作目录优先纯 ASCII 路径；`work/` 成片与工程资产可直接拷贝复用

## 配套工具与工程改动

- 新增 `work/test-01/film/check-frames.mjs`：多时间点检查帧渲染器（`node check-frames.mjs 61 82.6 99 …`），用于人工反馈的快速验证
- git 远端迁移：erduo1998-cell（上游）→ xianhetao/agent-motion（本用户仓库），本地 main 与远端对齐
- 成片修订：test-01 七处画面问题修复并重出 `film-final.mp4`（详见当日工作日志）

## 后续迭代约定

- 每轮制作/修订后，把新验证的根因按"现象 → 根因 → 规则"格式补进对应章节
- 新增参考文件必须登记 `stage-reading.json`（fail-closed，未登记直接报错）
- 文档记录的路径与镜像是实测值，换机器先验证再沿用
