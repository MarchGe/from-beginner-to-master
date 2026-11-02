# MoneyPrinterTurbo 从入门到精通

## 版本与核实信息

| 项目 | 说明 |
| --- | --- |
| 教程对象 | MoneyPrinterTurbo（开源项目 `harry0703/MoneyPrinterTurbo`，MIT 许可证） |
| 基准版本 | v1.3.9（2026 年 10 月 21 日发布） |
| 核实日期 | 2026 年 10 月 30 日 |
| 核实来源 | GitHub 官方 README 与 Releases（v1.3.7 至 v1.3.9）、仓库内 `config.example.toml`、`cli.py`、`app/controllers/v1/video.py`、`app/controllers/v1/base.py`、`app/asgi.py` |
| 适用平台 | Windows 10 及以上、macOS 11.0 及以上、主流 Linux 发行版 |
| 运行环境 | 本地部署需要 Python 3.11 及以上（推荐 Python 3.11）；GPU 非必需 |
| 版本声明 | 本文的命令、参数、配置项与默认值均对照 v1.3.9 的代码与官方说明核实。后续版本若出现行为差异，以官方 README 与 API 文档页为准 |

## 写作约定

- 文中的命令示例按运行环境分别标注为 `bash`（macOS、Linux）或 `powershell`（Windows PowerShell）。命令与输出分块展示，输出仅为节选示意。
- 全项目使用以下四种入口概念的统一缩写，各章沿用，不再重复展开：
  - **WebUI**：通过浏览器访问的图形界面，基于 Streamlit 构建；
  - **CLI**：命令行接口（Command Line Interface），即 `mpt cli` 或 `cli.py`；
  - **API**：以 HTTP 协议提供服务的接口，基于 FastAPI 构建；
  - **AI Agent**：能够读取技能文档并操作本地终端的智能代理程序，项目提供 `docs/skill/SKILL.md` 供其调用。
- `config.toml` 是全项目的核心配置文件，采用 TOML 格式。首次启动时，程序依据 `config.example.toml` 自动生成该文件；其中保存 API 密钥等敏感信息，不应提交到代码仓库或对外分享。
- 生成流水线的六个阶段名称（`script`、`terms`、`audio`、`subtitle`、`materials`、`video`）在 WebUI、CLI 与 API 三条链路中含义一致，第 1 章将给出完整解释。

---

# 第 1 章 项目全景与工作原理

无论读者希望手工产出一个精品短视频，还是搭建一条每天批量产出数十条内容的生产线，都需要先弄清楚同一件事：一个主题从输入到成片，究竟经过了哪些环节、每个环节由谁负责。本章先建立全景视角，再逐章深入，避免学习过程中只见局部、不见全貌。

## 1.1 项目定位

MoneyPrinterTurbo 是一个开源的 AI 短视频自动化生产工具。使用者只需提供视频主题或关键词，项目即可完成脚本生成、素材匹配、配音合成、字幕生成、背景音乐混音与成片渲染的全流程，最终输出符合短视频平台规格的高清视频文件。

该项目的核心能力可概括为四点：

1. **入口多元**：提供 WebUI、API、CLI 与 AI Agent 四种使用方式，既能单人手工创作，也能接入自动化系统。
2. **环节可控**：默认全自动，同时每个环节都允许替换为人工内容，例如自带脚本、自带配音、自带素材。
3. **服务商开放**：大模型、素材源、语音合成、背景音乐等能力各自对接数十家服务商，可按价格与质量自由组合，不绑定任何单一厂商。
4. **产物完整**：一次生成产出成片及配音、字幕等中间产物，支持批量多条成片、任务历史记录，以及生成设置与 API 密钥的导入、导出和恢复。

同时需要明确能力边界：该项目是自动化生产工具，而非专业剪辑工作台，不支持在时间轴上手动精修片段；脚本事实性与内容合规性由使用者把关；调用云端付费服务时，费用由使用者在各服务商平台自付。

## 1.2 核心工作流：六个流水线阶段

MoneyPrinterTurbo 的生产流程由六个串行阶段构成，阶段顺序与命名在全项目范围内统一：

```text
主题 / 关键词 / 自定义脚本
        │
        ▼
[script]    脚本站：由大模型生成台本，或直接采用使用者提供的完整脚本
        │
        ▼
[terms]     素材搜索词：由脚本提炼检索关键词（本地素材链路跳过此阶段）
        │
        ▼
[materials] 素材站：下载免费素材库素材，或使用本地文件，或调用 AI 生成
        │
        ▼
[audio]     配音站：语音合成旁白，或使用上传的音频，或生成静音音轨
        │
        ▼
[subtitle]  字幕站：依据配音时间戳生成本地字幕文件
        │
        ▼
[video]     合成站：背景音乐混音、片段拼接与转场、字幕渲染、最终编码
        │
        ▼
storage/tasks/<任务 ID>/final-1.mp4
        │                      （可选）
        ▼
[发布]  Upload-Post 服务 ──► TikTok / Instagram / YouTube Shorts
```

上述六个阶段名同时也是 CLI 的 `--stop-at` 选项取值：将 `--stop-at` 设为 `script`，命令会在脚本生成后立即停止并返回脚本内容，便于先审稿再继续；设为 `video`（默认值）则一次执行到成片完成。分阶段停止的用法将在第 11 章展开。

## 1.3 四种使用方式的选择

| 方式 | 启动入口 | 典型场景 |
| --- | --- | --- |
| WebUI | `webui.bat` / `webui.sh` / `mpt webui` | 个人创作、调整参数、预览试听、查看任务历史 |
| CLI | `mpt cli` 或源码运行 `cli.py` | 脚本化单机作业、定时批量生成、服务器无浏览器环境 |
| API | 源码运行 `main.py` 启动服务 | 接入 n8n 等自动化平台、自建前端、内部系统集成 |
| AI Agent | 向代理发送指定提示词 | 不愿手工安装配置，交由 Agent 完成安装与生成 |

四种方式读取同一份 `config.toml`，共享同一套服务商配置，因此可以在不同环节切换入口而不必重复配置。

## 1.4 免费与付费能力一览

以 v1.3.9 为基准，各环节的免费与付费情况如下：

| 环节 | 免费方案 | 付费方案（均按各平台计费） |
| --- | --- | --- |
| 大模型（脚本与搜索词） | Ollama 本地模型、Claude Code 订阅、Kimi Code 订阅、Groq、Pollinations 等免密钥或订阅渠道 | OpenAI、Claude、Gemini、DeepSeek、Kimi、通义千问等云端 API |
| 视频素材 | Pexels、Pixabay、Coverr、本地素材 | WaveSpeed、火山引擎 Seedance、OFox、秘塔 MiniMax H3、MuAPI 等 AI 文生视频；OpenAI 兼容文生图 |
| 配音 | Edge TTS（免费，无需 API 密钥） | Azure Speech、SiliconFlow、Gemini、MiniMax、ElevenLabs 等 |
| 字幕 | edge 模式（依赖 TTS 时间戳）、本地 faster-whisper | 无 |
| 背景音乐 | 内置曲库随机、本地曲目 | Sonilo、ElevenLabs AI 配乐 |
| 发布 | 手动上传 | Upload-Post 服务（订阅制，负责调用 TikTok、Instagram、YouTube 接口） |

软件本身以 MIT 许可证开源，四条使用链路均不收软件费用；成本几乎全部来自可自行选择开通或不开通的云端服务。第 6 章与第 7 章会逐一标注计费点。

## 1.5 目录结构与数据存放位置

| 路径 | 用途 |
| --- | --- |
| `config.toml` | 全项目唯一核心配置文件（由 `config.example.toml` 自动生成） |
| `storage/tasks/<任务 ID>/` | 任务产物目录，成片文件命名为 `final-1.mp4`、`final-2.mp4` 等 |
| `storage/local_videos/` | 通过 API 上传的本地视频素材 |
| `storage/bgm/` | 通过 API 上传的背景音乐文件 |
| `storage/cache_videos/` | 在线素材下载缓存 |
| `resource/songs/` | 内置背景音乐曲库目录，可添加自有音乐 |
| `resource/fonts/` | 字幕字体目录，可添加自有字体；字幕字体必须位于该目录内 |
| `models/` | 手动放置的本地模型目录（如 Whisper 模型） |

安装方式不同，根目录位置也不同：

- **一键启动包与源码部署**：上述目录均位于仓库根目录（一键包内）或克隆目录内。
- **pip 安装**：配置与生成文件保存在系统数据目录下，可在启动前设置环境变量 `MPT_HOME` 更改位置。各系统默认数据目录见第 2 章方式二的数据目录表。
- **Docker 部署**：`config.toml` 与存储目录由容器挂载，位置取决于部署时映射的宿主机目录。

所有 API 密钥一律写入 `config.toml` 或 WebUI 基础设置。该文件包含敏感凭据，应妥善保管，避免泄漏。

---

# 第 2 章 环境准备与五种安装方式择定

MoneyPrinterTurbo 提供 Windows 一键启动包、pip 安装、源码部署、Docker 部署与 Google Colab 共五种安装方式，覆盖从无命令行经验的使用者到服务器运维的多类场景。选择哪种方式，取决于使用者的系统环境、是否需要二次开发，以及是否需要与其他服务隔离部署。本章给出选择标准与五种方式的完整安装步骤，完成本章后即可进入第 3 章产出第一条视频。

## 2.1 系统要求与硬件建议

官方给出的配置基准如下：

| 项目 | 最低配置 | 推荐配置 | 理想配置 |
| --- | --- | --- | --- |
| CPU | 4 核 | 6 到 8 核 | 8 核及以上 |
| RAM | 4 GB | 8 GB | 16 GB 及以上 |
| GPU | 非必需 | 4 GB 显存及以上 | 8 GB 显存及以上 |

关于 GPU（图形处理器）的取舍，判断标准如下：

- 脚本生成、配音、字幕默认链路与生成的素材均来自云端，此时瓶颈在网络与 CPU，GPU 并非必需。
- 启用本地 `faster-whisper` 转写、或将字幕转写等重本地处理链路上线时，独立显卡会明显提升速度。
- 在吞吐与稳定性层面，若主要依赖云端服务，CPU 与内存的优先级高于 GPU。

## 2.2 基础软件准备

| 软件 | 用途 | 必要性 |
| --- | --- | --- |
| Python 3.11 | 运行环境（pip、源码部署方式必需） | 视安装方式而定 |
| Git | 克隆源码仓库 | 源码部署必需 |
| uv | Python 环境与依赖管理工具（官方推荐） | 源码部署推荐 |
| FFmpeg | 音视频处理工具，首次运行时自动下载与检测 | 自动，无需手动准备 |
| Docker Desktop | 容器运行环境 | 仅 Docker 部署需要 |

补充说明两点：

1. **路径规范**：Windows 上部署目录不应包含中文、特殊字符或空格。早期版本曾出现过含空格路径导致启动脚本误判端口的问题，v1.3.7 已修复，但规范命名仍可减少排查成本。
2. **FFmpeg**：属于开源的多媒体处理工具组，被用于配音合成、片段处理与最终编码。程序默认自动下载与检测；若检测失败，可从 `https://www.gyan.dev/ffmpeg/builds/` 手动下载，并在 `config.toml` 中设置 `ffmpeg_path` 指向可执行文件。

## 2.3 五种安装方式对比

| 序号 | 方式 | 适合人群 | 需要自备 | 升级方式 | 特点 |
| --- | --- | --- | --- | --- | --- |
| 1 | Windows 一键启动包 | 仅 Windows、不熟悉命令行 | 无 | 双击 `update.bat` | 免 Python 环境，开箱即用 |
| 2 | pip 安装官方包 | 已有 Python 环境 | Python 3.11 及以上 | `pip install --upgrade mpt` | 无需克隆仓库，命令入口统一为 `mpt` |
| 3 | 源码部署（uv） | 需要二次开发、跟随最新代码 | Python 3.11、Git、uv | `git pull` + `uv sync --frozen` | 可修改代码，功能最新 |
| 4 | Docker | 服务器部署、环境隔离需求 | Docker Desktop | 拉取新镜像 | 部署隔离，便于多实例 |
| 5 | Google Colab | 临时体验、不想装本地环境 | 浏览器 | 重新执行 Notebook | 云端免费算力，受网络与配额限制 |

## 2.4 方式一：Windows 一键启动包

一键启动包面向 Windows 用户，从 GitHub Releases 页面下载。注意 GitHub 页面其余自动生成的 `Source code (zip)` 与 `Source code (tar.gz)` 仅是源码包，只包含 `webui.bat`，不含 `start.bat` 与 `update.bat`，必须下载页面下方 **Assets** 区域的 `.7z` 压缩包。

1. 进入发布页 `https://github.com/harry0703/MoneyPrinterTurbo/releases/latest`，在 **Assets** 区域下载 `.7z` 压缩包。
2. 解压到不含中文、特殊字符、空格的目录。
3. 首次使用建议先双击 `update.bat`，将代码更新至最新版本。
4. 双击 `start.bat` 启动。启动后自动打开浏览器；若打开空白，改用 Chrome 或 Edge 打开。

## 2.5 方式二：pip 安装官方包

官方 PyPI 包名为 `mpt`，由作者 `harry0703` 维护。注意 PyPI 上名为 `MoneyPrinterTurbo` 的包由第三方维护，与本教程无关。该方式无需克隆仓库，适用于已有 Python 环境的用户。

第 1 步：创建并激活虚拟环境（虚拟环境是一套独立的 Python 依赖目录，可避免污染系统环境）。

