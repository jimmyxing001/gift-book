# aardio 单文件 exe 打包指南（含避坑要点）

> 适用：把纯前端网页（如电子礼簿的 `/web`）打包成**单文件便携 exe**，运行时由 WebView2 渲染。
> 本文档由实际踩坑（271c / 图标缓存）总结，供下次打包其它文件时直接复用。

---

## 一、打包原理

- aardio 把指定的网站目录（本项目为 `/web`：`index.html` + `guest-screen.html` + `static/`）作为**内嵌资源**整体打进 exe。
- 运行时由 `wsock.tcp.simpleHttpServer`（aardio 内嵌 HTTP 服务）从**内存资源流**按 HTTP 请求把页面喂给 `web.view`（WebView2/Chromium）渲染。
- 入口在 `main.aardio`：`wb.go("/web/index.html")`，页面内 `./static/...`、`guestscreen.html` 均相对 `/web` 解析。
- **唯一外部依赖** = Microsoft Edge WebView2 运行时（Win10/11 系统预装，未打包进 exe；Win7 需手动安装）。
- 属于「绿色单文件」：无安装、无注册表、无需 exe 外的应用文件。但**依赖全部内嵌**，故体积偏大（本项目 15MB，中文字体约占 8.5MB）。

---

## 二、工程目录结构与关键文件

```
aardio-project/
├── gift-book.aproj     ← 工程定义（最关键，原仓库曾缺失）
├── main.aardio         ← 打包入口（窗口/WebView2/路由/单实例/清理）
├── icon.ico            ← 程序图标（与 .aproj 同目录）
└── web/                ← 网站根（被整体内嵌）
    ├── index.html
    ├── guest-screen.html
    └── static/  (js / css / 字体 / 图片)
```

- `main.aardio` 核心片段（已包含单实例与关闭清理）：
  ```aardio
  import win.ui;
  import web.view;
  import wsock.tcp.simpleHttpServer; // 必须早于 wb.go 引入
  import process.mutex;
  import process;

  var winform = win.form(text = "电子礼簿 v1.1.1"; ...);

  // 单实例 + 清理无窗口的残留进程
  var myPid = ::Kernel32.GetCurrentProcessId();
  var exeName = "gift-book_v1.1.1.exe";
  try {
      var f = ..io.splitpath(process.getPath(myPid)).file;
      if (f) exeName = f; // 动态取当前 exe 名，兼容改名
  } catch(e) {}

  var singleMutex = process.mutex("电子礼簿_SingleInstance_GiftBook_v1.1.1");
  if (singleMutex.conflict) {
      var hwnd, pid = process.findWindow(,,"电子礼簿");
      if (hwnd) {
          ..win.setForeground(hwnd);
          if (::User32.IsIconic(hwnd)) {
              ::User32.ShowWindow(hwnd, 0x9/*_SW_RESTORE*/);
          }
          return;
      }

      // 找不到窗口：kill 残留的无窗口同名进程，然后继续启动
      try {
          for entry in process.each(exeName) {
              if (entry.th32ProcessID != myPid) {
                  var prcs = process(entry.th32ProcessID);
                  if (prcs) { prcs.kill(); prcs.free(); }
              }
          }
      } catch(e) {}
  }

  var wb = web.view(winform, { userDataDir = io.appData("/电子礼簿/webview2"); language = "zh-CN"; });
  wb.go("/web/index.html");

  // 辅助函数：kill 本进程派生的 WebView2 子进程
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

  // ⚠️ 绝不在 onClose 里调用 wb.closeAndWait() / wb.doScript / thread.delay 等
  //    同步操作，否则会阻塞主消息循环，导致窗口“未响应”。
  var isClosing = false;
  winform.onClose = function(hwnd, message, wParam, lParam) {
      if (isClosing) return false;
      isClosing = true;

      // 返回 null，允许 aardio 走默认关闭流程。
      // 默认流程销毁窗口时 WebView2 会触发网页 beforeunload，index.html 负责关闭 IndexedDB。
      return;
  }

  // 窗口真正销毁时：先清理 WebView2 子进程，再用 process.terminate(0) 结束当前 exe 主进程。
  // 注意：process.kill() 内部会先 suspend() 再 terminate()，对自己调用会冻结当前线程，
  // 导致 terminate() 无法执行，因此自杀时必须用 process(pid).terminate(0) 直接终止。
  winform.onDestroy = function() {
      try { killWebView2Children(); } catch(e) {}

      try {
          var curPid = ::Kernel32.GetCurrentProcessId();
          var selfPrcs = process(curPid);
          if (selfPrcs) { selfPrcs.terminate(0); }
      } catch(e) {}
  }

  winform.show(); win.loopMessage();
  ```
