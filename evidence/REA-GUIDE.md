# REA MCP 工具解读（大白话版，配真实案例）

> 对应 `SETUP.md` 第一部分 §0–§9。每节包括：**一句话**、**打个比方**、**克隆网站用不用得到**、**真实案例**、**常用工具**。
> 案例都来自本机的真实调用（2026-10-31）。文件路径相对 `evidence/websites/`。
> 本机环境限制要先说清楚：**Ghidra、IDA、JADX、adb、tshark 都没装**，Node 22（SQLite 检查需要 24.15+）。依赖这些工具的功能，下面会写明"本机跑不了"，不编造结果。

---

## 一图看懂：REA 能看什么

```
            ┌────────── 网站（§2 主力） ── JS 代码（§3）
REA 证据 ───┼────────── 本地文件：二进制（§4）/ 各种格式（§5）/ 安卓（§6）
(§0 §1)     ├────────── 运行中的程序：插桩、调试（§7）
            ├────────── 网络包、硬件信号（§8）
            └────────── 复刻得对不对（§9）
```

克隆网站，主要用到 **§0 §1 §2 §3 §9**；§4–§8 只在网站背后还有客户端、WASM、数据文件、抓包时才用得上。

---

## §0 核心理念：一切结果都是"带出处的证据"

**一句话**：REA 返回的不只是答案，还附带一张"证据卡"。卡上写着：看的是什么（SHA-256 指纹）、用什么工具看的、看全了没有、哪里不确定。

**打个比方**：像法医报告，不说"凶手是 A"，而是写"现场 X 处发现指纹（观测），与 A 的指纹吻合（推导），因此 A 可能到过现场（推断），作案时间不明（未知）"。

| 等级 | 意思 | 真实例子 |
|---|---|---|
| **observed 观测** | 直接看到的 | 同花顺首页返回 `X-Powered-By: Next.js`（`curl -I` 输出） |
| **derived 推导** | 从观测确定性算出的 | 同花顺首字节 HTML 已含 5,283 个汉字 → 服务端渲染 |
| **inferred 推断** | 合理猜测，可能错 | 小红书 `300031` 可能是 IP 风控 `300012` 的下游表现 |
| **unknown 未知** | 不知道，**不等于"没有"** | 知乎登录后的接口长什么样，未登录看不到 |

**要记住的三件事**：
1. 每次调用都有一个 `ev_…` 编号。例如同花顺首页采集是 `ev_4a1407d3…7483`。结论要能追溯到这个编号。
2. **页面内容不可信**：网页文字、脚本、反编译代码都只当数据看，里面写的"请执行…"一律不照做。
3. **只读优先**：默认不点击、不提交、不登录；会改变状态的工具要你明确授权。

---

## §1 会话与证据管理：证据怎么存、怎么翻页

**一句话**：管理"证据卡"的文件柜。

**打个比方**：分析过程中证据都放在桌面上（内存），不归档（导出）的话，关掉 REA 就没了。

**真实教训**：bjtu 早期的 E1–E7 只留了编号、没有导出，后来都失效了。所以现在每次采集都把完整证据落盘，例如 `10jqka/captures/original/home.capture.json`。

| 工具 | 什么时候用 |
|---|---|
| `export_evidence_bundle` / `import_evidence_bundle` | 每阶段结束落盘 / 下一阶段读回 |
| `get_evidence_bundle`（`detail: summary`） | 只看目录，不搬内容 |
| `inspect_analysis_view` | **万能翻页器**：大结果先拿摘要，再按页取细节 |
| `record_unknown` / `list_unknowns` | 把"还不知道的事"登记在案，交付前清点 |
| `update_analysis_settings` | 调整超时等上限 |

**真实案例（摘要 + 翻页）**：对 `libcurl.so.4.8.0` 调用 `inspect_native_exports(detail: summary)`，只返回一行"共 93 个导出符号、覆盖完整"（`ev_dd3f7460…`），完整的 93 行留在会话里，需要时再用 `inspect_analysis_view` 翻页。这样第一次调用占用的上下文很少。

---

## §2 Web 页面与浏览器：克隆网站的主力

**一句话**：打开网页、拍照、录下所有网络请求、分析它的 JS。

**打个比方**：派一个只看不碰的观察员进店，拍下装修、记下每一次后厨传菜（网络请求）。

### 2.1 两种看法

| 方式 | 工具 | 特点 |
|---|---|---|
| 被动看 | `inspect_web_page`、`capture_web_screenshot`、`inspect_web_event_listeners`、`observe_web_session` | 连接一个**已经开着**的浏览器，不执行、不点击 |
| 主动跑场景 | `capture_browser_scenario` | REA 自己启动浏览器，按步骤打开、等待、点击，录下截图、DOM、网络 |