macOS / Linux：

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

第 2 步：安装并启动。

```bash
python -m pip install mpt
mpt webui
```

`mpt webui` 启动后自动打开浏览器。需要指定端口且不自动唤起浏览器时，使用：

```bash
mpt webui --port 8503 --headless
```

| 参数 | 作用 |
| --- | --- |
| `--port` | 指定 WebUI 监听端口（示例值 `8503`） |
| `--headless` | 不自动打开浏览器 |

第 3 步：管理数据与升级。首次启动自动创建 `config.toml`；pip 安装版将数据放在系统数据目录，生成的视频保存在该目录下的 `storage/tasks/`：

| 系统 | 数据目录 |
| --- | --- |
| macOS | `~/Library/Application Support/MoneyPrinterTurbo` |
| Windows | `%LOCALAPPDATA%\MoneyPrinterTurbo` |
| Linux | `~/.local/share/MoneyPrinterTurbo`，或 `$XDG_DATA_HOME/MoneyPrinterTurbo` |

可在启动前设置环境变量 `MPT_HOME` 更改数据目录。升级时在已激活的虚拟环境中执行：

```bash
python -m pip install --upgrade mpt
```

使用 CLI 生成视频前，可执行 `mpt cli --help` 查看完整命令说明。

## 2.6 方式三：源码部署（uv）

推荐使用 uv（Astral 出品的 Python 环境与依赖管理工具）管理环境和依赖，项目支持 Python 3.11 及以上版本。

```bash
git clone https://github.com/harry0703/MoneyPrinterTurbo.git
cd MoneyPrinterTurbo
uv python install 3.11
uv sync --frozen
```

四条命令的含义：克隆仓库；进入目录；安装 Python 3.11 解释器（不影响系统默认版本）；按 `uv.lock` 锁文件精确安装依赖（参数 `--frozen` 表示严格按锁文件安装，不重新解析版本）。

若确不使用 uv，也可沿用 `venv + pip` 的传统方式：

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

| 依赖文件 | 定位 |
| --- | --- |
| `pyproject.toml` | 主依赖定义文件 |
| `uv.lock` | 锁文件，配合 `uv sync --frozen` 使用 |
| `requirements.txt` | 仅保留给旧版 pip 安装方式兼容使用 |

### 启动 WebUI

Windows（在项目根目录执行）：

```powershell
.\webui.bat
```

macOS 或 Linux:

```bash
sh webui.sh
```

启动脚本会自动优先使用项目内 `.venv` 或一键包内置 Python；若两者均不存在但已安装 uv，则自动切换为 `uv run streamlit`。启动后自动打开浏览器；若打开空白，建议改用 Chrome 或 Edge。

需要局域网内其他设备访问 WebUI 时，先将监听地址改为通配地址，再启动：

```bash
MPT_WEBUI_HOST=0.0.0.0 sh webui.sh
```

Windows CMD 对应写法为 `set MPT_WEBUI_HOST=0.0.0.0` 后再运行 `webui.bat`。

### 启动 API 服务

```bash
uv run python main.py
```

若已手动激活虚拟环境，也可直接执行 `python main.py`。

### 使用 CLI

```bash
uv run python cli.py --video-subject "人工智能如何改变日常生活"
```

完整参数说明见第 11 章，或随时执行 `uv run python cli.py --help` 查看。

## 2.7 方式四：Docker 部署

Docker 是一种容器化部署方式，可将程序连同依赖封装为镜像隔离运行。若尚未安装 Docker，先从 `https://www.docker.com/products/docker-desktop/` 下载并安装 Docker Desktop；Windows 用户可参考微软官方文档中的 WSL 与容器章节。

进入克隆目录并启动（推荐参数 `-f docker-compose.release.yml`，直接拉取 GitHub Container Registry 上的预构建镜像 `ghcr.io/harry0703/moneyprinterturbo:latest`，免去本地构建）：

```bash
cd MoneyPrinterTurbo
docker compose -f docker-compose.release.yml up
```

启动前需要完成一次配置复制，否则容器无配置可挂载：

```bash
cp config.example.toml config.toml
```

启动成功后：

- WebUI 地址：`http://127.0.0.1:8501`
- API 文档（Swagger UI）：`http://127.0.0.1:8080/docs`
- API 文档（ReDoc）：`http://127.0.0.1:8080/redoc`

说明三点：

1. 若需在本地重新构建镜像，可继续使用不带 `-f` 参数的 `docker compose up`。
2. API 默认仅允许同源网页访问。只有部署了独立网页前端、需要从其他来源直接调用 API 时，才通过环境变量 `CORS_ALLOWED_ORIGINS` 配置可信来源，例如 `http://localhost:3000,https://frontend.example.com`。curl、Postman、n8n 等服务端调用不受浏览器跨域限制。
3. 仓库内另有 `docker-compose.gpu.yml` 与 `Dockerfile.gpu`（面向本地 GPU 处理场景）、`docker-compose.claude.yml` 与 `Dockerfile.claude`（配套 Claude Code 订阅服务商），用于需要时参考其注释与端口配置。

## 2.8 方式五：Google Colab

Colab 适合无本地部署条件、想快速体验的用户。打开官方 README 中的 Colab 徽章链接（指向仓库内 `docs/MoneyPrinterTurbo.ipynb`），按 Notebook 单元格逐步运行即可；具体执行顺序与界面地址以 Notebook 中的说明为准。Colab 环境与本地安装的差异（如数据不在本机、会话随断连结束）使其只适合体验与低频使用。

## 2.9 首次启动自检

无论采用哪种方式，完成安装后按以下清单自查：

1. WebUI 打开正常；打开空白时改用 Chrome 或 Edge。
2. 项目目录下生成了 `config.toml`，且未提交到任何公开仓库。
3. 能打开 API 文档页（Docker 或手动启动 API 方式时访问 `http://127.0.0.1:8080/docs`）。
4. CLI 可正常输出帮助信息：`mpt cli --help` 或 `uv run python cli.py --help`。
5. 在 WebUI 基础设置中至少能看到大模型、素材、配音三类配置分区（第 3 章将逐一填写）。

若启动阶段报错，可先跳到第 14 章常见故障对照表核对原因，再前往各功能章节。

---

# 第 3 章 第一个视频：WebUI 最短路径实操

环境就绪后，最直接的验证方式不是阅读更多文档，而是产出一条完整的成片。本章以接近零成本的组合完成这条路径：大模型选用任意提供免费或新户额度的云端 API 或本地 Ollama，素材选用免费的 Pexels，配音选用免费的 Edge TTS（无需 API 密钥），字幕与背景音乐均走默认配置。完成本章后，读者将对脚本、素材、配音、字幕、合成的五个环节产出形成直观认识，后续各章再逐一深入。

## 3.1 准备两个密钥

本章唯一需要注册的是两个 API 密钥（API Key 是服务商用于识别调用者身份与计费的字符串凭证）：

| 密钥 | 注册地址 | 费用情况 |
| --- | --- | --- |
| 大模型 API Key | 以下任一：DeepSeek `https://platform.deepseek.com/api_keys`、Kimi `https://platform.kimi.com`、OpenAI `https://platform.openai.com/api-keys`、通义千问 `https://dashscope.console.aliyun.com/apiKey`，或本地运行 Ollama 无需密钥 | 多数服务商提供注册赠送额度，以各平台页面为准 |
| Pexels API Key | `https://www.pexels.com/api/` 免费注册 | 免费使用，存在速率限制 |

在 WebUI 基础设置中分别保存上述两把密钥。WebUI 的基础设置面板即 `config.toml` 的图形界面，保存后对应写入配置文件中的服务商分区，因此换终端或重启后依然生效。

## 3.2 设置大模型服务商

WebUI 基础设置中包含大模型服务商选择区块。配置思路如下：

1. 在服务商下拉框中选择已注册的平台（例如 DeepSeek 或 Kimi）。
2. 粘贴对应 API Key。
3. 模型名与接入地址留空即使用项目内置默认值；默认模型与地址由项目维护在 `app/models/llm_provider.py` 的服务商注册表中。
4. 保存后执行一次连接测试（或直接进入生成环节验证），连接错误时 WebUI 会给出明确提示。

大模型服务商的完整清单与选型对比见第 4 章。

## 3.3 设置视频素材源

素材源即成片画面的来源。本项目默认素材源为 `pexels`：

1. 将 Pexels API Key 填入基础设置的素材区块。
2. 支持填写多个密钥（英文半角双引号，英文逗号分隔，形如 `["key-1", "key-2"]`），密钥用尽或被限流时自动轮换。
3. 保持其他素材参数为默认值即可：片段最长时长 5 秒、播放速度 1.0、画面适配 `cover`（裁剪填满，本地或在线素材与画幅方向不一致时使用；详见第 9 章）。

## 3.4 保持配音默认值

配音服务商选择界面默认 `Azure TTS V1`，该名称对应的底层是 **Edge TTS**：微软免费语音合成服务，无需任何 API 密钥。本章保持默认，即可直接生成简体中文配音。界面提供单音色试听与完整配音预览；音色即声音角色（性别、口音、情感风格的具名组合）。界面所称 Azure TTS V1 与 Azure TTS V2 是两个不同体系：V1 免费；V2 是 Azure Speech 云服务，需要 Azure 控制台申请的语音密钥与区域标识，详见第 7 章。

## 3.5 提交生成并观察流程

完成配置后回到生成页面：

1. 在主题框输入示例主题："人工智能如何改变日常生活"。语言选择简体中文，段落数量保持 1。
2. 视频规格保持竖屏 `9:16`（分辨率 1080×1920，适配抖音、快手、视频号与 YouTube Shorts 等竖屏平台）。
3. 数量保持 1，点击生成按钮。

生成期间页面持续输出任务日志。v1.3.8 起，素材下载、片段处理与最终渲染阶段都有持续的进度与细化日志，后台线程产生的日志也会一同显示，因此长时间停留在同一百分比时可以从日志行判断是否仍在推进。

## 3.6 查看与导出

成片完成后，WebUI 默认会自动打开任务目录（该行为对应配置项 `open_task_folder_on_completion`，默认开启，可在设置中关闭）。任务目录位于 `storage/tasks/<任务 ID>/`，主要产物包括：

| 文件 | 说明 |
| --- | --- |
| `final-1.mp4` | 成片本体；批量生成多条时依次为 `final-2.mp4` 等 |
| 配音音频文件 | 在此任务目录内，文件名以任务实际产物为准 |
| 字幕文件 | 同一目录内的本地字幕文件 |

产物命名与目录布局以任务目录内实际文件为准，`final-<编号>.mp4` 是稳定的成片命名规则。

## 3.7 首验常见问题检查点

| 现象 | 排查方向 | 相关章节 |
| --- | --- | --- |
| 提示模型连接失败 | 核对服务商选择与 API Key 是否匹配；多数情况是密钥误填或粘贴带空格 | 第 4 章 |
| 素材下载失败或报 Pexels 限流 | 换用免费具备无限量的 Pixabay 或 Coverr，或配置素材请求代理 | 第 5 章、第 14 章 |
| 配音阶段失败或停顿 | 核对 Edge TTS 网络可达性；若日志出现 403，先升级至 v1.3.9（该版本已修复 Edge TTS 403 问题） | 第 7 章 |
| 完全没有画面 | 确认素材源仍为 `pexels` 且密钥有效；检查 WebUI 日志中 `materials` 阶段记录 | 第 5 章 |

完成第一个视频后，流水线全貌与成本结构已经具体可感。后续章节按流水线顺序逐一讨论每个站点的参数、服务商与调优方法。

---

# 第 4 章 脚本站：大模型选型、提示词调优与自定义脚本

脚本是一切后续环节的底稿：素材搜索词由脚本提炼，配音按脚本朗读，字幕按配音对齐，成片时长基本由脚本决定。脚本质量不高时，后续所有环节的优化都会被放大为整体成片的差距。本章讨论大模型服务商的选型与接入方法、脚本生成参数、自定义脚本与停顿标签的写法，完成后即可稳定产出信息充实、可直接配音的脚本。

## 4.1 脚本阶段的关键概念与字段对应

脚本周期的输入输出涉及六个核心字段，WebUI 界面、`config.toml` 的 `[ui]` 段与 CLI 参数一一对应：

| 字段（CLI/API） | WebUI 中的概念 | 说明 |
| --- | --- | --- |
| `video_subject` | 主题、关键词 | 脚本生成的触发输入，例如"人工智能如何改变日常生活" |
| `video_script` | 台本、文案 | 提供完整脚本时跳过大模型生成 |
| `video_language` | 脚本语言 | 语言代码，如 `zh-CN`、`en-US`；默认自动检测 |
| `paragraph_number` | 段落数量 | 生成段落数，取值 1 至 10，默认 1 |
| `video_script_prompt` | 生成要求、附加提示词 | 传达风格与内容约束的补充要求 |
| `video_terms` | 素材搜索词 | 由脚本提炼或使用者手工指定，交给素材阶段检索 |

与"脚本"相关的两个易混概念在此一并定义：

- **提示词（Prompt）**：发送给大模型的指令文本。`video_script_prompt` 是在默认提示词基础上追加的要求。
- **系统提示词（System Prompt）**：定义大模型角色与写作规范的全局指令。`custom_system_prompt` 提供后会整体替换项目内置的系统提示词，适合需要固定文风或栏目人设的场景。

## 4.2 大模型服务商的分类与选型

