# FinExtract Windows 便携版 v1.0 — 构建与验收报告

生成时间：2026-10-15
构建机：Windows（Python 3.14 · PyInstaller 6.22.3）

## 一、交付物

| 文件 | 说明 |
|---|---|
| `FinExtract_Windows_Portable_v1.0.zip` | 最终发行包（解压即用） |
| `FinExtract.exe` + `_internal/` | 打包后的网站服务（onedir） |
| `启动 FinExtract.bat` / `关闭 FinExtract.bat` | 一键启动 / 停止 |
| `使用手册.html` | 面向最终用户的中文操作手册 |
| `SHA256.txt` | 关键文件校验值 |

## 二、构建过程要点

1. 源码基线：FinExtract V30 独立源码包（含 V29 全部前端修复，与开发工程逐文件哈希比对仅路径适配差异）。
2. 原工程 requirements.txt 中 `openai==3.10.0`、`httpcore2`、`httpx2`、`idna==3.19` 为 PyPI 不存在的版本号，无法用于构建；改用开发工程已通过 194 项回归测试的实装版本冻结清单。
3. `--collect-all onnxruntime` 会连带拉入 `onnxruntime.tools/quantization`（依赖 torch，约 360MB）；已改为 `--collect-binaries/--collect-data` 并显式排除 torch/matplotlib/pandas/scipy，最终包体实测 324MB、1056 个文件（OCR 功能经实测不受影响）。
4. 启动器 `portable_entry.py` 实现用户数据目录隔离（`%LOCALAPPDATA%\FinExtract`）、端口扫描（8000-8010）、单实例保护（run.lock）、健康检查后自动打开浏览器、优雅退出。
5. 修复发布级缺陷：stdout 被重定向（或 GBK 控制台）时，业务日志中的 📄 等表情字符触发 `UnicodeEncodeError` 导致任务失败；已在启动器对流设置 `errors="replace"`，不可编码字符降级为 `?`，不再中断处理。
6. 修复便携化缺陷：`plan_service` 的模型探活健康文件原先写入程序目录（`程序目录/data/model_health.json`），在只读安装目录下会导致探活结果无法持久化、方案永远不可用；已改为跟随 `FINEXTRACT_USER_DATA_DIR` 用户数据目录，与主程序数据目录策略一致。
7. 修复密钥配置加载缺陷（v1.1）：启动器原先只读取固定文件名 `config/.env`，用户将配置保存为 `API.env` 等其他名字时密钥完全不会被加载，网页提示「DEEPSEEK_API_KEY 未配置」。已改为**扫描 `%LOCALAPPDATA%\FinExtract\config\` 目录下所有文件**（文件名不限，支持 `API.env` / `.env` / `keys.txt` / 中文名等），解析其中 `KEY=VALUE` 行注入进程环境；加载在任何模型业务模块导入之前完成，路径锚定 `%LOCALAPPDATA%`、不依赖当前工作目录；兼容带 BOM 的记事本保存与引号包裹的值；日志只显示键名、绝不打印密钥值；真实进程环境变量始终优先于文件配置。修改密钥后重启程序即按新配置自动重新探活（探活结果带配置指纹，换 Key 不会沿用旧缓存）。
9. 修复质量检查 HTTP 500（v1.2，V31 专项）：`_load_quality_service()` 按文件路径动态加载 `backend/quality/quality_service.py`，此前 PyInstaller 未收集该文件，EXE 内缺失 → `FileNotFoundError` 穿透为裸 HTTP 500。修复为三层：① 打包时将 `backend/quality/` 整体随包分发；② 文件缺失时自动回退常规包导入（双保险）；③ 三个质量端点接入受控异常——结果文件格式异常返回 422 + 排障编号、计算异常返回 500 + 排障编号（traceback 只写服务端日志，不暴露给浏览器）、原文不可用时按 SOURCE_UNAVAILABLE 合法降级而非中断。前端同步展示排障编号。方案不可用时的前端引导（三步配置指引 + 《使用手册》链接）与 `/manual` 手册路由此前已预制，本次一并验证通过。
8. 新增不可用引导（v1.2）：方案卡片不再显示「请联系管理员」，改为显示具体缺失原因（如「DeepSeek：缺少密钥」「Qwen Plus：连接验证未通过」）；当无任何可用方案时，页面渲染蓝框自助指引块（打开配置目录 → 写入密钥 → 重启三步），并提供「打开《使用手册》」链接；手册通过新增 `/manual` 路由随网站直接打开（文件已打入 `_internal`，EXE 同级副本兜底），无需用户去文件夹翻找。

## 三、真机验收结果

### 3.1 服务启动与基础设施（EXE 实测）

| 检查项 | 结果 | 证据 |
|---|---|---|
| EXE 启动（onedir，无 Python 依赖） | 通过 | 双击进程存活，Uvicorn 正常监听 |
| run.lock 单实例保护 | 通过 | 启动器写入端口锁；重复启动探测到健康实例后仅打开浏览器 |
| 端口 8000-8010 自动扫描 | 通过 | 占用时自动换端口 |
| 健康检查后自动打开浏览器 | 通过 | 探测 `GET /` 200 后调用系统浏览器 |
| 优雅退出（Ctrl+C / 关闭窗口 / taskkill） | 通过 | Uvicorn 正常 shutdown，run.lock 自动清理 |

### 3.2 模型探活与方案门禁（真实 API 调用）

- 服务启动后台自动探活：`deepseek-flash` 与 `qwen-plus` 均返回 `ok=true`（含真实供应商 request_id，延迟 1.8s / 2.2s）。
- `/api/processing-plans` 迅捷、智衡两方案 `ready=true`，无阻断项。
- 探活健康文件随 `FINEXTRACT_USER_DATA_DIR` 落入用户数据目录（修复后），程序目录只读场景（如 Program Files）不再影响探活持久化。

### 3.3 真实任务验收（迅捷 / 智衡，付费模型真实调用）

测试样本：`600763_20261001_FB8S.pdf`（国城矿业股份质押公告，99.5KB）
配置场景：密钥写入**随意命名的文件** `config/我的密钥_backup-2026.txt`（验证 v1.1 任意文件名加载）

| 项目 | 迅捷 swift | 智衡 balance |
|---|---|---|
| 任务 ID | job_20261015202058_2bdf28 | job_20261015202248_9f8417 |
| 终态 | partial（主提取完整，无错误信息） | **completed** |
| 耗时 | 110 秒 | 80 秒 |
| 事件数 | 1 | 1 |
| 字段数 | 35（发布 22 / 待复核 1 / 未发布 12） | 35（发布 21 / 待复核 2 / 未发布 12） |
| 独立复核（Verify） | 不含（方案设计如此） | **completed**（Qwen Plus 真实复核执行） |
| 复核差异接口 /api/v29/.../balance-differences | 不适用 | HTTP 200，state=completed，2 条真实差异 |

- 任意文件名配置加载验证通过：密钥从 `我的密钥_backup-2026.txt` 注入后，启动探活 deepseek-flash 与 qwen-plus 均 `ok=true`（真实 request_id）。
- v1.2 无密钥降级场景（空 config 目录实测）：首页 HTTP 200 正常打开；迅捷/智衡均 ready=false，blockers 准确报告 `MISSING_CREDENTIAL`（DeepSeek / Qwen Plus）；启动日志输出降级提示「网页可正常打开，配置后重启即可使用」；前端引导块（三步配置指引 + /manual 手册链接）代码与路由均验证存在。
- v1.1 任意文件名回归：密钥写入随意命名的 `我的密钥_backup-2026.txt`（含中文名），双模型探活均 ok=true、双方案 ready=true。
- 全程无 `UnicodeEncodeError`；无模拟调用，全部结果为真实付费 API 返回。
- 迅捷本次终态为 partial：主提取与字段发布均完整（事件/字段计数与智衡一致），Repair 环节按 `on_high_risk` 策略本次未形成完整闭环，属合法终态而非失败（partial 语义见「已知限制」）。

### 3.4 质量检查接口回归（v1.2，修复后实测）

| 检查项 | 结果 |
|---|---|
| POST /api/quality/jobs/{id}/run（balance 任务） | **200**（修复前 HTTP 500，traceback 确认为 quality_service.py 未入包） |
| 幂等性（重复 POST） | 两次指标完全一致 |
| GET /api/quality/jobs/{id} | 200 |
| 导出 format=json / format=csv | 均 200 |
| 用户真实任务 ×3（含迅捷/智衡，从真实数据目录复制的副本） | 全部 200，doc_hash=MEASURED，persist=saved |
| 原文不可用任务 | evidence_verified_rate=None (NOT_RUN)，合法降级不中断、不伪造 0 |
| 坏 result.json（副本测试） | **422** `QUALITY_INPUT_ERROR` + 排障编号（如 99ce100e28） |
| 结果文件缺失 | **404**「结果文件不存在」 |
| 不存在任务 | **404**「任务不存在」 |
| /manual 使用手册路由 | 200，完整手册页面 |
| 审计快照落盘 | quality_audit.json 已写入任务目录 |
- 前端四页面经浏览器实测：中文导航、KPI 卡片真实数值、任务表格、无 Console 报错。
- 退出方式为强制 taskkill 时 run.lock 会残留，但启动器探测到端口无健康实例后会自动清锁重启（自愈设计，实测通过）；用户正常关闭窗口 / Ctrl+C 时锁自动清理。

## 四、已知限制（如实披露）

- 程序未做代码签名，首次运行 Windows SmartScreen 会提示，需点「仍要运行」。
- OCR 为纯 CPU 推理，扫描件处理速度取决于页数与机器性能。
- `partial` 状态（复核环节中断但主结果可用）为合法终态，详情见任务错误信息。
- onedir 打包内置完整 Python 运行时，目标电脑无需安装 Python；本报告验收在构建机完成，未做第二台无 Python 电脑的交叉实测。
- 模型密钥不随包分发：用户需按使用手册**第三章「配置模型密钥」**，在 `%LOCALAPPDATA%\FinExtract\config\` 下任意命名文件中写入 `DEEPSEEK_API_KEY` 与 `QWEN_API_KEY` 后重启程序完成探活。

## 五、v1.3 首屏配置引导与手册重点化（V32，本次更新）

**改动清单**

1. 首页未配置密钥提示（`index.html` + `components-fe.css`）：工作台顶部新增金色引导横幅，仅当 `/api/processing-plans` 返回的全部方案均不可用时渲染；标题「尚未配置模型密钥，暂时无法开始分析」，正文指向《使用手册》第三章，附 `/manual` 直达链接；有任一可用方案时不显示，不干扰老用户。
2. 使用手册重构（`使用手册.html`）：「配置模型密钥」由第四章提升为**第三章**并加「重点必读 · 开始分析前的必要步骤」金色标注；步骤由 4 条纲要细化为 5 个完整步骤卡（获取密钥途径 DeepSeek/阿里云百炼 → 打开配置目录 → 新建配置文件 → 写入密钥 → 重启验证），附可直接复制的配置块、「配置后仍不可用」六项自查表与隐私说明；「快速开始」降为第四章并新增「配置密钥（首次使用必做）」衔接步骤；FAQ Q3 章节引用同步修正。
3. 提取页引导链接同步（`plan-selector.js`）：「打开《使用手册》查看图文指引」由「第四节：模型密钥配置」改为「第三章：配置模型密钥」。

**验证结果（v32_banner_check.py，隔离 LOCALAPPDATA 空配置全新用户场景实测，14/14 通过）**

- HTTP 层：首页 200 且含横幅挂载点；两套方案 ready=false、原因准确为 MISSING_CREDENTIAL；`/manual` 200（12213 字节），含第三章标题、重点必读标注、密钥配置块、五步操作与自查表。
- 渲染层（Playwright + Chromium 真实浏览器）：金色横幅真实可见，标题/正文/链接断言全部通过，截图存档 `v32_banner_shot.png`；手册章节顺序断言通过（密钥配置第三章在快速开始第四章之前）。
