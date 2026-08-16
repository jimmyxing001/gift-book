# gift-book_v1.1.2.exe 运行异常分析与修复方案

## 问题 1：关闭窗口后 exe 主进程仍在后台运行，多实例冲突

### 根因
`main.aardio` 原代码在窗口关闭时**没有处理 `winform.onClose` 事件**，也没有显式释放 `web.view` 控件。直接点击关闭后：

- 窗口虽然看不见了，但 `gift-book_v1.1.1.exe` 主进程**没有真正退出**；
- WebView2 子进程（`msedgewebview2.exe`）也继续驻留；
- 再次双击 exe 时会启动新的主进程，任务管理器里出现多个 "电子礼簿 - 红白喜事礼金记账系统" 后台进程。

### 修复方案
在 `main.aardio` 中：

1. **增加单实例互斥体**（`process.mutex`）。启动时若检测到同名互斥体已存在，先尝试找到已有窗口并前置；
2. **如果找不到窗口**（说明是残留的无窗口进程），则枚举并 `kill()` 所有同名 `gift-book_v1.1.1.exe` 进程，然后重新创建互斥体并继续启动；
3. **在 `winform.onClose` 中执行完整清理后调用 `ExitProcess(0)`**：
   - 触发网页 `beforeunload`，给前端一个落盘机会；
   - 等待 500ms，让 IndexedDB flush；
   - 调用 `wb.closeAndWait()` 释放 WebView2；
   - 遍历 kill 本进程派生的 `msedgewebview2.exe` 子进程；
   - 最后调用 `::Kernel32.ExitProcess(0)` 立即结束当前 exe 主进程，确保不会残留。

### 关键代码（已写入 `aardio-project/main.aardio`）

```aardio
// 单实例 + 清理无窗口的残留进程
import process.mutex;
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

    // 找不到窗口：清理残留进程后再尝试启动
    try {
        var myPid = ::Kernel32.GetCurrentProcessId();
        for entry in process.each("gift-book_v1.1.1.exe") {
            if (entry.th32ProcessID != myPid) {
                var prcs = process(entry.th32ProcessID);
                if (prcs) { prcs.kill(); prcs.free(); }
            }
        }
    } catch(e) {}

    singleMutex.close();
    singleMutex = process.mutex("电子礼簿_SingleInstance_GiftBook_v1.1.1");
    if (singleMutex.conflict) {
        winform.msgbox("电子礼簿已经在运行中，且无法清理残留进程。", "电子礼簿");
        return;
    }
}

// 关闭时清理并强制退出进程
var isClosing = false;
winform.onClose = function(hwnd, message, wParam, lParam) {
    if (isClosing) return;
    isClosing = true;

    try { wb.doScript("window.dispatchEvent(new Event('beforeunload'));"); } catch(e) {}
    ..thread.delay(500);
    try { wb.closeAndWait(); } catch(e) {}
    try { killWebView2Children(); } catch(e) {}

    ::Kernel32.ExitProcess(0); // 关键：确保 exe 主进程彻底退出
    return false;
}
```

---

## 问题 2：exe 内创建的记录，再次打开后无历史记录

### 根因一：源程序与 exe 的存储路径天然不同
- **源程序**（浏览器直接打开 `gift-book/index.html`）使用浏览器默认 Profile 存储 IndexedDB，例如：
  - Edge: `%LocalAppData%\Microsoft\Edge\User Data\Default\IndexedDB\...`
  - Chrome: `%LocalAppData%\Google\Chrome\User Data\Default\IndexedDB\...`
- **exe 版本** 在 `main.aardio` 中显式指定了 `userDataDir`：
  ```aardio
  userDataDir = io.appData("/电子礼簿/webview2");
  ```
  `io.appData()` 对应系统 `%LocalAppData%`，因此实际路径为：
  ```
  C:\Users\<用户名>\AppData\Local\电子礼簿\webview2
  ```

**结论**：浏览器源程序与 exe 版本的 IndexedDB 位于完全不同的目录，彼此看不到对方的历史记录是**正常现象**。

### 根因二：exe 内部创建的记录也未持久化
即使只看 exe 自身，之前也存在“创建后重启丢失”的 bug，根本原因是：