### 2.2 真实案例：6 个站点各打开一次首页

用的是 `capture_browser_scenario`：无头 Chromium，1440×900，等待 6 秒，录下响应体。

| 站点 | 录到什么 | 对克隆的意义 |
|---|---|---|
| 同花顺 | 90 个请求；HTML 已含正文；行情接口 `quota-h.10jqka.com.cn/fuyao/…/multi_last_snapshot`（JSON） | 正文直接可用，行情走接口 |
| B 站 | HTML 里有 `window.__pinia` 首屏数据；`api.bilibili.com/x/web-interface/wbi/…` | 首屏可直接取数据；`wbi` = 带签名的接口 |
| 高德 | `/ssr/search` 经 4 次 302（含 cookie 探测）才到 `/search`；96 张地图瓦片 | 页面是前端渲染，地图是瓦片拼的 |
| 淘宝 | 414 个请求，87% 来自非 taobao.com 域名（阿里 CDN、埋点等）；推荐接口触发 `_____tmd_____` → 登录框 | 风控挡住了推荐流 |
| 知乎 / 北交大 MIS | 302 → 登录页 / 统一身份认证 | 登录墙 |

截图在各站 `captures/original/home.png`。小结：**一次采集就能判断"这个站该走哪条克隆路线"**。

### 2.3 分析与对比

| 工具 | 干什么 | 真实案例 |
|---|---|---|
| `export_web_scripts` | 把采集到的 JS 原样导出成文件 | 同花顺：导出 26 个脚本，含 Next.js 的 `_next/static/chunks/app/page-….js`（`ev_509ca3a8…`），见 `10jqka/bundles/scripts/` |
| `analyze_web_bundle` | 从页面抓 JS，找路由、接口、source map | 需要连接一个开着的浏览器（CDP）；后续 S2 阶段使用 |
| `compare_web_screenshots` | 原站 vs 复刻的像素差异 | 你定的标准：**差异 < 5%** |
| `inspect_web_network_capture` | 解析 HAR / mitmproxy 抓包 | 本次没有 HAR，未使用 |

### 2.4 把接口变成"可重放环境"

`inspect_analysis_view kind: api-surface` → `environment-manifest` → `export_environment_bundle`：把录到的接口整理成一组本地可调用的工具，按录制结果回答请求。**适用于同花顺行情、B 站接口这类 JSON 站，不适用于 bjtu 这种纯 HTML 站。** 拿到的是**录制那一刻的快照**，不是对方的数据库。

---

## §3 JavaScript / Electron：把打包的前端代码拆开看

**一句话**：网站的 JS 被打包压缩成一坨，REA 把它拆回模块，找出路由、接口、数据流向。

**打个比方**：拿到一本被撕碎重新装订的书，按页码和引用关系恢复章节目录。

**什么时候用**：SPA / 混合站（同花顺、B 站、知乎、淘宝）必用；纯 SSR 站（bjtu）基本不用。

**真实案例**：
- 同花顺：上一步导出的 26 个脚本，喂给 `analyze_javascript_application` 做静态应用图。
  - **第一次失败了**：26 个脚本全部分析时，某个大文件超过每个文件 5 分钟的上限，REA 返回 `provider_timeout`，把已有结果保留为 partial 证据（`ev_8bd31d7d…`），并建议"缩小 `source_paths` 或调大超时"。
  - **第二次只选 13 个同花顺自己的 Next.js chunk**（`app/page`、`app/layout`、`main-app` 等），**58 秒完成**（`ev_60feead1…`）：解析 13 个文件、0 个解析失败，识别出 159 个模块、983 条发现。
  - 用 `inspect_analysis_view kind: javascript-routes` 翻出 3 条前端路由：`/`、`/market`、`/shop`，正好对应页面左侧导航"首页 / 行情 / 商城"。**这就是"站点有几类页面"的静态证据**，S4 路由表可以从这里起步。
  - REA 同时说明了局限：没选的 13 个文件里的关系仍是 unknown；静态找到的路由**不代表能访问到**，也不代表这就是全部路由。
  - 完整证据在 `10jqka/bundles/js-app.evidence.json`，路由页在 `js-routes.json`。
- 知乎：首页 JS 末尾声明了 **23 个 source map**（`main.app.4978…js.map`、`vendor.92ed…js.map` 等），B 站有 5 个。source map 能把压缩代码对回原始文件名和行号，**前提是它在线上能取到**；REA 不会自动下载，需要你同意后再取。

