# aardio 单文件 exe 封包避坑手册（通用版）

> **用途**：把任意纯前端网页（HTML+JS+CSS+资源）打包成**单文件便携 exe**，运行时由 WebView2 渲染、用 `wsock.tcp.simpleHttpServer` 从内存资源流提供页面。  
> 本文档由多次真实踩坑（271c / 图标缓存 / 关闭卡死 / 进程残留 / IndexedDB 跨运行丢失）提炼，**与具体业务无关**，可直接用于下次打包其它网页程序。  
> 配套完整范式见文末「✅ 最终 main.aardio 范式」。



> **严重度图例**：🔴 致命（不修必翻车）｜⚠️ 重要（易卡很久）｜💡 注意（省时间的细节）

---

## 〇、打包原理速记（先理解再动手）

- aardio 把指定网站目录（如 `/web`）作为**内嵌资源**整体打进 exe，**非外置**。
- 运行时 `wsock.tcp.simpleHttpServer` 从内存资源按 HTTP 请求喂给 `web.view`（WebView2/Chromium）。
- 入口 `wb.go("/web/index.html")`，页面内 `./static/...`、其它 html 均相对 `/web` 解析。
- **唯一外部依赖** = Microsoft Edge WebView2 运行时（Win10/11 预装，未打包；Win7 需手动装）。
- 体积：依赖全内嵌，必然偏大（中文字体最占空间）。若想瘦身必须 `/web` 外置（牺牲单文件便携）。

---

## 一、工程结构与 `.aproj` 资源清单（🔴 271c 元凶）

### 坑 1：`.aproj` 必须「平铺」，禁止嵌套 folder

- **现象**：F7 报 `271c`（`\xxx.html` 未找到），连锁 `7005`（生成 exe 失败）。
- **根因**：在 `web` 下又嵌套了 `static` 子 `<folder>`。aardio 发布时的 HTML 依赖预扫描未能正确注册 `/web/index.html`，于是把 `index.html` 里的 `./guest-screen.html` **回退到从工程根解析** → 去找 `\guest-screen.html` → 找不到 → 271c。
- **正确写法**：只用一个 `<folder name="web" embed="true">`，其下**直接列出全部 `<file>`**（含 `web\static\xxx`），**严禁**把 `static` 等再套成嵌套 `<folder>`。
  ```xml
  <folder name="web" path="web" embed="true">
      <file name="index.html"        path="web\index.html"/>
      <file name="guest-screen.html" path="web\guest-screen.html"/>
      <file name="app.js"            path="web\static\app.js"/>
      ...（所有 web 下文件逐一直接列出）
  </folder>
  ```

### 坑 2：每个 `<file path>` 写全路径