- IndexedDB 的事务写入在内存中缓冲，需要页面/渲染进程正常关闭时才会 flush 到磁盘；
- 原 `main.aardio` 关闭窗口时直接退出，WebView2 子进程被强制终止，IndexedDB 连接未正常关闭，导致事务未提交、数据损坏或丢失。

### 修复方案
1. **在 `index.html` 中增加 `beforeunload` 处理**，主动关闭 `DBManager` 持有的 IndexedDB 连接，触发 flush：
   ```javascript
   window.addEventListener("beforeunload", () => {
     try {
       if (window.giftBookApp?.dbManager?.db) {
         window.giftBookApp.dbManager.db.close();
       }
     } catch (e) { console.warn("关闭 IndexedDB 时出错:", e); }
   });
   ```
2. **在 `main.aardio` 的 `onClose` 中触发该事件并等待释放**，与前端配合完成落盘。

### 数据存储位置总结
| 运行方式 | IndexedDB 数据库名 | 实际磁盘路径（Windows 10/11） |
|---|---|---|
| 浏览器直接打开 `index.html` | `GiftRegistryDB` | 浏览器 Profile 下的 `IndexedDB\https_<域名>_0.indexeddb.leveldb` |
| exe 版本 | `GiftRegistryDB` | `%LocalAppData%\电子礼簿\webview2\EBWebView\Default\IndexedDB\...` |

> 若需要把浏览器里的历史记录迁移到 exe 版本，必须手动把浏览器 Profile 下的 `IndexedDB` 目录复制到 exe 的 `userDataDir` 下，且由于 `origin` 不同（本地文件 vs 内嵌 http://aardio/... 域），WebView2 可能无法识别，**不推荐迁移**。

---

## 已修改的文件清单

| 文件 | 修改内容 |
|---|---|
| `aardio-project/main.aardio` | 新增单实例互斥体、`onClose` 事件（含 `isClosing` 防递归 + PostMessage 二次关闭）、WebView2 子进程清理 |
| `index.html` | 新增 `window.giftBookApp` 暴露与 `beforeunload` 关闭 IndexedDB 逻辑 |
| `aardio-project/web/index.html` | 与根目录 `index.html` 同步 |
| `aardio-project/gift-book.aproj` | 版本号统一为 `1.1.1.0` |

---

---

## 问题 3（最终根因）：关闭后 exe 主进程仍残留 / 卡死无法退出

### 根因（关键！）
**`winform.onClose` 里调用 `wb.closeAndWait()` 才是所有“关不掉”的真正元凶。**

查 aardio 源码 `lib/web/view/_.aardio` 的 `closeAndWait`：

```aardio
closeAndWait = function(){
    owner._form.close();                       // 关闭 WebView2 内部 chrome 子窗口
    while(..win.isWindow(owner.hwndChrome))    // 自旋等待子窗口销毁
        ..thread.delay(10);
    ..thread.delay(500);
};
```

`while(isWindow(hwndChrome))` 这个等待循环**要求父窗口（我们自己的 `winform`）先完成销毁**，子窗口才会被 WebView2 释放。但我们是在 `winform.onClose`（父窗口正在关闭的回调里）调用它的——**父窗口的销毁被我们的回调挡住了**，于是 `hwndChrome` 永远不销毁，`while` 死循环，`ExitProcess` 永远到不了，进程就卡死在后台。

这就解释了：
- 为什么“点击关闭后程序仍在后台运行无法退出”（死循环卡住）；
- 为什么之前加 `PostMessage` 二次关闭、`ExitProcess(0)` 都不管用——因为 `closeAndWait()` 在 `ExitProcess` 之前就把线程卡死了。

> 注：C stack overflow 是更早一版（清理后直接 `return false` 取消关闭，但 WebView2 已被释放）的现象，属于另一个分支问题；最终根因统一为 `closeAndWait()` 死循环。

### 最终修复
**彻底移除 `wb.closeAndWait()`**，改为非阻塞清理后立即 `ExitProcess(0)` 强制结束整个 exe 进程：

