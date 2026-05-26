# 毕设工作日志

> 项目：数据驱动的"名山幻境"短视频生成——基于现实地理的玄幻视觉扩展
> 姓名：王琦　学号：10120212203131　指导教师：林凡
> 本文档作为论文第四章"实现过程"的原始素材存档

---

## 2026-04-21 周二：环境搭建与基础验证

### 一、本日目标

- 确定最终技术选型：基于 alibaba/lumenx 开源项目进行改造
- 完成本地开发环境搭建（Python、Node.js、FFmpeg、Git）
- 跑通 LumenX 完整前后端，验证与阿里云百炼 API 的连通性
- 以泰山神话场景为测试用例，验证剧本解析—实体抽取功能

### 二、技术选型决策

在对 alibaba/lumenx（阿里开源，145★）与 Forget-C/Jellyfish（63★）两个开源 AI 短剧/漫剧生成项目进行架构对比后，最终选择 LumenX 作为二次开发基座，主要理由：

1. **项目成熟度**：LumenX 已可完整运行；Jellyfish README 明确标注"项目原型规划中"，多数核心功能仍在开发
2. **架构契合度**：LumenX 的六步 SOP（Script → ArtDirection → Assets → StoryBoard → Motion → Assembly）与本课题"名山幻境"四段式叙事+分镜+图生视频+合成的流程高度同构
3. **模型链路完整**：原生集成阿里通义千问（Qwen）处理剧本逻辑、通义万相（Wanx）处理图像与视频生成，无需自行实现 API 适配层
4. **许可证**：MIT License，修改后二次分发无合规风险

### 三、环境搭建

#### 3.1 基础工具版本

| 工具 | 版本 | 用途 |
|---|---|---|
| Python | 3.12.4 | 后端 FastAPI 服务运行时 |
| Node.js | 20.15.0 | 前端 Next.js 构建与开发服务器 |
| Git | 2.44.0 (Windows) | 源码版本管理 |
| FFmpeg | 8.1-full | 视频合成与格式转换 |
| pnpm | 10.33.0 | 前端包管理器（替代 npm） |

#### 3.2 云服务账号

- 开通阿里云主账号并完成个人实名认证
- 开通阿里云百炼大模型服务（bailian.console.aliyun.com）
- 生成 DASHSCOPE_API_KEY 并本地配置
- 账户预充值 100 元用于后续 Wanx 图像与视频生成

#### 3.3 项目获取

- Fork：github.com/alibaba/lumenx → 个人仓库 lumenx_mountain
- Clone 至本地路径：`C:/Users/Astronaut/Desktop/Graduation/lumenx_mountain`

### 四、踩坑记录（论文素材）

本日实际投入约 4 小时，其中环境问题排查约 2.5 小时，共遇到 5 个工程性问题并全部解决。这些问题反映了开源项目二次开发中常见的适配性挑战。

#### 坑 1：中文路径导致 Python 编码错误

- **现象**：`python main.py` 启动时 Traceback，报错 `Directory 'C:\Users\Astronaut\Desktop\¤¤ҵ\mountain\lumenx_mountain\static' does not exist`
- **分析**：Windows 系统默认使用 GBK 编码传递文件路径，而项目原路径包含中文字符"毕业"，Python（使用 UTF-8）接收时发生编码错误，将"毕业"乱码为"¤¤ҵ"，导致 `StaticFiles` 挂载时路径不存在
- **解决**：将项目整体迁移至纯英文路径 `Desktop/Graduation/lumenx_mountain`
- **反思**：涉及跨语言栈（Python ↔ Node.js ↔ Shell）的项目，路径中应避免非 ASCII 字符

#### 坑 2：LumenX 桌面壳启动模式 /static 目录缺失

- **现象**：`python main.py` 后进程存活但浏览器访问 `/static/index.html` 返回 404
- **分析**：`main.py` 启动的是集成了 WebView2 的桌面壳模式，需要前端预先构建静态产物至 `static/` 目录。初次 clone 后该目录未生成，且 README 未明确说明
- **解决**：改用前后端分离的开发模式——
  - 后端：`bash start_backend.sh`（端口 17177）
  - 前端：`pnpm dev`（端口 3000）
- **收益**：分离模式更便于调试，浏览器访问即可演示，避免 WebView2 桌面窗口的调试困难；也更契合本课题"交互式原型系统"的答辩演示需求

#### 坑 3：前端依赖中潜藏阿里内网源污染（最严重）

- **现象**：`npm install --legacy-peer-deps` 持续报错 `ETIMEDOUT`，请求地址为 `registry.anpm.alibaba-inc.com`（阿里内网私有 npm 源，外网无法访问）
- **分析过程**：
  1. 首先怀疑本地 npm 配置，执行 `npm config get registry` 确认已指向 `registry.npmmirror.com`——排除
  2. 检查项目 `package.json`：`grep "anpm\|alibaba-inc"` 无输出——package.json 本身干净
  3. 根因定位：`package-lock.json` 及部分 transitive dependencies（happy-dom、whatwg-mimetype 等）的内部 package.json 硬编码了阿里内网源 URL，形成递归污染