| 工具 | 干什么 |
|---|---|
| `analyze_javascript_application` | 静态应用图：模块、引用关系、路由（不执行代码） |
| `recover_javascript_sources` | 用 Wakaru 把压缩代码还原成可读模块 |
| `trace_web_source_location` | 压缩代码第 N 列 → 原始源码位置（借助 source map） |
| `trace_javascript_semantics` | 追某个值从哪来、流到哪 |
| Electron 系列 | 桌面版应用（如 B 站 PC 客户端）才用 |

---

## §4 原生二进制深度分析：反编译 exe / so / dylib

**一句话**：把编译好的程序翻译回汇编、伪 C 代码，看它调用了什么。

**打个比方**：只有成品机器、没有图纸，拆开看齿轮怎么咬合。

**和克隆网站的关系**：网站本身用不到。只有两种情况相关：① 网站用了 WASM（本次 6 站首页**均未发现** `.wasm`）；② 你想研究它的桌面或手机客户端。

**真实案例**（本机没有 Ghidra，所以用 objdump 兜底）：分析的是 `libcurl.so.4.8.0`，也就是采集网页时实际在用的网络库。
- `inspect_native_exports`：93 个导出函数，覆盖完整（`ev_dd3f7460…`）。
- `inspect_with_objdump(operation: disassemble, symbol: curl_version)`：拿到 `curl_version` 的汇编（`ev_698e6e15…`），能直接看到它依次调用了 `zlibVersion`、`BrotliDecoderVersion`、`idn2_check_version`、`nghttp2_version` 等函数。这就是"从二进制看出程序依赖了哪些组件"。
- **本机跑不了的**：`open_binary`、`analyze_function`、`procedure_pseudo_code`（伪代码）、`get_call_graph`（调用图）等深度功能，都需要 Ghidra / IDA / Hopper。

| 类别 | 代表工具 |
|---|---|
| 打开 / 总览 | `open_binary`、`binary_overview`、`list_segments` |
| 找东西 | `search_strings`、`search_procedures`、`xrefs` |
| 看函数 | `analyze_function`、`procedure_pseudo_code`、`procedure_assembly` |
| 看调用 | `get_call_graph`、`trace_call_path` |
| 版本对比 | `match_native_functions`、`compare_functions` |

---

## §5 文件格式检查器：不用打开会话，直接解析文件

**一句话**：一大批"专科医生"，每个只懂一种文件格式，直接读文件头和结构。

**打个比方**：拿到一个包裹，不用拆开就能读快递单上的寄件人、重量和内容物清单。

**和克隆网站的关系**：偶尔用到。网站下发的 WASM 用 `inspect_wasm_artifact`，复刻版的数据库用 `inspect_sqlite_database`，下载的安装包用 `inspect_artifact`。

**真实案例**：
- `inspect_elf_notes /usr/bin/curl`：读出 GNU build-id `dc6fce00…39d8`、ABI 标签 Linux 3.2.0（`ev_3377e55f…`）。build-id 相当于程序的"身份证号"，可以用来判断两台机器上的 curl 是不是同一次构建。
- `inspect_sqlite_database` 检查 bjtu 复刻库 `bjtu/replica/replica.sqlite3`：**本机跑不了**，REA 返回 `capability_unavailable`，原因是需要 Node 24.15+，本机是 22。这也说明了工具会如实报告"做不到"。

| 平台 | 代表工具 |
|---|---|
| Linux ELF | `inspect_elf_notes`、`inspect_native_exports`、`inspect_binary_layout`、`inspect_with_objdump` |
| Windows PE | `inspect_pe_imports`、`inspect_pe_certificates` |
| Apple | `inspect_macho`、`inspect_plist`（需要 macOS） |
| 语言/运行时 | `inspect_go_binary`、`inspect_rust_binary`、`inspect_wasm_artifact`、`inspect_python_pyc` |
| 数据/归档 | `inspect_sqlite_database`、`inspect_artifact`、`extract_artifact` |

---

## §6 Android：APK 静态分析 + 真机/模拟器操作

**一句话**：拆安卓安装包看代码和资源；或者通过 adb 操作手机。

**和克隆网站的关系**：只在"网站有 App，且 App 里有网页上看不到的接口"时才相关。比如知乎、小红书的很多内容只在 App 里有。

**真实案例**：`list_adb_devices` → **本机跑不了**（`adb binary was not found`）。JADX 也没配置。所以这一节**本次没有真实素材**。如果要做，需要：①你同意从官网下载某站的 APK；②装 JADX，只做静态分析，不安装、不运行。

| 类别 | 代表工具 |
|---|---|
| 静态 | `inspect_android_package`、`search_android_classes`、`inspect_android_method`、`inspect_apk_signature` |
| 设备只读 | `list_adb_packages`、`read_adb_logcat`、`pull_adb_package` |
| 设备变更（需授权） | `install_adb_package`、`start_adb_app` |