项目的服务商接入分为三类，配置键统一写在 `config.toml` 的 `llm_provider` 分区，默认值为 `moonshot`（Kimi）。各服务商的默认模型名与接入地址维护在 `app/models/llm_provider.py` 注册表中；`model_name` 与 `base_url` 留空即跟随注册表默认值，仅在需要覆盖时填写。

| 类别 | 代表服务商 | 密钥申请地址 |
| --- | --- | --- |
| 原生厂商 API | Kimi（Moonshot）、OpenAI、Anthropic Claude、Google Gemini、DeepSeek、阿里云通义千问、火山引擎方舟、xAI Grok、MiniMax、小米 MiMo、讯飞星辰、魔搭 ModelScope | Kimi `platform.kimi.com`；OpenAI `platform.openai.com/api-keys`；Claude `platform.claude.com/settings/keys`；Gemini `aistudio.google.com/app/apikey`；DeepSeek `platform.deepseek.com/api_keys`；通义 `dashscope.console.aliyun.com/apiKey`；火山引擎 `console.volcengine.com/ark`；Grok `console.x.ai`；MiniMax `platform.minimaxi.com`（中国大陆）/`platform.minimax.io`（国际）；MiMo `platform.xiaomimimo.com` |
| 聚合与统一网关 | 胜算云、APIMart、AIHubMix、AIML API、EvoLink、OpenRouter、API Route、Cloudflare AI Gateway、OneAPI、LiteLLM，以及 Infistar、Jalapeno Cloud、Fluxion AI、Cheaper Inference、Requesty、FutureInfra、Y-API、Opper、onomeo、EmpirioLabs、Runcrate、CloudGPU 等 OpenAI 兼容网关 | OpenRouter `openrouter.ai/settings/keys`；其余按 `config.example.toml` 内各服务商注释填写 |
| 订阅与本地渠道 | Claude Code（Claude 订阅）、Kimi Code（Kimi 订阅）、OpenAI Codex 与 ChatGPT 供应商、Ollama 本地部署、Groq、Pollinations AI | 无需 API 密钥的渠道见 4.3 节 |

选型建议（均为参考性建议，理由如下）：

1. 预算敏感时优先选择提供注册赠送额度或低单价的厂商直连，例如 DeepSeek 与 Kimi；聚合平台适合需要同时调用多家模型、统一记账的场景。
2. 订阅类渠道（Claude Code、Kimi Code）复用已有订阅额度，适合有现成订阅的个人用户。
3. 隐私或离线需求场景选择 Ollama 本地模型，脚本质量取决于本地模型能力。
4. 脚本属于低频、单次输出量小的调用，模型档次对成本影响有限，但对成片质量影响显著；此环节不宜压缩预算。

## 4.3 订阅与本地渠道的接入要点

本节逐一说明无需传统 API 密钥的四种方式。

**Kimi Code 订阅**：v1.3.9 起，无需 API 密钥，在 WebUI 的设置区进入大模型设置完成一次浏览器授权登录（OAuth 设备授权，分中国大陆与全球平台），凭证随后自动保存并自动续期；默认模型为 `kimi-for-coding`，调用计入 Kimi Code 套餐用量，配额使用完后自行等待刷新或更换渠道。

**Claude Code 订阅**：复用本地已登录的 `claude` CLI（Claude 命令行工具）而非 API 密钥。

- 前提：运行 MoneyPrinterTurbo 的机器上已安装 `claude` CLI，且已在宿主机交互式登录完成认证。
- Docker 部署时无法交互式登录，需用长期凭证：在宿主机执行 `claude setup-token` 生成长期令牌，并导出为环境变量 `CLAUDE_CODE_OAUTH_TOKEN`。
- `claude_code_model_name` 留空时使用 CLI 当前默认模型，也可显式指定（配置注释中的示例为 `claude-opus-5`、`claude-sonnet-5`）。仓库还提供了配套的 `Dockerfile.claude` 与 `docker-compose.claude.yml`。

**OpenAI Codex 与 ChatGPT 供应商**：通过设备码或浏览器登录复用 ChatGPT 订阅额度，无需单独的 API 密钥。注意配置注释中的限制：浏览器登录要求浏览器与 Python 运行时在同一台计算机上，远程或 Docker 环境不支持该登录流程。

**Ollama 本地部署**：Ollama 是本机运行开源大模型的工具。在本机用 `ollama pull` 下载模型后，`ollama_model_name` 填写 `ollama list` 显示的模型名即可；本次素材仓库密钥等其他配置照常填写，脚本链路的成本为零。

## 4.4 脚本生成参数的用法

参数组合直接影响脚本的段落结构与风格。下列命令中各参数均在首次出现处解释过或已在 4.1 节的字段表中列出：

```bash
uv run python cli.py --video-subject "人工智能如何改变日常生活" \
  --video-language zh-CN \
  --paragraph-number 2 \
  --video-script-prompt "面向普通观众的科普内容，语气平实，避免专业术语，全篇控制在 60 秒左右"
```

使用要点：

1. `paragraph_number` 决定脚本段落数；段落越多，脚本与成片越长。短视频常用取值为 1 至 3。
2. `video_script_prompt` 适合逐次微调，如限定受众、口吻、时长倾向与叙事结构；内容为自由文本。
3. `custom_system_prompt` 整体替换内置系统提示词，适合固定栏目统一文风；替换后默认的脚本生成规范不再生效，需在自定义内容中自行覆盖。
4. 成片时长并非直接设定值，而是由"脚本朗读时长 + 片段参数"共同决定。估算依据是大致按中文每秒 4 至 5 字朗读（该数值仅为经验参考），并叠加以下因素：片段时长上限、播放速度与是否衔接转场。

区分说明：以上第 1 至 3 点为项目事实；第 4 点朗读速率与时长估算属于经验建议，实际以生成的配音与字幕时间轴为准。

## 4.5 自定义脚本与多语言改写

两种情况下建议绕开自动生成、直接提交完整脚本：文案已经审校定稿（如公众号文章改编）、或者需要严格口径的内容（产品宣传、品牌规范）。直接在 WebUI 的脚本框粘贴全文，或在 CLI 使用：

```bash
uv run python cli.py --video-script "这里粘贴完整脚本全文" \
  --video-terms "城市,清晨,通勤"
```

其中 `--video-terms` 以英文逗号分隔提供素材搜索词，跳过自动提炼环节；具体配合方式见第 11 章。

项目也支持 AI 改写多语言脚本：已有中文文案时，可提示大模型改写为其他语言的同内容脚本，再按目标语言的音色生成配音。

## 4.6 停顿标签

脚本支持原生停顿标记，写作格式如下（两种写法等价）：

```text
一个错误的习惯，可能悄悄毁掉三个好习惯。[pause: 2s] 先从观察现在的生活开始，改变并不遥远。
```

使用约束如下，均为 v1.3.7 引入并被后续版本延续的行为：

1. 停顿标签仅在使用 Azure TTS V1（Edge TTS）配音时生效。
2. 单次停顿时长范围为 0.1 至 10 秒；程序会校验标签合法性，无效片段会中止生成，而不是静默产出残缺配音。
3. 标签插入位置同时影响配音与字幕时间轴，两者保持同步。

## 4.7 脚本写作要点与自检

以下为面向短视频体裁的写作建议（建议性质，非程序行为）：

1. **五段框架**：参与钩子、展开、转折、实例、收尾，各占一至两句；段落数即框架的骨架。
2. **搜索词友好**：主题与用词尽量贴近可检索的现实事物（如"摩天楼""咖啡""深海"），避免高度抽象或专有概念，否则素材匹配结果会偏离。
3. **事实自查**：脚本由大模型生成，可能存在错误与虚构信息；涉及数据、引言、人物的内容务必人工核对后再发布。
4. **口径一致**：`custom_system_prompt` 固定角色设定时，注意检查生成结果是否符合要求的语气与称谓。

---

# 第 5 章 素材站：免费素材库、本地上传与素材策略

一件不合格的素材会把精心打磨的脚本直接拉低为边缘内容：与脚本无关的画面会让观众怀疑信息可信度。本章先梳理全部素材来源及其计费属性，再分述免费素材库配置、素材匹配规则与本地素材上传，这些是自建素材合规性与成片质量的基础。

## 5.1 素材源总览

素材源通过 `video_source` 配置项选择，v1.3.9 支持 13 种形式；CLI 与 WebUI 的公开范围略有差异（CLI 直接列出以下取值）：

| 取值 | 来源与计费说明 | CLI 支持 |
| --- | --- | --- |
| `pexels` | Pexels 免费素材库，需 API Key | 支持 |
| `pixabay` | Pixabay 免费素材库，需 API Key | 支持 |
| `coverr` | Coverr 免费素材库，需 API Key | 支持 |
| `local` | 本地上传图片与视频，不计费 | 支持 |
| `wavespeed` | WaveSpeed AI 文生视频，按段计费 | 支持 |
| `volcengine_seedance` | 火山引擎 Seedance，按段计费 | 支持 |
| `ofox` | OFox 多模型文生视频，按段计费 | 支持 |
| `metaso_minimax` | 秘塔代理的 MiniMax H3，按段计费 | 支持 |
| `muapi` | MuAPI 异步文生视频，按段计费 | 支持 |
| `infistar` | 无限星河 AI 视频，按任务计费 | 支持 |
| `jalapeno` | Jalapeno Cloud AI 视频，按任务计费 | 支持 |
| `openai_image` | OpenAI 兼容文生图接口，按图片计费 | 支持 |
| `codex_image` | 通过 ChatGPT 登录调用的图像能力 | 支持 |

其中 `loomloom`（胜算云付费源）仅在 WebUI 可用且每次生成前需在界面显式确认费用，CLI 公开清单中不包含该来源。付费来源的详细配置属于第 6 章；本章聚焦免费与本地来源。

## 5.2 免费素材库的配置

Pexels、Pixabay、Coverr 均免费，仅需在各平台注册并申请开发者密钥。

| 素材源 | 注册与密钥地址 | 配置键 |
| --- | --- | --- |
| Pexels | `https://www.pexels.com/api/` | `pexels_api_keys` |
| Pixabay | `https://pixabay.com/api/docs/` | `pixabay_api_keys` |
| Coverr | `https://coverr.co/developers` | `coverr_api_keys` |

三个配置键均为数组型字段，支持密钥轮换：可填写多把密钥，用完或被限流时自动轮换。格式要求为英文半角双引号，多把密钥用英文逗号分隔，形如 `["key-1", "key-2"]`。

影响检索吞吐的配置项为 `material_concurrency`（素材并发数，控制同时检索与下载的请求数）：

- 取值仅允许 1、2、4、6、8，程序按 8 封顶；WebUI 中出现其他值时会被重置为 1。
- 默认值为 1（串行处理）。数值越大下载越快，但越容易触发素材平台的速率限制。免费密钥以 1 或 2 起步为宜，出现限流报错后回落到 1。

## 5.3 素材匹配行为

脚本提交后进入 `terms` 阶段，由大模型将脚本内容提炼为若干搜索词（search terms，素材检索关键词），随后逐词检索素材。匹配阶段的关键规则如下：

1. **关键词优先顺序**：配置项 `match_materials_to_script` 默认关闭，素材选择存在随机性；开启后，搜索词与素材顺序将贴近脚本的叙事顺序，减少随机性。
2. **批量去重**：同一任务批量生成多条成片时，v1.3.7 起会减少 Pexels、Pixabay、Coverr 与本地素材之间的重复选用；素材池不足时仍会复用并给出提示。
3. **片段裁剪**：`video_clip_duration`（片段最长时长，秒）默认 5 秒，控制单条素材被截取的上限；`video_clip_speed`（播放速度倍率）默认 1.0，取值 0.5 至 2.0。
4. **画幅适配**：素材与目标画幅方向不一致时的处理方式由 `video_fit_mode` 决定：`cover` 居中裁剪填满（默认）；`contain` 保留完整画面并补黑边。该差异对横竖互转的观感影响显著，详见第 9 章。

## 5.4 本地上传素材

`video_source = local` 表示全部画面来自使用者自备的图片与视频。上传路径按使用方式区分：

- **WebUI**：在生成页面的素材区块上传图片或视频文件，程序自动存入 `storage/local_videos/`。
- **API**：调用 `POST /api/v1/video_materials` 上传（见第 12 章），返回存储后的文件名。
- **CLI**：`--video-source local --video-materials "./1.mp4,./2.mp4"`，路径以英文逗号分隔；相对路径相对当前工作目录解析，兼容回退到 `storage/local_videos`。

v1.3.8 对本地素材处理做了多方面增强：补齐支持的上传格式（包括多图相机 JPEG）、改进图片解码与手机照片方向识别、支持 CMYK 色彩，并会拒绝被误判为封面的专辑插图。v1.3.9 又加强了本地上传的启动预检，格式或路径有问题时在任务开始前即报错。

图片素材的动态效果由 `image_motion` 控制：`zoom` 为缓慢放大（默认），`static` 为静止展示。该选项为 v1.3.9 引入，此前版本图片一律应用缓慢放大。

## 5.5 素材使用合规提示

以下事项在发布前应主动确认：

1. Pexels、Pixabay、Coverr 的许可条款以各平台官方说明为准；涉及人物肖像、品牌标识时尤其注意。
2. AI 生成的视频与图片素材涉及各生成平台的版权与服务条款；发布环节还面临平台对 AI 内容标识的自有政策。
3. 对商用与长期运营的账号，本地素材的版权状况最可控，建议将其作为核心池，免费素材库作为补充。