```aardio
// 显式声明，避免某些环境未解析内核 API
var ExitProcess = ::Kernel32.api("ExitProcess", "void(int)");

var isClosing = false;
winform.onClose = function(hwnd, message, wParam, lParam) {
    if (isClosing) return;            // 防重入
    isClosing = true;

    // 1) 触发网页 beforeunload，flush IndexedDB
    try { wb.doScript("window.dispatchEvent(new Event('beforeunload'));"); } catch(e) {}
    // 2) 给事务落盘留时间（不再调用 closeAndWait）
    ..thread.delay(400);
    // 3) 清理 WebView2 子进程
    killWebView2Children();
    // 4) 强制结束当前 exe 主进程
    try { ExitProcess(0); } catch(e) {}
    // 5) 兜底：万一 ExitProcess 未生效，允许默认关闭流程自然退出
    return;
}
```

要点：
- **绝不在 `winform.onClose` 里调用 `wb.closeAndWait()`**（其内部等待循环会在父窗口关闭回调中死锁）。
- 数据落盘改为由 `beforeunload` 中的 `db.close()` + 400ms 延迟保证，不依赖 `closeAndWait()`。
- `ExitProcess` 显式声明 API，并保留“允许默认关闭”兜底，即使强制退出意外失败，进程也能自然退出，不会再卡死残留。

### 启动清理残留进程
单实例冲突且找不到窗口时，枚举并 kill 本 exe 同名进程（动态取当前 exe 文件名，兼容改名），再重新创建互斥体继续启动。

---

## 问题 4（最终修复）：点击关闭后窗口“未响应”，exe 仍无法退出

### 根因
问题 3 的修复虽然去掉了 `wb.closeAndWait()`，但 `winform.onClose` 里仍然保留了：

```aardio
wb.doScript("window.dispatchEvent(new Event('beforeunload'));");
..thread.delay(400);
killWebView2Children();
ExitProcess(0);
```

这三行全都在**主消息线程里同步执行**：

- `wb.doScript` 在窗口关闭过程中同步调用 WebView2 执行脚本，WebView2 此时可能已不响应 `doScript` 的同步等待，导致主线程卡住；
- `thread.delay(400)` 虽然会分发消息，但 400ms 的阻塞足以让 Windows 把窗口标记为“未响应”；
- 一旦主线程被卡住，`ExitProcess(0)` 就永远执行不到，进程只能显示 **Not Responding**。

### 最终修复
把 `winform.onClose` 改成**只做两件事**：

1. 设置 `isClosing` 防重入；
2. 启动一个**后台线程**作为兜底，600ms 后无条件 `ExitProcess(0)`；
3. 返回 `null`，允许 aardio 走**默认窗口关闭流程**。

默认关闭流程会正常销毁窗口，WebView2 在销毁前会触发网页自己的 `beforeunload`，`index.html` 里的监听器会关闭 IndexedDB。窗口真正销毁时，`winform.onDestroy` 再主动 kill WebView2 子进程并立即 `ExitProcess(0)`。

如果 `onDestroy` 因任何异常没触发，后台兜底线程 600ms 后也会强制退出，确保进程绝不残留。

### 关键代码（已更新 `aardio-project/main.aardio`）

```aardio
var isClosing = false;
winform.onClose = function(hwnd, message, wParam, lParam) {
    if (isClosing) return false;
    isClosing = true;

    // 兜底线程：给浏览器留 600ms 处理 beforeunload/flush，然后强制退出
    thread.create(
        function(){
            import thread;
            thread.delay(600);
            ..Kernel32.ExitProcess(0);
        }
    );

    // 返回 null，允许默认关闭流程。浏览器会触发网页 beforeunload。
    return;
}

winform.onDestroy = function() {
    try { killWebView2Children(); } catch(e) {}
    try { ::Kernel32.ExitProcess(0); } catch(e) {}
}
```

### 核心原则
- **`winform.onClose` 里不要有任何同步阻塞操作**（`wb.doScript`、`thread.delay`、`closeAndWait` 都不行），否则会阻塞消息循环导致“未响应”。
- **数据落盘交给浏览器自己的 `beforeunload`**，aardio 只需要保证进程最终退出。
- **双保险**：`onDestroy` 正常退出 + 后台线程兜底退出，确保 100% 不残留。