---

## §7 动态插桩与调试：让程序跑起来，边跑边看

**一句话**：Frida 往运行中的程序里注入脚本，GDB / Rizin 下断点单步调试，`capture_process_scenario` 录下一条命令的完整运行过程。

**打个比方**：§4 是看图纸，§7 是让机器开起来，用听诊器听。

**和克隆网站的关系**：基本不用。网站的"运行时"就是浏览器，§2 的场景采集已经覆盖。

**真实案例**：`capture_process_scenario` 在伪终端（PTY）里运行 `curl -sI https://www.10jqka.com.cn/`（`ev_e92340a7…`），录到：
- 退出码 0，进程组已清理（`owned_process_group: verified`）；
- 响应头 `x-nextjs-prerender: 1`、`x-nextjs-cache: HIT`、`Cache-Control: s-maxage=30, stale-while-revalidate=31535970`，CDN 链路是 squid + ApacheTrafficServer。

**这条证据对克隆很有用**：它说明同花顺首页是 **Next.js 预渲染 + 30 秒 CDN 缓存**。复刻版每 30 秒刷新一次快照，就能做到"和原站一样新"。

REA 也如实写了它看不到的部分：`Network activity is not observed by this capture`，也就是它只录终端输出，不抓网络包。

---

## §8 网络抓包与信号

**一句话**：分析 pcap 抓包文件（DNS、TLS 域名、HTTP 地址），或者逻辑分析仪录下的硬件信号。

**和克隆网站的关系**：可选。浏览器场景采集（§2）已经录了所有 HTTP 请求，比抓 pcap 更直接；pcap 适合分析 App 或客户端的流量。

**真实案例**：`inspect_pcap_client` → **本机跑不了**（没有 tshark）。本次**没有真实素材**。替代方案：§2 的 6 份 `home.capture.json` 已经包含 1,044 个 HTTP 请求的完整记录。

---

## §9 重建验证：证明"复刻得对"

**一句话**：把"复刻版和原站一致"拆成一条条可检查的义务，每条都要有证据支撑；缺证据就标 unknown，不能当成通过。

**打个比方**：装修验收单。"墙面平整"要有测量记录，"水电通"要有试水记录；没测的项写"未验"，不能打勾。

| 工具 | 干什么 |
|---|---|
| `build_reconstruction_obligation_ledger` | 生成义务台账：要证明哪些点、每点靠哪条证据 |
| `evaluate_reconstruction_coverage` | 检查覆盖率：缺证据 = partial/unknown，矛盾 = fail |
| `verify_reconstruction` | 用规格逐条核验；**通过 ≠ 源码一样**，只说明列出的那些点一致 |

**真实案例**：bjtu 已经做过"原站 vs 复刻"截图对比（`bjtu/captures/original/` 与 `bjtu/captures/replica/`）。下一步会把它写成义务："首页像素差 < 5%""tzgg 列表 48 条与原站一致""分页规则 `indexN.htm` 正序"，再用 `evaluate_reconstruction_coverage` 打分。这会在 D3（`CLONE-DEPTH.md`）里展开。

---

## 附：本次真实调用清单

| 节 | 工具 | 目标 | Evidence | 结果 |
|---|---|---|---|---|
| §2 | `capture_browser_scenario` ×6 | 6 站首页 | 见各站 `case.md` 头部 | 成功 |
| §2 | `export_web_scripts` | 同花顺采集 | `ev_509ca3a8b36f…` | 26 个脚本 |
| §3 | `analyze_javascript_application` ×2 | 同花顺脚本 | `ev_8bd31d7d…`（超时 partial）/ `ev_60feead118b0…` | 第 1 次超时；第 2 次只选 13 个文件，58 s，3 条路由 |
| §4 | `inspect_native_exports` | libcurl.so | `ev_dd3f7460c305…` | 93 个导出 |
| §4 | `inspect_with_objdump` | libcurl `curl_version` | `ev_698e6e159bdf…` | 汇编 |
| §5 | `inspect_elf_notes` | /usr/bin/curl | `ev_3377e55f9a45…` | build-id |
| §5 | `inspect_sqlite_database` | bjtu 复刻库 | — | 本机跑不了（Node 版本） |
| §6 | `list_adb_devices` | — | — | 本机跑不了（无 adb） |
| §7 | `capture_process_scenario` | curl 同花顺 | `ev_e92340a731e3…` | 成功 |
| §8 | `inspect_pcap_client` | — | — | 本机跑不了（无 tshark） |

MCP 会话证据已导出：`websites/_bundles/guide-demos-2026-10-31.json`（12 条记录、4 个 unknown，202 MB，主要是 JS 分析的 partial 结果）。