---

# 第 6 章 素材站进阶：AI 文生视频与文生图

免费素材库覆盖面广但同质化明显，热门话题、特定风格的画面往往没有现成素材。AI 素材来源可按脚本关键词直接生成全新画面，代价是每次生成计费。本章任务是把"何时用 AI 素材、用哪一家、如何避免误触扣费"交代清楚。

## 6.1 计费确认机制

付费 AI 素材来源全部按提交计费，且生成结果可能因模型失败而不理想。项目为此设置了显式确认机制：CLI 侧调用付费素材源时必须携带对应的 `--confirm-*-charge` 参数，WebUI 侧则在界面弹出费用确认；未确认时任务直接拒绝，不会向服务商发出请求。

| CLI 确认参数 | 对应 `video_source` |
| --- | --- |
| `--confirm-wavespeed-charge` | `wavespeed` |
| `--confirm-seedance-charge` | `volcengine_seedance` |
| `--confirm-ofox-charge` | `ofox` |
| `--confirm-metaso-minimax-charge` | `metaso_minimax` |
| `--confirm-muapi-charge` | `muapi` |
| `--confirm-infistar-charge` | `infistar` |
| `--confirm-jalapeno-charge` | `jalapeno` |

批量清单模式下，上述确认参数作为 CLI 全局默认值传入；单任务与批量正式提交前均会执行预检。

## 6.2 WaveSpeed AI

WaveSpeed 提供按提示词生成视频片段的服务。核心配置项（`config.toml` 的 `[app]` 段）：

| 配置项 | 说明 |
| --- | --- |
| `wavespeed_api_keys` | API 密钥列表，规则同 5.2 节 |
| `wavespeed_text_to_video_model` | 文生视频模型 ID，默认 `bytedance/seedance-2.0-fast/text-to-video`；要求模型支持 `prompt / aspect_ratio / duration` 参数 |
| `wavespeed_min_duration` / `wavespeed_max_duration` | 片段时长下限与上限，默认 4 至 15 秒；超出范围自动夹取后提交，多余的画面在剪辑时被裁掉 |

## 6.3 火山引擎 Seedance

火山引擎方舟的官方异步视频生成接口。核心配置项：

| 配置项 | 说明 |
| --- | --- |
| `volcengine_seedance_api_key` | 在 `https://console.volcengine.com/ark/region:ark+cn-beijing/apikey` 创建；留空时按顺序回退使用 `VOLCENGINE_ARK_API_KEY` 环境变量或 `volcengine_api_key` |
| `volcengine_seedance_base_url` | 默认 `https://ark.cn-beijing.volces.com/api/v3` |
| `volcengine_seedance_model` | 默认 `doubao-seedance-1-0-pro-250528`，支持文生视频、2 至 12 秒片段与 1080p；也可替换为账号内其他可选模型或 Endpoint ID |
| `volcengine_seedance_resolution` | `480p`、`720p` 或 `1080p`，默认 `1080p` |
| `volcengine_seedance_watermark` | 是否添加水印，默认关闭 |

## 6.4 OFox 多模型文生视频

OFox 以同一密钥接入 Seedance、Wan 等多款文生视频模型。值得注意的行为：

| 配置项 | 说明 |
| --- | --- |
| `ofox_api_key` | 在 `https://ofox.ai` 创建；留空时使用 `OFOX_API_KEY` 环境变量 |
| `ofox_text_to_video_model` | 模型列表可通过 `GET /v1/models` 获取，默认 `bytedance/seedance-2.0-fast` |
| `ofox_resolution` | 默认 `720p`；合法取值取决于所选模型（默认模型支持 `480p` 与 `720p`），非法值在下单前即被拒绝 |
| `ofox_provider` | 上游供应商锚定，默认 `byteplus`（国际）与内容策略一致性优先；设为 `volcengine` 使用中国大陆链路。注意锚定范围按模型划分：更换为非 Seedance 模型（如 `alibaba/wan-2.7`）后，需同步修改供应商锚定值或设为空串，否则请求以 `provider_type_unavailable` 被拒，不会创建任务、不会扣费 |

## 6.5 秘塔 MiniMax H3

通过秘塔科技代理的 MiniMax H3 异步视频接口，支持 `768P` 与 `2K` 两档分辨率、4 至 15 秒片段与三种画幅。注册与密钥地址为 `https://metaso.cn/minimax-h3/?s=MPT`。特别注意：`mk-` 开头的密钥属于独立凭证体系，不能与 MiniMax 语言模型的密钥混用。

## 6.6 MuAPI

MuAPI 提供公开的异步文生视频端点。三个关键点：

1. `muapi_api_key` 在 `https://muapi.ai` 创建；`muapi_base_url` 必须以 `https://api.muapi.ai/api/v1` 为根。
2. `muapi_video_endpoint` 选定服务端点，预算型默认值为 `seedance-lite-t2v`，支持 480p、720p、1080p 与 3 至 12 秒片段。
3. 按提交计费，WebUI 与 CLI 均要求显式费用确认。

## 6.7 胜算云与无限星河、Jalapeno 等渠道

- **胜算云（loomloom）**：该付费来源仅 WebUI 可用，每次生成前需在界面确认费用。胜算云的 LLM 服务（OpenAI 兼容）则另行填写 `shengsuanyun_api_key` 等 LLM 配置键。
- **无限星河 Infistar**：`infistar_video_api_key`、`infistar_video_base_url`、`infistar_video_model` 独立于其 LLM 配置，默认模型 `doubao-seedance-2-0-fast-260128`（5 秒），可选 `MiniMax-H3`（6 秒）与 `kling-v2-6`（5 秒，取决于账号线路）。注意 `MiniMax-H3` 当前线路返回横屏素材，竖屏成片会按画面适配模式裁剪或补边。
- **Jalapeno Cloud**：`jalapeno_video_api_key`、`jalapeno_video_base_url`、`jalapeno_video_model` 独立配置，默认模型 `Doubao-Seedance-2-5-260628`、分辨率 `480p`；密钥未单独配置时自动复用其 LLM 密钥，但接入地址不复用 LLM 的 `/v1` 地址。

## 6.8 OpenAI 兼容文生图

`openai_image` 不检索现成素材，而是按脚本关键词逐个生成图片，再渲染为动态视频片段。它兼容任何实现了 OpenAI `/images/generations` 协议的端点：云端服务（例如 OpenAI 官方接口）、自建中转、本地图像服务均可接入。

| 配置项 | 说明 |
| --- | --- |
| `openai_image_base_url` | 兼容端点地址，例如官方 `https://api.openai.com/v1` 或本地 `http://127.0.0.1:8080/v1` |
| `openai_image_api_keys` | 密钥列表；本地无需鉴权的网关留空即可，此时不发送授权头 |
| `openai_image_model` | 图像模型 ID |
| `openai_image_size` | 图片尺寸留空时按画幅自动选择：竖屏 `1024x1536`、横屏 `1536x1024`、方形 `1024x1024`；本地网关可提交任意自定尺寸，如 `1080x1920` |
| `openai_image_prompt_template` | 提示词模板，`{term}` 占位符注入关键词，例如 `cinematic photo of {term}, photorealistic` |

v1.3.9 起附带可选的本地 SDXL 图像服务（仓库 `docs/local-image-server/README.md` ），可通过 OpenAI 兼容接口为文生图链路提供本地算力，支持空闲时释放显存。上述默认按画幅的尺寸逻辑对官方 OpenAI 端点有效，本地网关若需精确输出分辨率，应显式设置 `openai_image_size`。

## 6.9 成本控制要点

AI 素材的计费特征是"按提交扣费"，即使后段剪辑弃用已生成的片段，费用也已经发生。控制建议如下（建议性质）：

1. 批量投产前，先用最低分辨率与最短时长试跑脚本链路，确认搜索词与画面质量。
2. 付费科技投入重点留在开头镜头与高信息密度段落，其余用免费素材库兜底。
3. 关注轮询参数（`*_poll_interval`、`*_run_timeout`）即可理解任务排队时长，无需单独付费调整。
4. `codex_image`（图像能力）复用 ChatGPT 登录凭据与套餐总额度，目录候选列表并不保证图像工具的可访问性；订阅用量重叠业务时应留意额度。

---

# 第 7 章 配音站：从免费 Edge TTS 到音色复刻

配音直接决定观众的听觉体验，也是字幕时间轴的基准，因此配音方式的选择与稳定性排查是高频问题。本章把三种配音模式、十余家语音合成服务商的接入要点、语速与音量的数值规则，以及音色复刻的操作约束一并交代清楚。

## 7.1 三种配音模式

配音方式由 `voice_mode` 控制，取值三种：

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| `tts` | 调用语音合成服务自动生成旁白 | 默认模式，全自动链路 |
| `upload` | 使用使用者上传的音频文件 | 已有专业录音，要求特定音质 |
| `none` | 生成静音音轨，成片无旁白 | 音乐类视频、画面主导内容 |

注意一点：即使选择无配音，流水线仍会生成占位的静音音轨，以维持字幕与剪辑时间轴的一致性；成片时长逻辑不会因此改变。

## 7.2 Edge TTS：默认免费方案

界面与配置所称 **Azure TTS V1** 的底层即 Edge TTS：微软提供的免费语音合成服务，无需 API 密钥，默认开启即用。要点如下：

1. CLI 未显式指定音色时的默认值为 `zh-CN-XiaoxiaoNeural-Female`（简体中文女声"晓晓"）。
2. 可用音色列表见仓库文件 `docs/voice-list.txt`；界面提供按音色试听与整段配音预览。
3. 单次流式请求超时由 `edge_tts_timeout` 控制，默认 30 秒，网络较慢时可调大；设为 0 表示禁用超时。
4. 若日志出现 Edge TTS 相关 403 失败，先升级至 v1.3.9（该版本更新了 Edge TTS 依赖并修复该问题）。

## 7.3 云端与自托管语音服务一览

| 服务商 | 配置分区 | 关键配置项与说明 |
| --- | --- | --- |
| Azure Speech | `[azure]` | `speech_key` 与 `speech_region`；与 Azure OpenAI 的密钥是两套体系，不可混用 |
| SiliconFlow | `[siliconflow]` | `api_key` |
| Google Gemini | Gemini 凭据 | 按 WebUI 界面提示填写对应凭据 |
| 小米 MiMo | 复用 `[mimo]` 分区密钥 | `mimo_tts_model_name` 默认 `mimo-v2.5-tts`；`mimo_tts_style_prompt` 默认要求自然、清晰、适合短视频旁白 |
| MiniMax | `[minimax_tts]` | `model_id` 默认 `speech-2.8-hd`；密钥与接入地址为空会自动复用 MiniMax LLM 配置；中国大陆与国际平台的密钥和地址必须成对匹配 |
| ElevenLabs | `[elevenlabs]` | `model_id` 默认 `eleven_multilingual_v2`，可选 `eleven_flash_v2_5`、`eleven_v3`；WebUI 直接展示收藏音色 |
| Fish Audio | `[fish_audio]` | `model` 默认 `s2.1-pro-free`；免费模型受平台公平使用条款与速率限制约束，商用前核对官方定价页 |
| Chatterbox（自托管） | `[chatterbox]` | 对接 OpenAI 兼容的自托管服务，默认地址 `http://127.0.0.1:4123/v1` |
| Kokoro（自托管） | `[kokoro]` | 默认地址 `http://127.0.0.1:8880/v1`；`voices` 留空时自动从服务端读取音色列表；Kokoro-82M 模型可在 CPU 上运行 |
| ModelBest VoxCPM | `[voxcpm]` | 默认地址 `https://api.modelbest.cn/v1`；`model_id` 必须选择支持 `speech_synthesis` 能力的模型 |

自托管方案（Chatterbox、Kokoro）需要自行部署对应的开源服务；选择服务商本身不会自动安装或认证底层服务。

## 7.4 语速与音量

| 配置项 | 范围与默认值 | 说明 |
| --- | --- | --- |
| `voice_rate` | 有限正数，默认 1.0 | 语速倍率 |
| `voice_volume` | 非负数，默认 1.0 | 旁白音量倍率，0 为静音 |

语速可以直接缓解"脚本偏长"问题：朗读时长与语速成反比，但过高语速会牺牲清晰度，建议增量调整并以字幕预览校验。

## 7.5 上传配音

已备音频时采用 `upload` 模式。约束如下：

1. CLI 支持 `--custom-audio-file`，格式为 MP3、WAV、M4A、AAC、FLAC、OGG 之一；相对路径相对当前工作目录解析。
2. 上传文件路径不会由 WebUI 保存，因此命令行若要复用同一音频，必须显式传参；WebUI 保存的其余配音设置不会自动替换该文件。
3. 上传音频可配合 `subtitle_provider = whisper` 由本地转写实生成字幕（见第 8 章）。
4. 程序会在替换现有文件前校验生成的音频；远程音频下载也带有大小限制与有效性校验。

## 7.6 音色复刻（VoxCPM 参考音频）

音色复刻指用一段参考录音驱动 TTS 输出近似音色。当前由 ModelBest VoxCPM 提供，功能包含：

1. 上传参考音频配置说话人音色；上传文件上限 20 MiB，转换为 WAV 后单个最大 5 MiB。
2. 可选"保留参考音频的节奏与情绪"：默认复用该段音频作为演绎参考，并要求填写其准确逐字稿；也可另选一段节奏与情绪示范音频，通过 `prompt_audio`（示范音频）与 `prompt_text`（示范逐字稿）延续演绎。
3. 逐字稿可由 WebUI 调用本地 Whisper 自动生成可编辑的草稿。