- `web/` 必须与当前仓库源码同步（`index.html`、`guest-screen.html`、`static/` 一致）。

---

## 三、`.aproj` 关键写法（避坑核心 ⚠️）

### 根节点 `<project>` 属性
```xml
<project ver="10" name="电子礼簿" libEmbed="true" ui="win"
         output="gift-book_v1.1.1.exe"
         CompanyName="aardio" FileDescription="电子礼簿 - 红白喜事礼金记账系统"
         LegalCopyright="Copyright (C) aardio.com" ProductName="电子礼簿" InternalName="电子礼簿"
         FileVersion="1.1.1.0" ProductVersion="1.1.1.0"
         publishDir="/Publish/" dstrip="false"
         icon="\icon.ico">
```
要点：
- `output` / `FileVersion` / `ProductVersion` 写死版本号（如 `1.1.1.0`），避免依赖 F7 时手动填。
- `icon="\icon.ico"`：**前导反斜杠 = 工程根相对路径**，写法正确（沿用官方 `WinAsar.aproj` 的 `icon="\app.ico"`）。`icon.ico` 与 `.aproj` 同目录。
- `icon` 是**编译期 PE 资源**，**不要**列入 embed `<folder>` 资源清单。

### 资源清单必须「平铺」（271c 的根因）
```xml
<file name="main" path="main.aardio"/>
<folder name="web" path="web" comment="" embed="true">
    <file name="index.html"        path="web\index.html"        comment="web\index.html"/>
    <file name="guest-screen.html" path="web\guest-screen.html" comment="web\guest-screen.html"/>
    <file name="tailwindcss.js"    path="web\static\tailwindcss.js" comment="web\static\tailwindcss.js"/>
    ...（所有 web 下文件，含 web\static\xxx，逐一直接列出）
</folder>
```
**铁律：**
1. 只用一个 `<folder name="web" embed="true">`，其下**直接列出全部 `<file>`**。
2. **严禁**把 `static` 再套成嵌套 `<folder>` 子目录（这是 271c 的真正元凶）。
3. 每个 `<file path>` 写**全路径（反斜杠）**：`web\guest-screen.html`、`web\static\tailwindcss.js`，不能只写文件名。
4. 生成 `.aproj` 后务必脚本校验：所有 `path` 在磁盘上真实存在（`missing == NONE`）。

---

## 四、构建步骤（必须在 GUI 下，无法无头）

> ⚠️ **aardio 无法无头构建**：`aardio.exe /build xxx.aproj` 会直接拉起 IDE 窗口并等待显示会话，无图形界面时 `rc=124` 超时、零输出、不生成文件。必须在**有图形界面的机器**上操作。

1. 打开 aardio（便携版解压即用，或官网安装版）。
2. `文件 → 打开工程` → 选 `aardio-project\gift-book.aproj`（或直接双击 `.aproj`）。
   - **不要**只双击 `main.aardio` 当临时工程发布——那样不会读取 `.aproj` 的资源清单，必报 271c。