- 必须带 `web\` 前缀的反斜杠全路径：`web\guest-screen.html`、`web\static\app.js`，**不能只写文件名**。

### 坑 3：生成 `.aproj` 后必须校验

- 脚本遍历真实 `web/` 生成 `.aproj`，并断言**所有 `path` 在磁盘真实存在**（`missing == NONE`）再交付构建。否则漏文件只会在 F7 时才暴露。

### 坑 4：工程目录要写到真实磁盘

- ⚠️ 若用沙箱/远程环境改过 `aardio-project/`，确认它已落到**真实磁盘**。曾出现“只存在于沙箱视图、真实磁盘缺失”导致无法复现构建。

---

## 二、程序图标（💡 缓存错觉最多）

### 坑 5：`icon` 属性写法

- `<project>` 加 `icon="\icon.ico"`：**前导反斜杠 = 工程根相对路径**（沿用官方 `WinAsar.aproj` 写法）。`icon.ico` 与 `.aproj` 同目录。
- 图标是**编译期 PE 资源**，**不要**列入 embed `<folder>` 资源清单。
- `icon="\icon.ico"` 写法正确，**不要**改成 `icon="icon.ico"`（无反斜杠反而破坏已验证配置）。

### 坑 6：打包后图标“看着没变”≈ Windows 图标缓存

- 同名同路径 exe 被覆盖时，资源管理器显示**旧缓存缩略图**，不代表打包失败。
- **判定是否真嵌入**：用 `pefile` 解析 exe 的 `RT_ICON`，若 9 张图字节数与 `icon.ico` 逐张吻合，则已正确嵌入。
- 让原名显示正确图标：结束 `explorer.exe` → 删 `%LocalAppData%\Microsoft\Windows\Explorer\iconcache*.db` → 重启 `explorer.exe`；或把 exe 改名/移到新路径（新名立即显示）。

---

## 三、构建 / 编译（⚠️ 最容易白忙）

### 坑 7：aardio 无法无头编译

- `aardio.exe /build xxx.aproj` 会直接拉起 IDE 窗口并等待显示会话，无图形界面时 `rc=124` 超时、零输出、不生成文件。
- **必须在有图形界面的机器上**：打开 aardio → `文件 → 打开工程` → 选 `.aproj` → 按 **F7 发布**。

### 坑 8：不要只双击 `main.aardio`

- 只双击 `main.aardio` 当作临时工程发布 → 不会读取 `.aproj` 资源清单 → 必报 271c。务必打开 `.aproj`。

### 坑 9：F7 完成框别勾「版本号自增」

- 勾了下次 F7 自动变 1.1.2。建议取消勾选，版本号在 `.aproj` 写死。

### 坑 10：杀软拦截

- 仅弹 `7005`（"安全监控类软件"）多为 Defender 实时防护拦截 exe 写入，临时关闭防护后重试。

---

## 四、关闭退出 & 进程管理（🔴 最磨人的一类）

### 坑 11：`winform.onClose` 里【禁止】任何同步阻塞操作

- 以下都会阻塞主消息循环，导致窗口卡死 / 报 “C stack overflow” / 显示 “未响应”：
  - `wb.closeAndWait()`：内部是 `while(isWindow(hwndChrome)) delay(10)` 死循环（父窗口在 onClose 中根本没机会销毁，循环永不结束）。
  - `wb.doScript()`：关闭时 WebView2 不响应 → 主线程卡住。
  - `thread.delay()`：直接阻塞消息循环 → Windows 标记 Not Responding。
- **正确**：`onClose` 只设 `isClosing` 标志，**返回 `null`** 走默认关闭流程，不做任何清理。

### 坑 12：真正清理放 `winform.onDestroy`

- `onDestroy` 中：① kill 本进程派生的 `msedgewebview2.exe` 子进程；② 用 `process(curPid).terminate(0)` 结束当前 exe 主进程。

### 坑 13：自杀【禁用】`::Kernel32.ExitProcess(0)`

- 实测在该 aardio + WebView2 场景下，`ExitProcess(0)` 后主进程仍从“应用”降级为“后台进程”继续驻留；后台 `thread.create` 线程也未按预期执行。

### 坑 14：自杀【禁用】`process(curPid).kill()`

- aardio 的 `kill()` 内部先 `suspend()` 再 `terminate()`；对自己调用会**冻结当前线程**，进程看似“从应用降级为背景进程”实际被挂起 → 无法退出。
- ✅ **自杀唯一正确**：`process(curPid).terminate(0)`（直接调用 Windows `TerminateProcess`）。

### 坑 15：`self` 是 aardio 保留字

- `var self = ...` 会报编译错误 `7007 Expected: '<name>' Near: 'self'`。自杀时改用 `var selfPrcs = process(curPid)`。

### 坑 16：必须做单实例互斥体

- `process.mutex("你的唯一名")`：冲突时前置已有窗口；若找不到窗口，则动态取当前 exe 名（`io.splitpath(process.getPath(pid)).file`）枚举 kill 残留同名进程后再启动。避免多实例抢端口、数据混乱。

### 坑 17：网页侧负责关闭 IndexedDB

- `index.html` 必须在 `beforeunload` 中 `dbManager.db.close()`，让浏览器默认关闭流程触发落盘。**不要在 aardio 中同步触发**。

---

## 五、IndexedDB 跨运行持久化（🔴 致命坑，本手册核心）

### 坑 18：simpleHttpServer 默认随机端口 → 记录跨运行“消失”

- **根因**：`web.view` 经 `wsock.tcp.simpleHttpServer` 提供页面，该服务器**默认每次启动分配随机空闲端口（49152~65535）**。而 IndexedDB 以“源（协议+主机+**端口**）”隔离，**端口一变源就变**，记录被写入各自独立的空库 → 重开即“无历史”。
- **实证**：磁盘 IndexedDB 目录下出现一堆 `http_127.0.0.1_<不同端口>.indexeddb.leveldb`。

### 坑 19：固定端口的【正确写法】——直接设置 startPort

- ✅ 在 `wb.go()` **之前**直接设置：
  ```aardio
  var FIXED_PORT = 52417;
  wsock.tcp.simpleHttpServer.startPort = FIXED_PORT;
  wb.go("/web/index.html");
  ```
- ❌ **不要先创建探测服务器再 `stop()` 释放**：`var t = wsock.tcp.simpleHttpServer("127.0.0.1", FIXED_PORT); t.stop();` ——该实例可能无法立即释放端口，导致 `wb.go()` 内部正式服务退到随机端口，IndexedDB 仍写在随机 origin 下（这是二次翻车的真实原因）。

### 坑 20：exe 源 vs 浏览器源天然不同（设计使然，非 bug）

- 浏览器直接打开 `index.html`（file:// 或固定 dev 端口）与 exe 版本（内嵌随机/固定 HTTP 源）是**不同源**，IndexedDB 互不通用，属于设计使然，不要误判为持久化 bug。

### 坑 21：重新初始化数据 = 删对应 IndexedDB 目录

- 路径：`%LocalAppData%\<你的userDataDir>\EBWebView\Default\IndexedDB\http_127.0.0.1_<端口>.indexeddb.leveldb`
  - 本项目示例：`C:\Users\<用户>\AppData\Local\电子礼簿\webview2\EBWebView\Default\IndexedDB\http_127.0.0.1_52417.indexeddb.leveldb`
- **端口即源即目录名**。固定端口后，要清数据只需删该端口对应的文件夹（先彻底关闭 exe，否则文件被锁）。
- 历史 bug 留下的其它随机端口文件夹是废弃空库，可顺手清理。

### 坑 24：固定端口后，WebView2 会缓存旧页面 →「改了代码打包后功能没更新」

- **现象**：修正端口为固定值（如 52417）并 persistent `userDataDir` 后，改了网页代码、F7 重新打包，**运行后新功能却没出现**（比如副屏脱敏开关加了但 exe 里没有）。
- **根因**：`wsock.tcp.simpleHttpServer` 对静态资源会返回 `Etag` + `Last-Modified` 头、并响应 `If-None-Match` → **304**；且**不带 `Cache-Control: no-store`**。端口固定前，每次运行端口都变、URL 不同，缓存键跟着变，所以从不踩雷；**端口一旦固定，URL 长期稳定，WebView2 的 HTTP 磁盘缓存就按 URL 复用了旧页面**（即使 exe 已重建，旧 URL 的缓存条目仍可能被命中）。
- **✅ 正确写法（禁用 HTTP 缓存，一劳永逸）**：在创建 `web.view` 时传入 Chromium 启动参数 `--disable-http-cache`，让浏览器**永远直接从 exe 内嵌资源（simpleHttpServer 提供的当前版本页面）读取**，根本不写、不读磁盘 HTTP 缓存：
  - `main.aardio`：`var wb = web.view(winform, { userDataDir = ...; language = "zh-CN"; }, "--disable-http-cache");`
  - 副屏 `window.open("./guest-screen.html", ...)` 直接打开即可（同属该 WebView2，同样不吃缓存）。
  - 页面 URL 保持干净（`/web/index.html`，**不要**加 `?v=` 之类查询串）。
- **❌ 错误写法（曾在项目中踩过）**：用「每次运行 URL 都唯一」的缓存破坏符，例如 `wb.go("/web/index.html?v=" + GetTickCount())` 或 `?v=" + APP_VERSION`。
  - 若按**每次启动**（毫秒数/随机串）变化 → 每打开一次 exe 就在磁盘留一份缓存副本，**开 1000 次 = 1000 份**，纯属污染磁盘，绝对不可取。
  - 若按**版本号**变化 → 同版本内 URL 不变、正常缓存（可接受），但「同版本号重建 exe、内容已变」时仍会命中旧缓存；且本质上仍依赖 HTTP 缓存机制，不如直接关掉干净。