隐私与合规约束如下：参考素材仅保存在当前浏览器会话与当前任务中，不写入配置、预设、任务历史或日志；但参考音频与逐字稿在请求合成时会被发送至 ModelBest 服务端，因此仅允许使用本人或已获授权的录音。

## 7.7 无配音与音乐主导模式

`none` 模式适合纯音乐驱动的画面演示。使用时注意两点：脚本长度仍决定任务时长节奏，通常建议配合较短的 `video_clip_duration` 与更精简的脚本；背景音乐音量可单独调高（第 9 章的 `bgm_volume`），以弱化无旁白造成的"空白感"。

---

# 第 8 章 字幕站：自动字幕、逐词动画与样式工程

字幕样式决定短视频的完成度观感：字体模糊、位置不当或时间轴错位，直接损害观众的观看体验。本章解释两种字幕生成机制的区别、Whisper 本地模型的部署方法、逐词字幕与动画效果，以及字幕样式的全部可调参数。

## 8.1 两种字幕生成机制

字幕由 `subtitle_provider` 控制，取值三种：

| 取值 | 机制 | 特点 |
| --- | --- | --- |
| `edge` | 直接使用语音合成返回的时间戳 | 速度快，无需 GPU；默认值 |
| `whisper` | 本地 `faster-whisper` 转写配音音频 | 时间轴更精确，适配需要逐词对齐的场景；首次使用需下载模型 |
| 留空 | 跳过字幕生成 | 按需关闭字幕 |

`whisper` 模式仅在显式选择时加载模型；Edge 模式的失败不会自动触发 Whisper 模型下载。

## 8.2 Whisper 本地模型配置

`[whisper]` 分区的配置项如下：

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `model_size` | `large-v3` | 约 3 GB；更轻更快可选 `large-v3-turbo`（约 1.6 GB） |
| `device` | `cpu` | 可选 `cuda`（需 NVIDIA GPU） |
| `compute_type` | `int8` | CPU 常用 `int8`；`cuda` 常用 `float16` 或 `int8_float16` |
| `initial_prompt` | 留空 | 可选的转写偏置提示词，用于引导品牌名、专业术语的识别 |

首次使用时程序自动从 Hugging Face 下载模型；本机网络无法自动下载时，可手动下载后放入本地目录：程序的查找规则是存在匹配的 `./models/whisper-{model_size}` 目录时跳过下载。以 `large-v3` 为例，最终目录结构应满足：

```text
MoneyPrinterTurbo
  ├─ models
  │    └─ whisper-large-v3
  │         ├─ config.json
  │         ├─ model.bin
  │         ├─ preprocessor_config.json
  │         ├─ tokenizer.json
  │         └─ vocabulary.json
```

手动下载的原始为 `https://huggingface.co/Systran/faster-whisper-large-v3`。

## 8.3 显示模式与入场动画

v1.3.7 起支持逐词字幕与弹跳动画，取值如下：

| 配置项 | 取值 | 默认 |
| --- | --- | --- |
| `subtitle_display_mode` | `sentence`（按句显示）、`word_by_word`（逐词显示） | `sentence` |
| `subtitle_animation` | `none`（无动画）、`pop_spring`（弹跳出现） | `none` |

约束与判断标准：

1. 逐词时间轴在 Edge TTS 与 Whisper 下最准确；其他配音服务缺少逐词时间戳，可能按短语或整句显示。
2. 弹跳动画渲染时字幕画面与透明蒙版同步缩放，避免出现黑色轮廓。

## 8.4 字幕样式工程

以下样式参数（`config.toml` 的 `[ui]` 段保存，CLI 可用同名参数覆盖）：

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `font_name` | `MicrosoftYaHeiBold.ttc` | 字幕字体文件名，必须位于 `resource/fonts/` 目录 |
| `font_size` | `60` | 字号 |
| `text_fore_color` | `#FFFFFF` | 文字颜色，十六进制 `#RRGGBB` 格式 |
| `stroke_color` | `#000000` | 描边颜色 |
| `stroke_width` | `1.5` | 描边宽度 |
| `subtitle_position` | `bottom` | `top` / `center` / `bottom` / `two_thirds_bottom` / `custom` |
| `custom_position` | `70.0` | 自定义位置的百分比（自画面顶部起算，0 至 100），仅在 `subtitle_position = "custom"` 时生效 |
| `subtitle_background_enabled` | `false` | 字幕背景块开关 |
| `subtitle_background_color` | `#000000` | 背景颜色 |
| `rounded_subtitle_background` | `false` | 背景块圆角 |

两点重要约束：

1. **字体安全边界**：字幕字体必须解析到项目 `resource/fonts` 目录内；指向外部目录的路径与符号链接会被拒绝（v1.3.8 起生效）。若默认字体在部署环境中不可用，应改用该目录内实际存在的字体文件；CLI 在没有可读保存值时的内置回退字体为 `STHeitiMedium.ttc`。
2. **颜色冲突校验**：关闭背景块时不可同时指定背景颜色或圆角，WebUI 与 API 均会在任务创建前拒绝该组合。

## 8.5 时间轴细节与适用判断

faster-whisper 属于本地转写模型，适用于微调场景；实践中建议（建议性质）：

1. 默认先沿用 `edge` 模式，观察字幕错位程度；对边缘、逐词可见效果的短内容切换 `whisper`。
2. 多段旁白、含停顿标签的脚本，`whisper` 的时间轴复原度更好；v1.3.8 至 v1.3.9 期间已改进时间戳舍入、长词换行、Unicode 与停顿标签识别，并避免纯符号行、分隔线干扰时间轴。
3. 关于字幕文件的位置：字幕文件与配音、成片存放在同一任务目录。更换字幕样式需要重新执行生成；预算敏感时，可先用 CLI 的 `--stop-at subtitle` 只产出字幕文件、核对时间轴后再跑完整生成。

---

# 第 9 章 配乐与合成站：背景音乐、画幅与编码

合成阶段把脚本、素材、配音、字幕与配乐合并为最终文件，也是参数最多、对成片风格影响最直接的一站。本章覆盖背景音乐的四种模式、画幅与画面适配、拼接与转场、以及编码器与 FFmpeg 运行参数。

## 9.1 背景音乐的四种模式

背景音乐模式下 `bgm_type`，取值五种：

| 取值 | 来源 | 计费与否 |
| --- | --- | --- |
| `none` | 无背景音乐 | 否 |
| `random`（默认） | 项目内置曲库随机选取 | 否 |
| `custom` | 指定本地音乐文件 | 否 |
| `sonilo` | Sonilo 与视频内容匹配的原创配乐服务 | 是 |
| `elevenlabs` | ElevenLabs AI 音乐接口 | 是，复用其密钥 |

相关说明：

1. 内置曲库位于 `resource/songs/`，可放入自有音乐；v1.3.7 起支持在 WebUI 中选定预设曲目并试听，无需先生成视频。
2. `custom` 模式通过 `custom_bgm_file` 指定文件，也可调用 `POST /api/v1/musics` 上传（MP3、M4A、AAC、WAV、FLAC、OGG、OPUS、WMA，单文件上限 30 MB，存入 `storage/bgm/`）。
3. AI 配乐需要密钥与风格提示词：Sonilo 密钥在 `https://platform.sonilo.com/dashboard` 创建（留空时也可通过环境变量 `SONILO_API_KEY` 传入）；ElevenLabs 音乐走独立型号 `music_v2`（`music_model_id`），提示词长度上限由服务商自身规定，生成前会被校验。
4. 背景音乐音量独立控制：`bgm_volume` 默认 0.2，非负数值；旁白音量按第 7 章 `voice_volume` 分开调节，两条混音线互不覆盖。

## 9.2 音轨混音行为

成片的音轨由旁白与背景音乐两层合成。v1.3.7 至 v1.3.9 期间改进了背景音乐淡出与短片段转场的处理表现；虽然不支持在时间轴上手动调整音乐断点，但更换时长接近的音乐文件或使用 `custom` 模式，可以避免生硬的收尾。

## 9.3 画幅与画面适配

项目支持三种画幅，所有分辨率均为 1080p 等级：

| 画幅 | 分辨率 | 主要平台 |
| --- | --- | --- |
| `9:16` | 1080×1920 | 抖音、快手、视频号、TikTok、YouTube Shorts、Instagram Reels |
| `16:9` | 1920×1080 | B 站、YouTube 长视频等横屏平台 |
| `1:1` | 1080×1080 | 信息流、方形投放 |

画面适配行为（`video_fit_mode`）决定"素材画幅与目标画幅不一致"时的处理：

| 取值 | 行为 | 适用 |
| --- | --- | --- |
| `cover`（默认） | 等比放大并居中裁剪，画面填满 | 观感连贯，但边缘构图信息可能被裁掉 |
| `contain` | 等比缩放至完整放入，两侧或上下补黑边 | 画面必须保留完整构图（图表、字幕截图）时使用 |

画幅选择按素材来源分别保存（配置键形如 `video_aspect_pexels`），因此切换素材源不需要反复调整该设置。另值得注意的是：横屏素材投给竖屏成片时，观感取决于适配模式的选择；部分 AI 素材源还可能返回固定横屏画面（如 `MiniMax-H3` 的当前线路），此时竖屏更依赖 `cover` 与 `contain` 的取舍。

## 9.4 拼接顺序与转场

| 配置项 | 取值与默认 | 说明 |
| --- | --- | --- |
| `video_concat_mode` | `random`（默认）、`sequential` | 多条片段按随机顺序或脚本顺序拼接 |
| `video_transition_mode` | `none`（默认）以及 `shuffle` / `fade-in` / `fade-out` / `slide-in` / `slide-out` / `zoom-in` / `zoom-out` | 片段间转场；CLI 取值用连字符小写形式，配置文件保存 PascalCase 形式如 `FadeIn` |
| `video_clip_duration` | 默认 5 秒 | 单条素材被截取的最长时长 |
| `video_clip_speed` | 0.5 至 2.0，默认 1.0 | 片段播放速度倍率 |
| `image_motion` | `zoom`（默认）、`static` | 图片画面的动态效果（v1.3.9 起可选静止） |

## 9.5 编码器与渲染参数

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `video_codec` | `libx264` | 可选 `h264_nvenc`（NVIDIA）、`h264_amf`（AMD）、`h264_qsv`（Intel 核显）、`h264_mf`（Windows 媒体基础）、`h264_videotoolbox`（Apple 芯片）；环境不支持时自动回退 `libx264` |
| `n_threads` | `2` | FFmpeg 工作线程数 |
| `ffmpeg_concat_timeout_seconds` | `3600` | 片段拼接的最长等待秒数；超时即中止任务并标失败，避免长期停滞 |
| `ffmpeg_tts_timeout_seconds` | `600` | 无配音音频与停顿标签合成的单次调用超时；超时任务直接失败 |

渲染链路自 v1.3.8 起统一使用 BT.709 色彩转换与色彩元数据标记，改善成片在不同播放器中的颜色一致性。编码器按硬件选型即可获得明显提速：NVIDIA 显卡选 `h264_nvenc`、AMD 显卡选 `h264_amf`、Intel 核显选 `h264_qsv`；Apple 芯片设备选 `h264_videotoolbox`。

## 9.6 FFmpeg 的获取与路径

FFmpeg 通常自动下载与检测；检测失败时手动下载安装，并将可执行文件路径写入 `ffmpeg_path`（Windows 示例 `ffmpeg_path = "C:\\path\\to\\ffmpeg.exe"`，注意双反斜杠）。

---

# 第 10 章 任务与产物管理

自动化批量生产会持续产生任务记录、成片文件与中间产物，不加以治理会让磁盘快速耗尽，且找不到历史素材。本章说明任务目录的组织方式、WebUI 的任务管理能力、并发上限，以及产物与缓存的磁盘治理。

## 10.1 任务目录与产物结构

每个任务占用一个独立目录：

```text
storage/tasks/<任务 ID>/
  ├─ final-1.mp4        成片本体（批量生成多条时依次为 final-2.mp4 等）
  ├─ 配音音频文件
  ├─ 字幕文件
  └─ 其他中间产物（文件名以任务内实际文件为准）
```

约束与含义：

1. `final-<编号>.mp4` 是稳定的成片命名规则。
2. 音频、字幕、脚本等中间产物也存放在同一目录，文件名以任务内实际文件为准。
3. pip 安装版中，上述路径位于系统数据目录（涉及 `MPT_HOME`，见第 2 章方式二的数据目录表）。

## 10.2 WebUI 的任务管理能力

- 任务历史记录：成片后可在任务管理页预览、打开目录或删除。
- 自动打开文件夹：`open_task_folder_on_completion` 默认开启，生成完成后自动弹出资源管理器窗口；关闭后仍可手动打开。
- 生成日志：`hide_log = true` 可隐藏 WebUI 中的生成日志显示。
- 批量失败保护：v1.3.9 起，批量生成中部分任务失败时，已完成任务的成片予以保留。

## 10.3 生成设置与 API 密钥的导入、导出与恢复

WebUI 支持生成设置与 API 密钥的导入、导出和恢复。其数据即 `config.toml` 中的服务商分区与 `[ui]` 偏好分区。提醒两层风险：导出文件包含明文密钥，应保存在可信位置；恢复配置会覆盖当前设置，操作前建议备份现有 `config.toml`。

## 10.4 并发与队列上限