---

## 已修改的文件清单

| 文件 | 修改内容 |
|---|---|
| `aardio-project/main.aardio` | 单实例互斥体 + 无窗口残留进程清理（动态取 exe 名）；`onClose` 只设标志并返回，让浏览器默认流程触发 `beforeunload`；`onDestroy` 中 kill WebView2 子进程并用 `process(curPid).kill()` 结束当前 exe 主进程；**移除 `ExitProcess` 与后台线程兜底（实测无效）** |
| `index.html` | 新增 `window.giftBookApp` 暴露与 `beforeunload` 关闭 IndexedDB 逻辑 |
| `aardio-project/web/index.html` | 与根目录 `index.html` 同步 |
| `aardio-project/gift-book.aproj` | 版本号统一为 `1.1.1.0` |

---

## 问题 5（最终验证修复）：`ExitProcess(0)` 调用后 exe 主进程仍残留

### 现象
加入诊断日志后，关闭窗口时的日志完整走到：

```
onClose 进入
onClose 返回 null（允许默认关闭）
onDestroy 进入
onDestroy 准备 kill 子进程
onDestroy kill 子进程完成
onDestroy 准备 ExitProcess
```

之后**没有**任何异常，但任务管理器里 `gift-book_v1.1.1.exe` 主进程从“应用”降级为“后台进程”，继续驻留。

### 根因
`::Kernel32.ExitProcess(0)`（以及显式 `::Kernel32.api("ExitProcess","void(int)")(0)`）在该场景下**没有真正结束 aardio 主进程**。具体原因未明，但两次实测结果一致：

- `onDestroy` 中调用 `ExitProcess(0)` 后进程不退出；
- 后台 `thread.create` 线程也没有日志输出，说明线程函数未实际执行或无法跨线程调用该 API。

而 `process(pid).kill()` 已经反复验证可以结束 WebView2 子进程，因此改用 `process(curPid).kill()` 结束当前 exe 主进程。

### 最终修复
`winform.onClose` 只做一件事：设置 `isClosing` 标志并返回，让浏览器默认关闭流程触发网页的 `beforeunload`。

`winform.onDestroy` 中：
1. 调用 `killWebView2Children()` 结束本进程派生的 `msedgewebview2.exe`；
2. 用 `process(curPid).kill()` 结束当前 exe 主进程自身。

```aardio
winform.onClose = function(hwnd, message, wParam, lParam) {
    if (isClosing) return false;
    isClosing = true;
    return; // 允许默认关闭流程，WebView2 会触发网页 beforeunload
}

winform.onDestroy = function() {
    try { killWebView2Children(); } catch(e) {}

    try {
        var curPid = ::Kernel32.GetCurrentProcessId();
        var selfPrcs = process(curPid);
        if (selfPrcs) {
            // 自杀不能用 kill()：kill() 内部先 suspend() 再 terminate()，
            // 对自己调用会冻结当前线程，导致无法真正终止。
            selfPrcs.terminate(0);
        }
    } catch(e) {}
}
```

### 关键结论
- **`wb.closeAndWait()` 必须移除**（在 `onClose` 中会死循环）。
- **`wb.doScript`、`thread.delay` 不要放在 `onClose`**（会阻塞消息循环，导致“未响应”）。
- **`ExitProcess` 在该 aardio + WebView2 场景下实测无法结束主进程**，不要再依赖它。
- **自杀必须用 `process(curPid).terminate(0)`**，不能用 `process(curPid).kill()`：
  - 查 aardio 源码，`kill()` 内部先调用 `suspend()` 再调用 `terminate()`；
  - 对自己调用 `suspend()` 会冻结当前进程的所有线程，导致后续 `terminate()` 无法执行；
  - 进程因此看起来“从应用降级为背景进程”，实际上是被挂起/冻结，无法退出。
- **`process(pid).terminate(0)` 直接调用 Windows `TerminateProcess`**，不先 suspend，可以正常结束当前 exe 主进程。

---

## 问题 5（新增）：exe 内创建记录，重新打开后历史记录消失

### 现象
用最新 exe 在界面里创建若干记录，关闭程序后再次打开，下拉框/列表里没有任何历史记录（数据“丢失”）。

