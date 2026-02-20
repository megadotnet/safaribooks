# 2. 核心逻辑与 API 交互 (Core Logic & API)

## 简介
本文深入探讨 `SafariBooks` 工具内部的运作机制。它通过模拟标准的用户浏览器行为，利用 Safari Books Online 的 REST API 获取书籍内容。

核心逻辑分为三个阶段：**认证 (Authentication)**、**内容抓取 (Crawling)**、**资源下载 (Assets Download)**。

---

## 阶段一：认证与元数据 (Auth & Metadata)

这是程序启动后的第一步，目的是确保我们有权限访问书籍，并知道这本书包含了多少个章节。

```mermaid
sequenceDiagram
    participant User as 用户 User
    participant Tool as SafariBooks Tool
    participant API as Safari Books API
    participant FS as 本地文件系统 (File System)

    User->>Tool: 启动命令 (Email:Pass, BookID)

    Tool->>FS: 检查 cookies.json?
    alt 存在 Cookies
        FS-->>Tool: 读取 Session
    else 无 Cookies
        Tool->>API: POST /member/auth/login/ (Email, Pass)
        API-->>Tool: 200 OK (Set-Cookie)
        Tool->>FS: 保存 cookies.json
    end

    Note right of Tool: 开始获取书籍信息

    Tool->>API: GET /api/v1/book/{BookID}/
    API-->>Tool: JSON {title, authors, cover_url, ...}
    Tool->>User: 显示书籍基本信息

    Note right of Tool: 获取章节列表 (分页)

    loop 直到没有下一页 (Pagination)
        Tool->>API: GET /api/v1/book/{BookID}/chapter/?page=X
        API-->>Tool: JSON {results: [chapters], next: url}
        Tool->>Tool: 追加章节到列表
    end

    Tool->>FS: 创建书籍目录 (Books/{Title})
```

### 关键点分析
1.  **Session 持久化**: 登录成功后，Cookies 会被保存到 `cookies.json`。下次运行时，如果 Cookies 未过期，可以直接跳过登录步骤，提升体验。
2.  **递归分页 (Recursive Pagination)**: 书籍章节列表通常很大，API 会分页返回。程序会自动请求下一页，直到获取所有章节。

---

## 阶段二：内容抓取与处理 (Content Crawling)

拿到章节列表后，程序开始逐个下载章节内容（HTML）。这一步不仅是下载，还包括了重要的**解析与清洗 (Parsing & Cleaning)** 工作。

```mermaid
stateDiagram-v2
    [*] --> 初始化队列
    初始化队列 --> 获取下一章节: Pop Chapter
    获取下一章节 --> 检查本地文件: File Exists?

    state 检查本地文件 {
        direction LR
        是 --> 跳过下载
        否 --> 下载HTML
    }

    下载HTML --> 解析HTML: lxml.html.fromstring
    解析HTML --> 提取资源链接: Find CSS & Images
    提取资源链接 --> URL重写: Rewrite Links
    URL重写 --> 注入样式: Inject Custom CSS
    注入样式 --> 保存XHTML: Save as .xhtml

    保存XHTML --> 获取下一章节
    跳过下载 --> 获取下一章节

    获取下一章节 --> [*]: 队列为空
```

### 关键逻辑：HTML 转换 (HTML Transformation)
原始网页的 HTML 不能直接用于 EPUB。我们需要做以下转换：
1.  **链接本地化 (Localization)**: 将网页中的图片链接（如 `https://safari.../image.png`）替换为相对路径（如 `Images/image.png`），以便离线查看。
2.  **样式注入 (CSS Injection)**:
    - 针对 Kindle 设备，移除可能导致显示异常的 `overflow: hidden` 等样式 (`--no-kindle` 选项)。
    - 注入基础排版样式。
3.  **资源收集**: 在解析 HTML 时，程序会“顺手”把遇到的 CSS 和图片 URL 放入待下载队列。

---

## 阶段三：并发资源下载 (Concurrent Assets Download)

为了加快速度，图片和 CSS 文件的下载是并发进行的。

```mermaid
graph LR
    subgraph "主线程 (Main Thread)"
        P[解析器 Parser] -->|发现图片 URL| Q_IMG[图片队列 Image Queue]
        P -->|发现 CSS URL| Q_CSS[样式队列 CSS Queue]
    end

    subgraph "下载工作进程 (Worker Processes)"
        Q_IMG --> W1[Worker 1]
        Q_IMG --> W2[Worker 2]
        Q_CSS --> W3[Worker 3]

        W1 -->|HTTP GET| API[Safari CDN]
        W2 -->|HTTP GET| API
        W3 -->|HTTP GET| API

        W1 -->|Save| FS_IMG[Images/ 目录]
        W2 -->|Save| FS_IMG
        W3 -->|Save| FS_CSS[Styles/ 目录]
    end
```

### 性能优化点
- **多进程/多线程**: 利用 `multiprocessing` (Linux/Mac) 或多线程 (Windows) 并行下载资源，显著减少等待时间。
- **去重 (Deduplication)**: 即使多章引用了同一张图片，程序也会检查是否已下载，避免重复请求。

---
*下一篇文档将展示这些数据如何被组装成最终的 EPUB 文件。*