3. 按 **F7 发布** → 在 `aardio-project\Publish\` 生成 `gift-book_v1.1.1.exe`。
4. 完成框里：
   - **取消勾选「版本号自增」**（否则下次 F7 自动变 1.1.2）。
   - 不勾 UPX 压缩；不点「转换为独立 EXE」（已经是独立单文件）。
5. 若仅弹 7005（"安全监控类软件"），是杀毒/Defender 实时防护拦截 exe 写入，临时关闭后重试。

---

## 五、常见错误与解决

### ❌ 271c（`\guest-screen.html` 未找到）→ 连锁 7005（生成 EXE 失败）
- **根因**：`.aproj` 资源清单非平铺——在 `web` 下又嵌套了 `static` 子 `<folder>`。aardio 发布时的 HTML 依赖预扫描未能正确注册 `/web/index.html`，于是把 `index.html` 里的 `./guest-screen.html` **回退到从工程根解析** → 去找 `\guest-screen.html` → 找不到 → 271c → 7005。
- **修复**：改成「平铺结构」（见第三节）：单一 `web` folder 下直接列全部 file，每个 path 带 `web\` 前缀，不再有 `static` 子 folder。
- **验证**：脚本遍历真实 `web/` 生成 `.aproj`，并断言所有 `path` 磁盘存在。

### ❌ 程序图标颜色不正常 / 改名后才变
- **情况 A（完全没图标/默认图标）**：`<project>` 缺 `icon` 属性。修复：加 `icon="\icon.ico"`，并把 `icon.ico` 放到工程根（与 `.aproj` 同目录）。
- **情况 B（打包后"看着没变"）**：**几乎都是 Windows 资源管理器图标缓存**——同名同路径 exe 被覆盖时显示旧缓存缩略图，不代表打包失败。
  - 验证是否真的嵌入：用 `pefile` 解析 exe 的 `RT_ICON`，若 9 张图字节数与 `icon.ico` 逐张吻合，则已正确嵌入。
  - 让原名显示正确图标：结束 `explorer.exe` → 删 `%LocalAppData%\Microsoft\Windows\Explorer\iconcache*.db` → 重启 `explorer.exe`；或把 exe 改名/移到新路径（新名立即显示正确图标）。
- **注意**：`icon="\icon.ico"` 前导反斜杠写法正确，**不要**改成 `icon="icon.ico"`（那反而会破坏已验证可用的配置）。

### ❌ 关闭窗口后 exe/WebView2 仍在后台运行，多实例冲突
- **根因（关键）**：`winform.onClose` 里调用了 `wb.closeAndWait()`。查 aardio 源码，该函数内部是 `owner._form.close(); while(isWindow(hwndChrome)) delay(10); …`——它要等**父窗口（我们自己的 winform）先销毁**子窗口才会释放。但在父窗口的 `onClose` 回调里父窗口根本没机会销毁，于是 `while` 死循环，进程卡死在后台。
- **修复**：
  1. 启动单实例互斥体；若已存在则前置窗口，若找不到窗口则枚举并 kill 同名 exe 残留进程后继续启动。
  2. `winform.onClose` 中：**不调用 `wb.closeAndWait()`，也不调用 `wb.doScript` 或 `thread.delay`**。只做：设置 `isClosing` 标志 → 返回 `null` 允许默认关闭流程。
  3. `winform.onDestroy` 中清理 WebView2 子进程，然后使用 `process(curPid).terminate(0)` 结束当前 exe 主进程。实测 `::Kernel32.ExitProcess(0)` 在该场景下无法结束 aardio 主进程；而 `process(pid).kill()` 虽然能结束子进程，但对自己调用会先 `suspend()` 再 `terminate()`，导致当前线程被冻结、进程仍残留。因此自杀必须用 `process(pid).terminate(0)` 直接终止。
- **验证**：关闭窗口后任务管理器中不应再残留 `gift-book_v1.1.1.exe` 或 `msedgewebview2.exe`；再次双击 exe 应激活已有窗口而非新建实例。

### ❌ exe 中创建的历史记录，重启后消失（同一 exe 多次运行之间）
- **根因 A（不同运行方式之间，设计使然）**：浏览器直接打开 `index.html`（file:// 或固定 dev 端口）与 exe 版本（内嵌 HTTP 服务）是**不同的“源”**，IndexedDB 天然隔离，历史记录互不通用——这不属于 bug。
- **根因 B（真正的 bug：同一 exe 多次运行也读不到）**：`web.view` 通过 `wsock.tcp.simpleHttpServer` 提供 `/web/...` 页面，该服务器**默认每次启动分配一个随机空闲端口**（49152~65535）。而 IndexedDB 以“源（协议+主机+**端口**）”为隔离边界，端口一变源就变，记录就被写入各自独立的库；重新打开加载的是另一个空源，于是历史记录“消失”。磁盘上可见一堆 `http_127.0.0.1_<不同端口>.indexeddb.leveldb` 即为证。
- **修复**：在 `wb.go()` **之前**固定 `wsock.tcp.simpleHttpServer.startPort`（先探测端口可用再设置，占用则退回自动分配）：
  ```aardio
  var FIXED_PORT = 52417;
  try {
      var testSrv = wsock.tcp.simpleHttpServer("127.0.0.1", FIXED_PORT);
      if (testSrv) { testSrv.stop(); wsock.tcp.simpleHttpServer.startPort = FIXED_PORT; }
      else { wsock.tcp.simpleHttpServer.startPort = 0; }
  } catch(e) { wsock.tcp.simpleHttpServer.startPort = 0; }
  ```
  固定后源稳定为 `http://127.0.0.1:52417`，IndexedDB 库随之稳定。