- **为什么 `--disable-http-cache` 最对路**：它正好实现你想要的「缓存里的程序 = 打开的程序」——因为根本不存在 HTTP 缓存层，浏览器每次都实时读取 exe 内嵌的当前版本；不产生任何缓存副本；IndexedDB 是独立的 Web 存储（非 HTTP 缓存），不受此影响，历史记录依旧正常持久化。
- **一次性清旧缓存（升级后若仍见旧页面才需要）**：删 `%LocalAppData%\<App>\webview2\EBWebView\Default\` 下的 `Cache`、`Code Cache`、`GPUCache` 即可（**千万别删 `IndexedDB` 文件夹**，那会清掉礼簿记录）。关掉 HTTP 缓存后这些目录通常不再生成。

---

## 六、调试方法论（💡 盲修无效时）

### 坑 22：本环境无法测试 aardio GUI 构建

- 无头 `aardio /build` 会拉起 IDE 卡死。所有 `main.aardio` 改动只能靠读 aardio 源码推演（盲修），需你在**本机 F7 打包 + 运行测试**。

### 坑 23：盲修连败时，先“测量”再改

- 给 `main.aardio` 加日志：`io.file` 追加写 `%LocalAppData%\<你的App>\close-debug.log`，在 `onClose`/`onDestroy` 各阶段打点。
- 让你跑一次把日志发回，先看卡在哪一步，再针对性改，而不是继续猜。

---

## 七、交付前验证清单（每次都过一遍）

- [ ] `.aproj` 平铺、path 全路径、脚本校验 `missing == NONE`。
- [ ] 图标：pefile 查 `RT_ICON` 9 张与 `icon.ico` 逐张匹配。
- [ ] 版本：exe 属性 → 详细信息，文件版本 = 产品版本 = 设定值。
- [ ] 体积符合预期（全内嵌必然偏大）。
- [ ] 单实例：再次双击只激活已有窗口。
- [ ] 进程清理：关闭后任务管理器无残留 `msedgewebview2.exe`。
- [ ] 主进程退出：关闭后无残留 `<你的exe>.exe`，不卡顿、不报 “C stack overflow”、不 “未响应”。
- [ ] 数据持久化：新建数据 → 关闭 → 重开仍可见（IndexedDB 目录只剩固定端口那一个）。
- [ ] 运行：Win10/11 双击即开，本机已装 WebView2 运行时。

---

## 八、✅ 最终 main.aardio 范式（已验证可用）

> 把下面的“电子礼簿”/端口/路径换成你的项目即可。此为 2026-08-17 经过多轮修复后的稳定版。

```aardio
import win.ui;
import web.view;
import wsock.tcp.simpleHttpServer; // 必须早于 wb.go
import process.mutex;
import process;