API 服务的任务并发与排队由配置控制：

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `max_concurrent_tasks` | `5` | 同时运行的任务数上限 |
| `max_queued_tasks` | `100` | 排队中的任务数上限 |
| `enable_redis` | `false` | 可选的 Redis 任务队列（专门的内存数据库，用于跨进程共享任务状态），开启后 `redis_host / redis_port / redis_db / redis_password` 生效 |

超限的新任务会被拒绝（API 返回 429 状态码，含队列已满的说明）；CLI 批量清单另有"最多 100 个任务、不超过 1 MiB"的独立限制（第 11 章）。Redis 队列可跨进程重启恢复，详细行为在第 12 章说明。

## 10.5 磁盘治理

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `material_directory` | 空串 | 在线素材缓存位置的策略：空串表示缓存至 `./storage/cache_videos`；`task` 表示素材存入各任务目录；绝对路径表示统一存放至指定目录 |

治理建议（建议性质）：

1. 定期清理 `storage/cache_videos` 与过期的 `storage/tasks` 目录；项目自 v1.3.8 至 v1.3.9 已改进缓存清理逻辑（包含中断写入遗留临时文件的回收与误删刚更新缓存的防护），自动维护能力较完善，但仍需定期人工清理。
2. 长期运行的服务器可将 `material_directory` 设为独立磁盘路径，便于容量核算与卸载迁移。
3. 极端磁盘压力下的应急手段：关闭字幕与配乐、缩短 `video_clip_duration`，减小单任务产物体积。

---

# 第 11 章 批量生产：CLI 与批量清单实战

当内容生产从单条创作转为流水线作业（例如每天定时产出十条短视频），WebUI 的手工操作会最先成为瓶颈。CLI 提供了脱离浏览器的全参数入口与批量清单机制，本章覆盖单任务命令、分阶段停止、付费确认、批量清单格式与结果汇总，以及完整的参数速查表。

## 11.1 入口与参数优先级

CLI 有两个入口：pip 安装用户使用 `mpt cli`；源码部署用户使用 `uv run python cli.py`。两者参数一致，本章后续命令统一用 `uv run python cli.py` 表示，`mpt cli` 用户替换前缀即可。完整帮助随时可查：

```bash
uv run python cli.py --help
```

字幕样式与配音参数的取值优先级为：**命令行显式参数 ＞ `config.toml` 中 `[ui]` 保存的 WebUI 设置 ＞ 内置默认值**。其余生成设置（背景音乐、视频数量、段落数量等）不会自动沿用 WebUI 的保存值。若 WebUI 中选择了上传自备音频，命令行需要显式传入 `--custom-audio-file`，因为音频路径不会被保存。

## 11.2 单任务全流程命令

最简完整命令已在第 2 章出现过：

```bash
uv run python cli.py --video-subject "人工智能如何改变日常生活"
```

该命令依赖两个前提：已配置可用的 LLM（用于生成脚本）与 Pexels 密钥（默认素材源）；配音默认走 Edge TTS，无需密钥。命令行为遵循以下规则（为程序行为，非经验建议）：

1. 任务文件写入 `storage/tasks/<任务 ID>/`。
2. 成功时向标准输出打印一个 JSON 对象并返回退出码 0；任务失败返回 1；参数或清单错误返回 2。
3. 运行日志输出到标准错误（stderr），与结果 JSON 不会互相混入。

结合已备脚本与停顿标签的完整示例：

```bash
uv run python cli.py \
  --video-subject "清洁能源的未来" \
  --video-language zh-CN \
  --paragraph-number 2 \
  --video-clip-duration 5 \
  --video-aspect 9:16 \
  --voice-name zh-CN-XiaoxiaoNeural-Female \
  --bgm-type random \
  --stop-at video
```

## 11.3 分阶段停止

`--stop-at` 的取值即第 1 章定义的六个阶段：`script`、`terms`、`audio`、`subtitle`、`materials`、`video`（默认 `video`）。命令在到达指定阶段后立即停止并打印该阶段结果，用途包括：

```bash
# 只生成脚本与搜索词，审稿后再继续
uv run python cli.py --video-subject "阅读如何改变大脑" --stop-at script

# 本地素材链路跳过搜索词阶段（搜索词对本地素材无意义）
uv run python cli.py --video-source local \
  --video-materials "./1.mp4,./2.mp4" \
  --video-subject "民宿改造记录" --stop-at video
```

两条约束：`--stop-at terms` 对 `--video-source local` 无效；`--stop-at subtitle` 不能与 `--no-subtitle-enabled`（关闭字幕）同时使用。

## 11.4 特殊配音与配乐命令

无配音与自备音频的典型命令：

```bash
# 使用备好的旁白音频并依赖 Whisper 生成字幕
uv run python cli.py --video-subject "周末整理房间" \
  --custom-audio-file "./narration.mp3" \
  --stop-at video

# 静音成片：直接停用自动配音
uv run python cli.py --video-script "完整台本内容" \
  --voice-name no-voice --stop-at video
```

AI 配乐的两种模式需要按服务商显式选择，否则提示词会被丢弃：

```bash
# Sonilo 配乐：需先在 config.toml 配置 sonilo_api_key
uv run python cli.py --video-subject "深海里的微光" \
  --bgm-type sonilo \
  --video-music-prompt "低沉、空灵的电子氛围"
```

## 11.5 付费来源的确认参数

以下来源触发素材生成前必须显式计费确认（见第 6 章 6.1 节），否则命令以参数错误退出：

| 命令 | 说明 |
| --- | --- |
| `--video-source wavespeed --confirm-wavespeed-charge` | WaveSpeed |
| `--video-source volcengine_seedance --confirm-seedance-charge` | 火山引擎 Seedance |
| `--video-source ofox --confirm-ofox-charge` | OFox |
| `--video-source metaso_minimax --confirm-metaso-minimax-charge` | 秘塔 MiniMax H3 |
| `--video-source muapi --confirm-muapi-charge` | MuAPI |
| `--video-source infistar --confirm-infistar-charge` | 无限星河 |
| `--video-source jalapeno --confirm-jalapeno-charge` | Jalapeno Cloud |

计费确认仅在素材实际生成阶段（`--stop-at` 为 `materials` 或 `video`）被强制；停在脚本、搜索词、配音、字幕等纯文本或音频阶段时不触发素材生成，也不需要确认。CLI 公开 13 种素材源（含 `openai_image` 与 `codex_image`）；`loomloom` 仅在 WebUI 中可用。

## 11.6 批量清单机制

`--batch-file` 接收 UTF-8 编码的 JSON 数组或 JSONL（JSON Lines，即每行一个独立 JSON 对象的文本格式）文件，功能规则如下：

1. **上限**：最多 100 个任务，文件不超过 1 MiB。
2. **字段**：每个对象是任务参数，可覆盖 `VideoParams` 字段（即一次视频生成任务的全部参数字段，名称与第 11.8 节速查表一致）；未知字段在预检时被拒绝。每个合并后的任务必须具备 `video_subject` 或 `video_script` 之一。
3. **路径解析**：清单文件路径相对当前工作目录；清单内的 `custom_audio_file`（自备配音）与本地 `video_materials[].url` 的相对路径以清单所在目录为基准；命令行选项中的相对路径仍以当前工作目录为基准。
4. **预检**：所有任务在第一个任务启动前完成参数与本地文件预检；任何条目有误时整批退出码为 2，不会启动任何任务。
5. **容错**：单个任务运行失败不阻塞后续条目；结束后输出统一的 JSON 汇总（字段包括 `total`、`succeeded`、`failed`、`tasks`，每个任务条目含 `index`、`task_id`、`status`、`result`、`failed_stage`、`error`）。
6. **全局参数**：`--stop-at` 对整批生效；CLI 其他选项作为各任务的默认值，被清单对象逐字段覆盖。

批量清单示例（`tasks.jsonl`）：

```json
{"video_subject": "为什么海洋保护很重要", "video_aspect": "16:9"}
{"video_subject": "多巴胺穿搭的心理学", "video_aspect": "9:16", "paragraph_number": 2}
{"video_script": "直接提供的完整台本……", "video_source": "local", "video_materials": [{"provider": "local", "url": "./shots/clip1.mp4", "duration": 0}]}
```

执行：

```bash
uv run python cli.py --batch-file ./tasks.jsonl --stop-at video
```

## 11.7 批量结果与退出码约定

| 退出码 | 含义 |
| --- | --- |
| `0` | 单任务成功；批量模式全部任务的整体成功（汇总 JSON 仍会区分单独失败） |
| `1` | 单任务或批量中存在任务级失败 |
| `2` | 参数或清单校验错误（未启动任何任务即退出） |

定时任务（cron 或 Windows 计划任务）按退出码做成功告警即可；标准错误中的运行日志用于排查失败阶段（`failed_stage` 指向第 1 章定义的六个阶段之一）。

## 11.8 CLI 参数速查表

以下按 argparse 分组列出全部已核实参数（默认值以 v1.3.9 源码为准）。

**脚本与内容组**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--video-subject` | 空 | 主题；与 `--video-script` 至少提供一个 |
| `--video-script` | 空 | 完整脚本，提供后跳过脚本生成 |
| `--video-terms` | 自动生成 | 英文逗号分隔的素材搜索词 |
| `--video-language` | 自动检测 | 脚本语言代码，如 `zh-CN`、`en-US` |
| `--paragraph-number` | 1 | 段落数，1 至 10 |
| `--video-script-prompt` | 无 | 脚本生成的附加要求 |
| `--custom-system-prompt` | 无 | 替换默认系统提示词 |

**素材与流程组**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--video-source` | `pexels` | 13 种素材源见第 5 章 5.1 节 |
| `--video-materials` | 无 | 逗号分隔的本地图片/视频路径，仅配 `local` 使用 |
| `--stop-at` | `video` | 六阶段之一 |
| `--confirm-*-charge` | 无 | 七种付费来源的计费确认（见 11.5 节） |

**视频输出组**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--video-count` | `1` | 生成数量，至少 1 |
| `--video-aspect` | `9:16` | `9:16` / `16:9` / `1:1` |
| `--video-fit-mode` | `cover` | `cover` 填满裁剪；`contain` 保留完整画面补黑边 |
| `--video-concat-mode` | `random` | `random` / `sequential` |
| `--image-motion` | `zoom` | `static` / `zoom` |
| `--video-transition-mode` | `none` | `shuffle`、`fade-in`、`fade-out`、`slide-in`、`slide-out`、`zoom-in`、`zoom-out` |
| `--video-clip-duration` | `5` | 单条素材最长时长（秒），至少 1 |
| `--video-clip-speed` | `1.0` | 0.5 至 2.0 |
| `--match-materials-to-script` | 关闭 | 布尔开关，`--no-match-materials-to-script` 显式关闭 |
| `--n-threads` | `2` | FFmpeg 工作线程数 |

**配音与配乐组**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--voice-name` | `[ui]?.voice_name`，否则 `zh-CN-XiaoxiaoNeural-Female` | `no-voice` 表示无配音；第三方音色带前缀，如 `gemini:`、`mimo:`、`elevenlabs:`、`chatterbox:`、`kokoro:`、`voxcpm:` |
| `--voice-volume` | `[ui]?.voice_volume`，否则 1.0 | 非负倍数 |
| `--voice-rate` | `[ui]?.voice_rate`，否则 1.0 | 正数倍数 |
| `--custom-audio-file` | 无 | MP3/WAV/M4A/AAC/FLAC/OGG；跳过 TTS；配 `whisper` 生成字幕 |
| `--bgm-type` | `random` | `none`、`random`、`custom`、`sonilo`、`elevenlabs` |
| `--bgm-file` | 无 | 位于 `storage/bgm` 或 `resource/songs` 内的文件；隐含 `custom` |
| `--bgm-volume` | `0.2` | 非负倍数 |
| `--sonilo-bgm-prompt` | 无 | 仅配 `sonilo`，最长 2000 字符 |
| `--video-music-prompt` | 无 | AI 配乐风格提示词；需 `--bgm-type sonilo` 或 `elevenlabs` |

**字幕组**

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--subtitle-enabled` / `--no-subtitle-enabled` | `[ui]?.subtitle_enabled`，默认开启 | 布尔开关 |
| `--font-name` | `[ui]?.font_name`，否则 `STHeitiMedium.ttc` | 字体文件名，限于 `resource/fonts` |
| `--font-size` | `[ui]?.font_size`，否则 `60` | 字号，至少 1 |
| `--subtitle-position` | `[ui]?.subtitle_position`，否则 `bottom` | 五种位置之一 |
| `--custom-position` | `[ui]?.custom_position`，否则 `70` | 0 至 100，需配 `custom` |
| `--subtitle-display-mode` | `[ui]?.subtitle_display_mode`，否则 `sentence` | `sentence` / `word_by_word` |
| `--subtitle-animation` | `[ui]?.subtitle_animation`，否则 `none` | `none` / `pop_spring` |
| `--text-fore-color` | `[ui]?.text_fore_color`，否则 `#FFFFFF` | `#RRGGBB`；部分 shell 中需引号防止 `#` 被解析为注释 |
| `--stroke-color` / `--stroke-width` | `#000000` / `1.5` | 描边颜色与宽度 |
| `--subtitle-background-enabled` / `--subtitle-background-color` / `--rounded-subtitle-background` | 各按 `[ui]?` 保存值 | 背景块开关、颜色与圆角 |

**执行组**