- **验证**：exe 中新建事项 → 关闭 → 重新打开，"选择事项"下拉框应能看到历史记录；磁盘上 IndexedDB 目录应只有一个固定端口的库。

### ❌ 关闭窗口后程序不退出，卡死 / 报 “C stack overflow” / 显示 “未响应”
- **根因**：在 `winform.onClose` 里做了同步阻塞操作（`wb.closeAndWait()`、`wb.doScript()`、`thread.delay()`）。这些操作会阻塞主消息循环，导致：
  - `wb.closeAndWait()` 内部 `while(isWindow)` 死循环；
  - `wb.doScript()` 在关闭过程中同步等待 WebView2 响应，WebView2 不响应则主线程卡住；
  - `thread.delay()` 直接阻塞消息循环，Windows 将窗口标记为 **Not Responding**；
  - 最终进程只能强制结束。
- **修复**：
  1. `winform.onClose` 中不做任何同步清理：只设 `isClosing` 标志，返回 `null` 允许默认关闭流程。
  2. 真正的清理放在 `winform.onDestroy` 中：kill WebView2 子进程 → 使用 `process(curPid).terminate(0)` 结束当前 exe 主进程自身。
  3. **不要依赖 `::Kernel32.ExitProcess(0)`**：实测在该 aardio + WebView2 场景下，`ExitProcess(0)` 调用后主进程仍会从“应用”降级为“后台进程”继续驻留。
  4. **自杀不要用 `process(pid).kill()`**：aardio 的 `kill()` 会先 `suspend()` 再 `terminate()`，对自己调用会冻结当前线程，导致进程看起来“挂起/残留”。自杀应使用 `process(pid).terminate(0)` 直接调用 Windows `TerminateProcess`。
- **验证**：点击关闭按钮后 `gift-book_v1.1.1.exe` 进程应立即从任务管理器消失，窗口不显示“未响应”，无错误弹窗。

---

## 六、交付前验证清单

- [ ] exe 内嵌资源：二进制扫描含 `tailwindcss`/`pdf-lib`/`fontkit`/`xlsx`/`MaShanZheng`/`NotoSansSC`/`guest-screen.html`/`电子礼簿` 等标记。
- [ ] 图标：pefile 查 `RT_ICON` 共 9 张，尺寸/字节与 `icon.ico` 逐张匹配。
- [ ] 版本：右键 exe → 属性 → 详细信息，**文件版本 = 产品版本 = 1.1.1.0**。
- [ ] 体积：约 15MB（全内嵌 /web 的必然体积，非外置瘦身）。
- [ ] 单实例：再次双击 exe 只激活已有窗口，不启动第二个实例。
- [ ] 进程清理：关闭窗口后任务管理器中无残留 `msedgewebview2.exe`。
- [ ] 主进程退出：关闭窗口后任务管理器中无残留 `gift-book_v1.1.1.exe`。
- [ ] 正常退出：点击关闭按钮后程序立即退出，不卡顿、不报 “C stack overflow”。
- [ ] 数据持久化：exe 内新建事项并关闭，重新打开后能看到历史记录。
- [ ] 运行：Win10/11 双击即开；确认本机已装 WebView2 运行时。