### 根因（已用磁盘证据定位）
**不是权限问题，也不是 userDataDir 失效，而是「每次启动分配的 HTTP 端口不同，导致 IndexedDB 的“源”不同」。**

- `web.view` 通过 `wsock.tcp.simpleHttpServer` 把 `/web/...` 作为内嵌 HTTP 页面提供。
- 该服务器**默认每次启动分配一个随机空闲端口**（49152~65535，aardio 文档原文：“端口为0或省略则自动选择未用端口”）。
- 实测磁盘上 IndexedDB 目录确实每次都是新端口：
  ```
  ...\IndexedDB\http_127.0.0.1_49230.indexeddb.leveldb
  ...\IndexedDB\http_127.0.0.1_49248.indexeddb.leveldb
  ...\IndexedDB\http_127.0.0.1_49305.indexeddb.leveldb
  ...\IndexedDB\http_127.0.0.1_49743.indexeddb.leveldb  （每个数字都不同）
  ```
- **IndexedDB 以“源”（协议 + 主机 + 端口）为隔离边界**。端口一变，源就变；源一变，记录就被写入各自独立的库。重新打开时加载的是另一个（空）源，于是历史记录“消失”。
- `userDataDir = io.appData("/电子礼簿/webview2")` 本身每轮都一致，问题只出在端口（源）不固定。

> 说明：exe 与“浏览器直接打开源程序（file:// 或固定 dev 端口）”天然是两个不同源，历史记录互不通用——这是设计使然，并非本 bug。本 bug 是**同一个 exe 多次运行之间**源不统一。

### 修复方案
在 `wb.go("/web/index.html")` **之前**，固定 `wsock.tcp.simpleHttpServer.startPort`：

```aardio
var FIXED_PORT = 52417;
try {
    // 先探测端口是否可用，避免被占用时 wb.go 跳转空白页
    var testSrv = wsock.tcp.simpleHttpServer("127.0.0.1", FIXED_PORT);
    if (testSrv) {
        testSrv.stop();                       // 释放端口，交给 web.view 正式绑定
        wsock.tcp.simpleHttpServer.startPort = FIXED_PORT;
    } else {
        wsock.tcp.simpleHttpServer.startPort = 0; // 占用则退回自动分配
    }
} catch(e) {
    wsock.tcp.simpleHttpServer.startPort = 0;
}
wb.go("/web/index.html");
```

固定端口后，源稳定为 `http://127.0.0.1:52417`，IndexedDB 库也固定，记录可正确跨运行持久化与读取。

### 关键代码（已写入 `aardio-project/main.aardio`）
- 端口固定逻辑插入在 `wb.go()` 之前（见上）。

### 说明
- 已存在的若干 `http_127.0.0.1_<旧端口>.indexeddb.leveldb` 是此前各次随机端口产生的空/孤儿库，可手动删除，不影响新端口下的数据。
- 由于旧数据分散在互不相同的源中、且当初关闭后从未在新源里可见，实质已无法复用，需在新 exe 中重新创建一次。

---

## 重新打包步骤

1. 关闭当前 aardio 工程（如果已打开）。
2. 双击 `D:\WorkBuddy\gift-book\aardio-project\gift-book.aproj`。
3. 按 **F7** 发布，覆盖生成 `gift-book_v1.1.1.exe`。
4. 验证：
   - 双击一次运行后，任务管理器中不应再残留多个 `msedgewebview2.exe`；
   - 再次双击 exe，应自动激活已有窗口，而非启动第二个实例；
   - **点击关闭按钮后，任务管理器中 `gift-book_v1.1.1.exe` 和 `msedgewebview2.exe` 进程应立即消失，不残留**；
   - 在 exe 中新建事项并关闭，重新打开后应在“选择事项”下拉框看到历史记录。

---

## 验证数据是否落盘的方法

打包后运行 exe，新建一个事项并关闭程序，然后检查以下路径是否存在 LevelDB 数据文件：

```
%LocalAppData%\电子礼簿\webview2\EBWebView\Default\IndexedDB\
```