| 参数 | 说明 |
| --- | --- |
| `--task-id` | 自定义任务 ID（UUID 格式），与 `--batch-file` 互斥 |
| `--batch-file` | UTF-8 JSON 数组或 JSONL 清单，见 11.6 |

表中 `[ui]` 表示 `config.toml` 的 `[ui]` 段，`[ui]?` 表示按该键是否存在可用保存值决定。

---

# 第 12 章 API 集成与自动化

当生成能力需要挂入更大的系统——定时任务平台、自建网页端、内部工具链——CLI 的进程模型就不够用了。API 服务以 HTTP 常驻进程方式提供任务创建、队列与进度查询、产物下载，本章覆盖端点、鉴权、浏览器跨域、任务队列与集成示例。

## 12.1 启动服务与发现端点

启动命令在第 2 章出现过：

```bash
uv run python main.py
```

服务监听地址与端口由 `config.toml` 顶部控制：`listen_host`（默认示例值为 `0.0.0.0`，仅本机访问可改为 `127.0.0.1`）与 `listen_port`（默认 `8080`）。交互式接口文档提供两套界面：

- Swagger UI：`http://127.0.0.1:8080/docs`
- ReDoc：`http://127.0.0.1:8080/redoc`

接口文档页列出全部端点与请求/响应模型字段，可直接试调用；接口标题与版本号随进程加载，启动日志会提示文档地址。

## 12.2 端点总览

路由统一前缀为 `/api/v1`（由项目路由定义固定），任务产物另以独立静态路径 `/tasks/` 暴露：

| 方法与路径 | 用途 | 状态码约定 |
| --- | --- | --- |
| `POST /api/v1/videos` | 创建短视频生成任务（一次走到成片） | 成功 200；队列满 429；参数错 400 |
| `POST /api/v1/subtitle` | 仅生成字幕 | 额外约束见 12.3 |
| `POST /api/v1/audio` | 仅生成配音音频 | 无 |
| `GET /api/v1/tasks` | 分页查询全部任务（`page` 默认 1；`page_size` 默认 10，上限 1000） | 无 |
| `GET /api/v1/tasks/{task_id}` | 查询单个任务状态与产物链接 | 404 表示任务不存在 |
| `DELETE /api/v1/tasks/{task_id}` | 删除任务及其产物目录 | 任务仍在运行返回 409 |
| `GET /api/v1/musics` | 列出本地背景音乐 | 无 |
| `POST /api/v1/musics` | 上传背景音乐（8 种格式、30 MB 上限） | 文件无效 400；校验服务不可用 500 |
| `GET /api/v1/video_materials` | 列出已上传的本地素材 | 无 |
| `POST /api/v1/video_materials` | 上传本地素材文件 | 文件名非法或无效 400 |
| `GET /api/v1/stream/{file_path}` | 视频流式播放（支持 Range 分段） | 416 范围异常 |
| `GET /api/v1/download/{file_path}` | 下载任务产物文件 | 404 文件不存在 |

`GET` 与 `HEAD` 同时支持：`HEAD` 请求只返回元数据（如 `Content-Length`、是否支持 `Accept-Ranges`），便于客户端预检而不下载内容。

## 12.3 创建任务与轮询结果

最小化请求示例（body 为任务参数 JSON，字段即第 11.8 节表格中的名称）：

```bash
curl -s -X POST http://127.0.0.1:8080/api/v1/videos \
  -H "Content-Type: application/json" \
  -d '{"video_subject":"人工智能如何改变日常生活","video_aspect":"9:16"}'
```

响应为任务对象（节选示意，完整字段以 `/docs` 为准）：

```json
{
  "task_id": "9f0c3ba2-…",
  "params": { "video_subject": "人工智能如何改变日常生活" }
}
```

补充两点运行约束：

1. 创建 `videos` 任务时若开启字幕，接口会预先校验字幕颜色与字体路径（字体越界返回 403 或 404），避免在生成成片这一付费链路后才失败。
2. `subtitle` 端点要求该任务开启字幕；`audio` 端点仅生成配音音频；两个上传端点对文件格式与大小实施校验（见 12.2 表格的状态码约定）。

轮询单任务状态：

```bash
curl -s http://127.0.0.1:8080/api/v1/tasks/9f0c3ba2-9acd-4a53-8632-1234567890ab
```

任务状态字典包含 `state` 之类的运行状态字段与产物链接；链接形态如 `/tasks/<任务 ID>/final-1.mp4`（配置了 `endpoint` 时生成完整的外部地址）。下载产物：

```bash
curl -s -o final-1.mp4 \
  http://127.0.0.1:8080/api/v1/download/9f0c3ba2-9acd-4a53-8632-1234567890ab/final-1.mp4
```

## 12.4 鉴权

`config.toml` 中 `[app]` 段的 `api_key` 控制接口保护：

1. 留空（默认）时保持本地免认证行为，程序启动日志会提示"API key authentication is disabled"。
2. 设置后，`/api/v1` 全部路由与 `/tasks` 产物路径均要求请求头 `x-api-key` 与配置值一致；产物下载同样携带该请求头。
3. 由于浏览器地址栏无法附加自定义请求头，启用鉴权后应通过 API 客户端（如 `curl -H "x-api-key: …"`）下载产物。

密钥只应保存在 `config.toml`，不要写进代码或提交到版本库。

## 12.5 浏览器跨域（CORS）行为

CORS 是浏览器同源策略下的跨域机制，只约束浏览器中的 JavaScript 调用，不影响 curl、Postman、n8n 或服务端 SDK。项目行为：

1. 默认仅允许同源网页访问：跨域白名单为空。
2. 部署了独立网页前端时，通过环境变量 `CORS_ALLOWED_ORIGINS` 显式列出可信来源（逗号分隔，可含默认端口折叠后的形式）；可填 `*` 表示全部来源，但免鉴权状态下程序会从日志给出安全告警。
3. 服务端另有来源防护：无法在白名单匹配的浏览器 Origin 请求会被 403 拒绝，这一中间件覆盖无需预检的简单请求。

## 12.6 Redis 任务队列

内存队列（默认）任务状态保存在进程内，进程重启后任务记录消失。开启 `enable_redis` 后改用 Redis（一种进程外内存数据库）承载任务状态与队列，行为包括：

1. 队列条目可跨 API 进程重启存续；新进程启动时会自动恢复排队任务的调度。
2. 相关配置为 `redis_host`（默认 `localhost`）、`redis_port`（默认 6379）、`redis_db`（默认 0）、`redis_password`（默认空）。
3. 安全提示：Redis 实例仅供本项目使用并置于可信网络；不应允许不可信写入者修改序列化的任务记录。

## 12.7 视频播放的 Range 支持

`GET /api/v1/stream/{file_path}` 支持 HTTP Range（分段请求）：响应带 `Accept-Ranges: bytes`，范围命中时返回 206 与 `Content-Range`；浏览器与播放器借此实现拖动进度与预加载。注意两点：多段 Range 请求会被拒绝（返回 416）；`HEAD` 请求只返回文件大小与字节范围支持标记，不传输内容。

## 12.8 集成方案参考

以下为常见接入形态（建议性质）：

1. **n8n / 自动化工作流**：以 HTTP Request 节点调用 `POST /api/v1/videos`，Wait 节点轮询 `GET /api/v1/tasks/{id}`，成功后调用 `download` 接口落盘。
2. **自建网页端**：前端直接调用 API 时必须先配置 `CORS_ALLOWED_ORIGINS`；若启用 `api_key`，则前端需要自带请求头代理层。
3. **反向代理下载**：配置 `[app]?.endpoint`（外部下载基址）后，任务链接会生成完整的反向代理地址，适合产物域名与 API 域名分离的部署。

---

# 第 13 章 部署专题与跨平台发布

前几章的部署都是"一台机器、一个进程"的默认形态，而服务器发布产物为多条视频、一个账号或一家的自动化运营提出更强的隔离与远程访问诉求。本章讨论局域网与远程访问、Docker 进阶部署形态、数据迁移，以及 TikTok、Instagram、YouTube Shorts 的一键发布。

## 13.1 局域网与远程访问

三类服务的开放方式：

| 服务 | 局域网开放方法 | 默认状态 |
| --- | --- | --- |
| WebUI | 启动前设置环境变量 `MPT_WEBUI_HOST=0.0.0.0` | 本机访问 |
| API | `listen_host = "0.0.0.0"`（示例配置默认值已是全接口监听） | 视部署环境配置 |
| Docker | 容器端口映射至宿主机（WebUI `8501`、API `8080`） | 由 compose 文件定义 |

对公网暴露时的安全建议：为 API 配置 `api_key` 鉴权；WebUI 服务宜置于内网或 VPN 后；如有必要使用反向代理并限制来源。

## 13.2 Docker 进阶形态

| 部署形态 | 文件 | 适用 |
| --- | --- | --- |
| 预构建镜像（推荐） | `docker-compose.release.yml` | 直接拉取 `ghcr.io/harry0703/moneyprinterturbo:latest` |
| 本地构建 | `docker-compose.yml` 或默认 `docker compose up` | 自定义代码改动 |
| GPU 场景 | `docker-compose.gpu.yml` + `Dockerfile.gpu` | 本地 GPU 处理需求（如 Whisper 转写、本地 SDXL 图像服务） |
| Claude Code 订阅 | `docker-compose.claude.yml` + `Dockerfile.claude` | 以 Claude 订阅作为 LLM 服务商 |

镜像升级即拉取新版本镜像后重新 `docker compose up`；`config.toml` 与挂载目录持久保留。挂载细节（目录映射、卷命名）以 compose 文件注释为准。

## 13.3 数据迁移与 MPT_HOME

pip 安装版的所有数据位于第 2 章方式二数据目录表所列的系统数据目录。迁移或自定义路径的做法：在启动前设置环境变量 `MPT_HOME` 指向目标目录，程序在该目录下维护 `config.toml`、`storage/` 等子目录。源码部署下的数据就在项目根目录内，整个目录拷贝即可完成迁移。

## 13.4 跨平台发布：Upload-Post

成片完成后可按配置自动发布到 TikTok、Instagram、YouTube Shorts。该功能通过 **Upload-Post**（`https://upload-post.com/`，订阅制第三方服务）实现。

`config.toml` 的 `[app]` 段相关配置：

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `upload_post_enabled` | `false` | 功能总开关 |
| `upload_post_api_key` | 空 | 在 Upload-Post 创建 |
| `upload_post_username` | 空 | 绑定的账号标识 |
| `upload_post_platforms` | `["tiktok", "instagram"]` | 发布目标平台列表，可含 `youtube` |
| `upload_post_auto_upload` | `false` | 生成成功后自动上传 |
| `upload_post_youtube_privacy_status` | `public` | YouTube 隐私级别：`public` / `unlisted` / `private` |
| `upload_post_youtube_made_for_kids` | `false` | YouTube 受众声明；必须显式书写 TOML 布尔值。未配置时后续 YouTube 上传默认声明为非面向儿童；儿童内容必须显式设为 `true` |
| `upload_post_max_pending_tasks` | `10` | 单进程内运行与排队的发布任务数上限 |

值得注意的行为边界（程序行为）：视频生成结果与发布状态分开记录；达到发布任务上限时，视频生成本身继续成功，仅跨平台发布被拒绝；发布作业在当前进程内执行，进程重启后未完成的排队作业不会恢复；新设置的 `made_for_kids` 不会修改此前已上传的视频。

操作步骤：

1. 注册 Upload-Post 账号并绑定目标平台的授权。
2. 在 `config.toml` 配置上表对应字段；`upload_post_enabled = true` 后按需打开 `upload_post_auto_upload`。
3. 上传前确认各平台对 AI 内容标识、音乐版权与商标的政策义务；YouTube 的受众声明涉及合规责任，按视频真实受众设置。

---

# 第 14 章 故障排查、性能与成本优化

服务长期运行后，素材限流、密钥失效与服务成本上升会陆续出现，此时需要的不再是逐章的学习，而是一套定位问题的方法与调优手段。本章给出诊断日志的开启方法、按错误信息定位的处理方案、性能与成本的优化路径，并附上全项目配置速查表与术语表，供后续各章随时回查。

## 14.1 诊断日志

日志等级由 `[app]` 段的 `log_level` 控制，取值 `DEBUG`、`INFO`、`WARNING`、`ERROR`，默认 `DEBUG`。`[diagnostics]` 段提供两个开关：

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `log_level` | 未设置 | 可选覆盖，同时作用于 API、CLI 与 WebUI 三个进程 |
| `log_requests` | `false` | 请求追踪：按需记录经脱敏、限长的请求参数与服务端响应状态、耗时；不记录响应体。提示词内容可能出现在日志中，建议仅在排查期间开启 |

v1.3.9 起，WebUI 基础设置中有界面化的诊断日志设置（设置区的接口项）：在该进程内调整日志等级立即生效；其他进程或手工修改配置文件后仍需重启才生效。

## 14.2 常见错误对照表