- **解决路径**（尝试顺序）：
  1. 清理 `node_modules` 与 `package-lock.json` → 无效
  2. 切换 npm 镜像至 npmmirror → 无效（lockfile 优先于 registry 配置）
  3. 增加 `--prefer-online --legacy-peer-deps` 参数 → 无效
  4. **最终方案**：改用 pnpm 包管理器，在项目级 `.npmrc` 中配置 `registry=https://registry.npmmirror.com`，利用 pnpm 的源重写能力绕过 lockfile 硬编码
- **结果**：pnpm install 耗时 47.6 秒完成 5000+ 包的安装
- **反思**：阿里内部开源项目存在该问题的概率较高，开发者在内网环境下提交代码时未清理私有源引用。这是开源项目"从能用到好用"过程中常见的工程化缺口，本文后续若开源改造版本，将一并修复此问题

#### 坑 4：httpx 0.28 版本破坏性变更导致剧本解析失败

- **现象**：前端调用"提取实体"功能，后端返回 `剧本解析失败: Client.__init__() got an unexpected keyword argument 'proxies'`
- **分析**：httpx 0.28.0 版本移除了 `Client.__init__` 的 `proxies` 参数（改为 `proxy`），而 LumenX 依赖的 openai SDK 旧版本仍使用 `proxies=`，导致 dashscope LLM 适配器初始化时抛出 `TypeError`
- **解决**：降级 httpx 至 0.27 系列：`pip install "httpx<0.28"`
- **反思**：这是 Python 生态在 2024 年末-2025 年初的一个典型"链式塌方"事件。LumenX 的 `requirements.txt` 未对 httpx 进行版本锁定（pin），导致 pip 安装了最新的不兼容版本。工程上，项目应对所有间接依赖进行版本锁定（使用 `pip-compile` 生成完整 lockfile）

#### 坑 5：API Key 泄露风险

- **现象**：在终端操作过程中误将 API Key 作为文件名执行了 `cp .env.example sk-xxx...`
- **风险**：Key 在终端历史、聊天记录中留存，存在泄露风险
- **处置**：立即在百炼控制台禁用原 Key，重新生成新 Key；确认 `.gitignore` 包含 `.env` 项以防后续 commit 泄露
- **反思**：涉及付费 API 的密钥管理需要形成肌肉记忆——只写入 `.env`，不粘贴到任何其他位置

### 五、今日完成的最小验证

在阿里云百炼控制台开通 qwen-plus 模型调用权限后，以泰山神话场景作为测试输入：

> **输入剧本**：泰山之巅，云海翻腾。东岳大帝身着赤金朝服，手持玉笏，立于云端之上。碧霞元君凤冠霞帔，手持如意，自云中缓缓降临。两位神灵相视一笑，金光洒满山川。

Qwen-plus 实体抽取输出（全部命中）：

| 类型 | 实体 | LLM 抽取描述 |
|---|---|---|
| 角色 | 东岳大帝 | 神明形象，体型成年男性，五官端正威严，身着赤金朝服，手持玉笏 |
| 角色 | 碧霞元君 | 女神形象，体型成年女性，五官秀丽高贵，凤冠霞帔，手持如意 |
| 场景 | 泰山之巅 | 云海翻腾，云端之上，金光洒满山川，氛围神圣 |
| 道具 | 玉笏 | 玉制板状物，温润洁白，古代神灵手持礼器 |
| 道具 | 如意 | 传统祥瑞器物，柄端呈云形或心形，材质珍贵 |

LLM 输出的描述词（如"赤金朝服"、"温润洁白"、"柄端呈云形"）可直接作为后续 Wanx 图像生成环节的 Prompt 片段使用，验证了"地理—神话映射数据库"通过 LLM 结构化输出的可行性。

### 六、版本控制存档

本日完成状态已提交至 fork 仓库：

```
commit: Initial setup: env configured, first entity extraction test passed
branch: main
```

### 七、明日（4/22）计划

- [ ] Step 2 Art Direction：配置"玄幻国风"视觉风格全局提示词（正向+负向）
- [ ] Step 3 Assets：为东岳大帝、碧霞元君生成正面参考图 + 三视图 + 头像特写
- [ ] Step 3 Assets：为泰山主峰、云海、南天门、玉皇顶生成场景参考图
- [ ] Step 4 StoryBoard：将已准备的 8 镜分镜脚本导入，逐个生成关键帧
- [ ] 开始撰写论文第一章"绪论"
- [ ] 预算控制：Wanx 调用当日不超过 30 元

### 八、备注

- Key 管理：新 API Key 已存 `.env`，**勿在任何协作工具/聊天记录中粘贴**
- 后端终端需保持存活，前端开发在独立终端进行
- 截图路径约定：所有步骤截图存 `./figs/` 目录，命名格式 `MM-DD_步骤_描述.png`
- 首张存档截图：`figs/04-21_实体提取_东岳大帝_碧霞元君.png`（当前界面已截图）

---