如果该目录下有 `http_127.0.0.1_52417.indexeddb.leveldb` 文件夹，且大小随创建记录增加，说明持久化已生效。

---

## 问题 6（修正）：固定端口后仍丢失历史记录

### 现象
在 `main.aardio` 中按“问题 5”的方案加了端口固定后，资源监控仍看到 `gift-book_v1.1.1.exe` 同时监听 **52417** 和另一个随机端口（如 62991）。exe 内创建记录后重新打开，依然没有历史记录。

### 根因（关键！）
**探测用的临时 `simpleHttpServer` 占用了 52417 且没有真正释放，导致 `web.view` 实际退到随机端口。**

之前的固定端口代码是：

```aardio
var testSrv = wsock.tcp.simpleHttpServer("127.0.0.1", FIXED_PORT);
if (testSrv) {
    testSrv.stop();                       // 意图：释放端口
    wsock.tcp.simpleHttpServer.startPort = FIXED_PORT;
}
```

- `testSrv` 是一个 `wsock.tcp.simpleHttpServer` 实例，创建后就会真正监听 52417；
- `testSrv.stop()` 未能立即/可靠地释放该端口（可能对象未析构、端口仍在监听，或处于 TIME_WAIT 仍被 OS 占用）；
- 随后 `wb.go("/web/index.html")` 内部调用 `wsock.tcp.simpleHttpServer.startSpaUrl()` 启动正式 HTTP 服务时，发现 52417 已被占用，自动退到另一个随机端口（如 62991）；
- 结果：资源监控里同时看到 52417（探测服务器残留）和 62991（实际页面服务），IndexedDB 实际写在 62991 的 origin 下，关闭重开后端口又变，历史记录继续“丢失”。

磁盘证据：

```
%LocalAppData%\电子礼簿\webview2\EBWebView\Default\IndexedDB\
    http_127.0.0.1_62991.indexeddb.leveldb   ← 最新写入
    http_127.0.0.1_60445.indexeddb.leveldb
    http_127.0.0.1_63693.indexeddb.leveldb
    ...（没有 http_127.0.0.1_52417.indexeddb.leveldb）
```

### 修正方案
**彻底移除探测服务器**，直接设置 `wsock.tcp.simpleHttpServer.startPort`，让 `web.view` 自己启动 simpleHttpServer 并绑定到固定端口。

```aardio
var FIXED_PORT = 52417;
wsock.tcp.simpleHttpServer.startPort = FIXED_PORT;
debugLog("set simpleHttpServer.startPort = " + FIXED_PORT);

// 页面初始化/加载完成后记录实际 URL，用于验证端口固定是否生效
wb.onDocumentInit = function(url) {
    debugLog("onDocumentInit url = " + (url : "null"));
}
wb.onDocumentComplete = function(url) {
    debugLog("onDocumentComplete url = " + (url : "null"));
    winform.setTimeout(function(){
        try {
            var origin = wb.eval("location.origin");
            var href = wb.eval("location.href");
            debugLog("page origin = " + (origin : "null") + ", href = " + (href : "null"));
        } catch(e) {
            debugLog("get origin error: " + e);
        }
    }, 500);
}

wb.go("/web/index.html");
```

同时在前端 `index.html` 加载成功后打印 origin：

```javascript
app.init().then(() => {
  ...
  console.log("页面 origin:", location.origin, "href:", location.href);
});
```

### 验证要点
1. 资源监控中应只看到 `gift-book_v1.1.1.exe` 监听 **52417**（WebView2 内部可能还有一个随机通信端口，这正常，不影响 IndexedDB origin）；
2. `%LocalAppData%\电子礼簿\webview2\EBWebView\Default\IndexedDB\` 下应出现并持续使用 `http_127.0.0.1_52417.indexeddb.leveldb`；
3. `close-debug.log` 中应记录到 `page origin = http://127.0.0.1:52417`；
4. 创建记录、关闭、重新打开后，历史记录应正常出现。

### 说明
- 旧的随机端口 `http_127.0.0.1_<旧端口>.indexeddb.leveldb` 是此前各次运行产生的孤儿库，可安全删除；
- 由于历史数据分散在不同 origin，无法自动合并，需在新 exe 中重新创建记录。