| 错误或现象 | 原因定位 | 处理方案 |
| --- | --- | --- |
| `RuntimeError: No ffmpeg exe could be found` | FFmpeg 自动下载或检测失败 | 从 `https://www.gyan.dev/ffmpeg/builds/` 手动下载，并将可执行文件路径写入 `ffmpeg_path` |
| `OSError: [Errno 24] Too many open files` | 并发处理大量素材导致文件句柄耗尽 | 降低 `material_concurrency`、`video_clip_concurrency`，或在系统层面提高文件句柄上限 |
| Whisper 模型下载失败 | 本机网络无法访问 Hugging Face | 改用更小的 `large-v3-turbo`，或手动下载模型放入 `models/whisper-{model_size}`（第 8 章） |
| Edge TTS 请求返回 403 | 早期版本依赖问题 | 升级至 v1.3.9（已更新 Edge TTS 依赖修复该问题） |
| 大模型连接失败 | 密钥失效、填写错误或服务商地址不匹配 | 核对服务商选择、密钥与接入地址；v1.3.9 已改进模型连接错误的提示内容 |
| API 返回 429 | `max_concurrent_tasks` 与 `max_queued_tasks` 队列占满 | 等待任务释放或调整上限；批量场景考虑开启 Redis |
| 浏览器调用 API 返回 403 | 跨域来源不在白名单 | 配置 `CORS_ALLOWED_ORIGINS`（第 12 章） |
| 字幕字体相关 403/404 | 字体名越界或不存在 | `font_name` 改为 `resource/fonts/` 内实际存在的字体文件 |
| Pexels / Pixabay 素材下载失败 | 速率限制或网络问题 | 增加密钥（列表轮换）、降低 `material_concurrency`，或配置 14.3 节的素材请求代理 |

该表之外的情况，请先按 14.1 提升日志等级复现任务，再按第 1 章的阶段划分逐段定位。

## 14.3 素材请求代理

`[proxy]` 段用于给素材下载链路配置代理，对 Pexels、Pixabay、Coverr 与素材下载生效；格式为标准的 HTTP 带凭据形式：

```toml
[proxy]
# http = "http://<用户名>:<密码>@<代理地址>:<端口>"
# https = "http://127.0.0.1:1080"
```

仅在需要经代理访问上述素材平台时启用，格式错误或代理不可达时应回退直连。

## 14.4 性能优化

| 手段 | 配置项 | 说明 |
| --- | --- | --- |
| 素材下载并发 | `material_concurrency` | 默认 1（串行）；取值 1、2、4、6、8，按 8 封顶。调大加速明显但更易触发速率限制 |
| 片段渲染并发 | `video_clip_concurrency` | 默认 1；取值同上。调大占用更多 CPU 与内存 |
| 硬件编码 | `video_codec` | 按平台选 NVENC、AMF、QSV、媒体基础或 VideoToolbox（第 9 章） |
| 渲染线程 | `n_threads` | 默认 2，可按核心数适度调大 |
| 转写加速 | `[whisper].device` | `cuda` 配合 `float16` 计算类型显著快于 CPU |
| 任务隔离 | `enable_redis` | 多进程共享任务状态与队列，v1.3.8 起改善了队列恢复与排队停滞问题 |

调优次序（建议性质）：先确认单任务的成片质量，再放量并发；并发调大后观察速率限制报错与内存占用，回落到稳定值。

## 14.5 成本结构优化

成本主要发生在四个环节：脚本调用的 LLM Token、AI 视频素材的每段计费、云端 TTS 与 AI 配乐、发布订阅。免费组合的可行形态是"Edge TTS + 免费素材库 + 本地字幕"，仅 LLM 一项按量付费。优化要点：

1. LLM 调用发生在脚本与搜索词两个点，量级稳定；`paragraph_number` 增大略增 Token 消耗。
2. AI 视频素材在成本中占比最高（按段计费）；先在 480p 或低分辨率档试跑，确认可用后再上高分辨率，`volcengine_seedance_resolution`、`ofox_resolution`、`muapi_resolution` 均可调。
3. 云端 TTS 与 AI 配乐按需启用：Edge TTS 免费档足以覆盖多数常规配音需求，成本控制的关键是把付费 TTS 留给少数高价值栏目。
4. 素材密钥池与服务商分散组合，可提高配额上限并降低单一厂商故障的影响。

## 14.6 升级与变更跟踪

项目更新较快，发布说明列明每版本的重点与升级注意。跟踪与升级方式随安装方式而异：

| 安装方式 | 升级命令 |
| --- | --- |
| Windows 一键启动包 | 双击 `update.bat` |
| pip 官方包 | `python -m pip install --upgrade mpt` |
| 源码部署 | `git pull` 后执行 `uv sync --frozen` |
| Docker | 拉取新镜像后重新 `docker compose up` |

升级前建议备份 `config.toml`；读取发布说明中的"使用与升级说明"，例如 v1.3.8 起字体路径收紧、并发默认值变化等行为差异均有记录。

---

# 附录 A 配置项速查表

以 v1.3.9 的 `config.example.toml` 为准。未列出的网关类服务商（Infistar、Jalapeno、Fluxion、Cheaper Inference、Requesty、FutureInfra、Y-API、Opper、onomeo、EmpirioLabs、Runcrate、CloudGPU 等）遵循统一命名 `*_api_key / *_base_url / *_model_name`，密钥地址见文件内注释。

**顶层与 `[diagnostics]` 分区**

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `log_level` | `DEBUG` | 日志等级，取值 DEBUG、INFO、WARNING、ERROR |
| `listen_host` | `0.0.0.0` | API 监听地址；仅本机访问改为 `127.0.0.1` |
| `listen_port` | `8080` | API 监听端口 |
| `[diagnostics].log_level` | 未设置 | 三进程共享的可选覆盖 |
| `[diagnostics].log_requests` | `false` | 请求追踪开关（脱敏且限长，不记录响应体） |

**`[app]` 通用与 API**

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `api_key` | 空 | API 鉴权密钥；配置后 `/api/v1` 与 `/tasks` 需 `x-api-key` 请求头 |
| `hide_config` | `false` | 隐藏 WebUI 基础配置面板 |
| `tls_verify` | `true` | 校验外部 API 与素材下载的 TLS 证书，仅在可信代理或自签名环境临时关闭 |
| `edge_tts_timeout` | `30` | 单次 Edge TTS 流式请求超时，0 为禁用 |
| `endpoint` | 空 | 生成产物下载链接的外部基址（反向代理场景） |
| `max_concurrent_tasks` / `max_queued_tasks` | `5` / `100` | API 并发与排队上限 |
| `enable_redis` 及 `redis_*` | `false` / 见原文注释 | Redis 任务队列与连接参数 |

**素材与语音字幕**

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `video_source` | `pexels` | 素材源取值见第 5 章 5.1 节 |
| `pexels_api_keys` / `pixabay_api_keys` / `coverr_api_keys` | `[]` | 免费素材密钥，支持逗号分隔列表轮换 |
| `match_materials_to_script` | `false` | 按文案顺序选择素材 |
| `material_concurrency` / `video_clip_concurrency` | `1` / `1` | 并发上限 8，取值 1、2、4、6、8 |
| `video_codec` | `libx264` | 五种硬件编码可选（第 9 章），不支持时回退 |
| `ffmpeg_path` | 空 | FFmpeg 可执行文件路径（检测失败时设置） |
| `ffmpeg_concat_timeout_seconds` | `3600` | 拼接停滞超时 |
| `ffmpeg_tts_timeout_seconds` | `600` | 单次音频转换超时 |
| `material_directory` | 空 | 空串缓存至 `storage/cache_videos`；`task` 存任务目录；绝对路径统一存放 |
| `subtitle_provider` | `edge` | `edge` / `whisper` / 留空不生成字幕 |
| `[whisper].model_size` | `large-v3` | 兼容 `large-v3-turbo`（约 1.6 GB） |
| `[whisper].device` | `cpu` | `cpu` / `cuda` |
| `[whisper].compute_type` | `int8` | CPU 常用 `int8`；CUDA 常用 `float16`、`int8_float16` |
| `[whisper].initial_prompt` | 空 | 转写偏置提示词 |

**发布**

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `upload_post_enabled` / `upload_post_api_key` / `upload_post_username` | `false` / 空 / 空 | Upload-Post 功能开关与账号凭据 |
| `upload_post_platforms` | `["tiktok", "instagram"]` | 目标平台列表，可含 `youtube` |
| `upload_post_auto_upload` | `false` | 生成成功后自动上传 |
| `upload_post_youtube_privacy_status` | `public` | `public` / `unlisted` / `private` |
| `upload_post_youtube_made_for_kids` | `false` | YouTube 受众声明，须书写 TOML 布尔值 |
| `upload_post_max_pending_tasks` | `10` | 单进程发布任务上限 |

**`[ui]` WebUI 偏好（CLI 无显式参数时按表回退）**

| 配置项 | 默认 | 说明 |
| --- | --- | --- |
| `language` | `zh` | WebUI 界面语言 |
| `video_concat_mode` / `video_transition_mode` | `random` / `None` | 拼接顺序与转场 |
| `video_fit_mode` | `cover` | `cover` / `contain` |
| `video_clip_duration` / `video_clip_speed` | `3` / `1.0` | 片段最长时长（配置文件缺省为 3；CLI 单任务默认为 5）与播放倍率 |
| `video_count` | `1` | 批条数 |
| `tts_server` | `azure-tts-v1` | 配音服务商（Azure TTS V1 即 Edge TTS） |
| `voice_mode` / `voice_name` | `tts` / 空 | 配音模式与音色 |
| `voice_volume` / `voice_rate` | `1.0` / `1.0` | 音量与语速倍率 |
| `bgm_type` / `bgm_volume` | `random` / `0.2` | 背景音乐模式与音量 |
| `subtitle_enabled` | `true` | 字幕开关 |
| `subtitle_display_mode` / `subtitle_animation` | `sentence` / `none` | 按句或逐词；动画 none / pop_spring |
| `font_name` / `font_size` | `MicrosoftYaHeiBold.ttc` / `60` | 字体与字号 |
| `text_fore_color` / `stroke_color` / `stroke_width` | `#FFFFFF` / `#000000` / `1.5` | 文字、描边颜色与描边宽度 |
| `subtitle_position` / `custom_position` | `bottom` / `70.0` | 字幕位置与自定义百分比 |
| `subtitle_background_enabled` / `subtitle_background_color` / `rounded_subtitle_background` | `false` / `#000000` / `false` | 背景块 |
| `open_task_folder_on_completion` | `true` | 生成后自动打开任务目录 |
| `hide_log` | `false` | 隐藏生成日志 |

---

# 附录 B 术语表

| 术语 | 定义 |
| --- | --- |
| AI Agent | 能读取技能文档并操作本地终端，代为完成安装、配置与生成的智能代理程序 |
| API | 以 HTTP 协议向外部程序提供任务创建、查询与下载能力的接口 |
| API Key | 服务商颁发、用于识别调用者身份与计费的字符串凭证 |
| BGM | 背景音乐（Background Music） |
| CLI | 命令行接口，即 `mpt cli` 或 `cli.py` |
| config.toml | 全项目唯一的 TOML 格式核心配置文件，存放 API 密钥与服务商设置 |
| cover / contain | 画面适配模式：裁剪填满或保留完整画面补黑边 |
| CORS | 浏览器同源策略下的跨域机制，本项目用 `CORS_ALLOWED_ORIGINS` 配置白名单 |
| Docker | 容器化运行环境；镜像由 `ghcr.io/harry0703/moneyprinterturbo` 发布 |
| FFmpeg | 开源音视频处理工具组，用于配音合成、片段处理与最终编码 |
| JSONL | JSON Lines 文本格式：每行存放一个独立 JSON 对象 |
| LLM | 大语言模型（Large Language Model），用于脚本与搜索词生成 |
| MPT_HOME | pip 安装版的数据目录环境变量，更改配置与产物的存放位置 |
| pip | Python 官方包管理器 |
| Redis | 进程外内存数据库，本项目可选的任务队列实现 |
| Streamlit | Python 的 Web 应用框架，WebUI 基于其构建 |
| uv | Astral 出品的 Python 环境与依赖管理工具 |
| venv | Python 标准库的虚拟环境，隔离项目依赖 |
| WebUI | 基于浏览器的图形操作界面 |
| Whisper | 本地语音转写模型，通过 faster-whisper 实现字幕生成 |
| x-api-key | 启用 API 鉴权后，客户端必须携带的请求头名称 |
| 画幅 | 成片宽高比：`9:16`、`16:9`、`1:1` |
| 素材搜索词 | 由脚本提炼的检索关键词，用于在素材源中匹配画面 |
| 音色 | 具名的声音角色，由服务商按性别、口音、情感区分 |
| 音色复刻 | 用参考音频驱动 TTS 输出近似音色的功能（当前由 VoxCPM 提供） |
| 停顿标签 | 脚本内插入静音的标记，仅 Edge TTS 支持，单次 0.1 至 10 秒 |
| 逐词字幕 | 按词语逐个出现的字幕显示模式 |
| 转场 | 相邻视频片段之间的过渡效果 |
| 显存 | GPU 的独立显存容量，影响本地转写与本地图像生成的配置规模 |
| 批量清单 | CLI 的多任务输入文件（JSON 数组或 JSONL），单批最多 100 个任务 |

---

## 结语与版本说明

本教程按流水线顺序覆盖了安装、脚本、素材、配音、字幕、配乐合成、任务管理、批量、API、部署与发布的完整链路。使用建议如下：先按第 2 章与第 3 章完成一条完整成片，再以第 4 章至第 9 章逐站深化，最后通过第 11 章与第 12 章把流程固化为批量与自动化作业。

再次提示：本文以 v1.3.9 与 2026 年 10 月 30 日核实的信息为基准。项目更新频繁，第三方生态（各服务商的密钥地址、付费模型与额度政策）变化也很快；涉及计费与隐私的关键操作，请始终以官方 README、`config.example.toml` 注释与各服务商平台的当时文档为准。