---

## 七、下次打包时可直接粘贴给 AI 的「避坑须知」

> 用 aardio 把网页打成单文件 exe 时，请遵守：
> 1. **`.aproj` 必须平铺**：只用一个 `<folder name="web" embed="true">`，其下直接列出全部 `<file>`（含 `web\xxx`），**严禁把 static 等再套嵌套 `<folder>`**（否则 F7 报 271c）。
> 2. 每个 `<file path>` 写全路径（反斜杠）：`web\guest-screen.html`、`web\static\xxx`，不能只写文件名。
> 3. `main.aardio` 保持 `wb.go("/web/index.html")`，不要改入口。
> 4. **aardio 没有无头编译**：别试 `aardio.exe /build`，必须假设在有图形界面的机器上由我（或你）打开工程按 F7。
> 5. 重建 `aardio-project/` 时务必写到**真实磁盘**（早期它只存在于沙箱视图、真实磁盘不存在，导致无法复现构建）。
> 6. 生成 `.aproj` 后，先脚本校验所有 `path` 在磁盘存在（`missing=NONE`）再交付构建。
> 7. 程序图标：`<project>` 加 `icon="\icon.ico"`（前导反斜杠，工程根相对），`icon.ico` 与 `.aproj` 同目录；图标是 PE 资源，勿列入 embed 清单。
> 8. F7 完成框**别勾「版本号自增」**，否则下个版本会变 1.1.2。
> 9. 打包后图标"看着没变"大概率是 **Windows 资源管理器图标缓存**，用 pefile 查 `RT_ICON` 是否 9 张匹配来判定，别急着改 `icon` 路径。
> 10. **`winform.onClose` 里【禁止】做任何同步阻塞操作**：包括 `wb.closeAndWait()`（内部 `while` 死循环）、`wb.doScript()`（关闭时 WebView2 不响应会卡死）、`thread.delay()`（阻塞消息循环 → Not Responding）。`onClose` 只应设置标志并返回。
> 11. **关闭退出要用“onClose 放行 + onDestroy 强制结束进程”**：`onClose` 返回 `null` 走默认关闭流程；`onDestroy` 中先 kill WebView2 子进程，再用 `process(curPid).terminate(0)` 结束当前 exe 主进程自身。**不要依赖 `::Kernel32.ExitProcess(0)`**——实测在该场景下无法结束 aardio 主进程；**也不要用 `process(curPid).kill()`**——其内部先 `suspend()` 再 `terminate()`，对自己调用会冻结当前线程、进程仍残留。
> 12. **`main.aardio` 必须做单实例互斥体**（`process.mutex`）：已存在则前置窗口；若找不到窗口，则 kill 残留的无窗口同名进程（动态取当前 exe 名）后再启动。
> 13. **`index.html` 必须在 `beforeunload` 中关闭 IndexedDB 连接**（`dbManager.db.close()`），让浏览器默认关闭流程触发 beforeunload，不要在 aardio 中同步触发。
> 14. **🔴 致命坑：`web.view` 的 `wsock.tcp.simpleHttpServer` 默认每次启动分配随机端口，导致 IndexedDB 源（含端口）每次都变、记录跨运行“消失”**。必须在 `wb.go()` 之前固定 `wsock.tcp.simpleHttpServer.startPort`（先 `simpleHttpServer("127.0.0.1",端口)` 探测可用、`stop()` 释放、再设置 `startPort`；占用则退回 `0` 自动）。磁盘上出现多个 `http_127.0.0.1_<不同端口>.indexeddb.leveldb` 即为此坑的实证。