var winform = win.form(text="你的程序标题"; right=1180; bottom=820; bgColor=0xFFFFFF);
winform.enableDpiScaling();

var myPid = ::Kernel32.GetCurrentProcessId();

// ── 单实例互斥体 ──
var exeName = "your_app.exe";
try {
    var f = ..io.splitpath(process.getPath(myPid)).file;
    if (f) exeName = f; // 动态取当前 exe 名，兼容改名
} catch(e) {}

var APP_MUTEX = "你的程序_单实例_唯一标识";
var singleMutex = process.mutex(APP_MUTEX);
if (singleMutex.conflict) {
    var hwnd = process.findWindow(,,"你的程序标题");
    if (hwnd) {
        ..win.setForeground(hwnd);
        if (::User32.IsIconic(hwnd)) ::User32.ShowWindow(hwnd, 0x9/*_SW_RESTORE*/);
        return;
    }
    // 找不到窗口：kill 残留无窗口同名进程后继续
    try {
        for entry in process.each(exeName) {
            if (entry.th32ProcessID != myPid) {
                var prcs = process(entry.th32ProcessID);
                if (prcs) { prcs.kill(); prcs.free(); }
            }
        }
    } catch(e) {}
}

var wb = web.view(winform, { userDataDir = io.appData("/你的程序名/webview2"); language = "zh-CN"; });

// ── 🔴 固定 HTTP 端口（保证 IndexedDB 跨运行持久化）──
// 直接设置，不要先探测再 stop（端口可能未释放 → 实际退随机端口）
var FIXED_PORT = 52417;
wsock.tcp.simpleHttpServer.startPort = FIXED_PORT;

// 辅助：kill 本进程派生的 WebView2 子进程
killWebView2Children = function() {
    try {
        var curPid = ::Kernel32.GetCurrentProcessId();
        for entry in process.each("msedgewebview2.exe") {
            if (entry.th32ParentProcessID == curPid) {
                var prcs = process(entry.th32ProcessID);
                if (prcs) { prcs.kill(); prcs.free(); }
            }
        }
    } catch(e) {}
}

// ── onClose：禁止任何同步阻塞，只设标志放行 ──
var isClosing = false;
winform.onClose = function(hwnd, message, wParam, lParam) {
    if (isClosing) return false;
    isClosing = true;
    return; // 返回 null/不返回，走默认关闭流程（触发网页 beforeunload）
}

// ── onDestroy：清理子进程 + 自杀（必须用 terminate）──
winform.onDestroy = function() {
    try { killWebView2Children(); } catch(e) {}
    try {
        var curPid = ::Kernel32.GetCurrentProcessId();
        var selfPrcs = process(curPid); // 注意 self 是保留字，用 selfPrcs
        if (selfPrcs) selfPrcs.terminate(0); // ✅ 自杀唯一正确方式
    } catch(e) {}
}

wb.go("/web/index.html");
winform.show();
win.loopMessage();
```

### 网页侧配套（`index.html`）

```js
// 关闭时由浏览器默认流程触发，关闭 IndexedDB 连接确保落盘
window.addEventListener("beforeunload", () => {
  if (window.giftBookApp?.dbManager?.db) {
    window.giftBookApp.dbManager.db.close();
  }
});
```

---

## 九、速查：最常翻车的 5 条（贴给 AI / 自己备忘）

1. **`.aproj` 平铺**，禁嵌套 folder → 否则 271c。
2. **aardio 不能无头编译**，必须 GUI 按 F7。
3. **`onClose` 零同步操作**，清理全放 `onDestroy`；自杀用 `process(curPid).terminate(0)`，禁用 `kill()`/`ExitProcess(0)`。
4. **`self` 是保留字** → 编译错 7007，改名 `selfPrcs`。
5. **`wb.go()` 前直接 `wsock.tcp.simpleHttpServer.startPort = 固定端口`**，否则 IndexedDB 跨运行丢失（禁探测-stop 写法）。
