# 1. 业务全景概览 (Overview)

## 简介
这个工具 (SafariBooks) 的核心价值是帮助用户将 **Safari Books Online** 平台上的书籍内容（文本、图片、样式）下载到本地，并重新打包成通用的 **EPUB 电子书格式**。

对于产品经理而言，可以将这个工具理解为一个**自动化内容抓取与格式转换引擎**。它模拟用户在浏览器中的行为，自动化地完成“登录 -> 查找 -> 阅读 -> 保存”这一系列繁琐的操作。

---

## 用户旅程 (User Journey)

下面的流程图展示了用户如何使用该工具，以及工具在后台的主要响应逻辑。

```mermaid
graph TD
    %% 节点样式定义
    classDef user fill:#f9f,stroke:#333,stroke-width:2px;
    classDef system fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;
    classDef external fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;

    User((用户 User)):::user
    System[SafariBooks 工具]:::system
    Safari[Safari Books Online 平台]:::external

    subgraph "准备阶段 (Setup)"
        User -->|1. 获取账号| Safari
        User -->|2. 获取书籍ID| Safari
        Safari -.->|返回书籍ID URL| User
    end

    subgraph "核心流程 (Core Process)"
        User -->|3. 运行命令 + 凭证 + 书籍ID| System

        System -->|4. 登录验证| Safari
        Safari -.->|验证通过 (Cookies)| System

        System -->|5. 获取书籍元数据 (Info)| Safari
        Safari -.->|返回书名/作者/目录| System

        System -->|6. 循环下载章节| Safari
        Safari -.->|返回 HTML 内容| System

        System -->|7. 解析 & 本地化资源 (CSS/Images)| System
        System -->|8. 并发下载资源| Safari
    end

    subgraph "产出阶段 (Output)"
        System -->|9. 打包 EPUB| EPUBf[书籍文件 .epub]
        EPUBf --> User
    end
```

## 核心价值点 (Value Proposition)

1.  **自动化 (Automation)**:
    - 自动处理登录鉴权 (Authentication)。
    - 自动遍历书籍的所有章节 (Pagination & Traversal)。
    - 自动下载并关联书中的图片和样式表 (Resource Management)。

2.  **格式标准化 (Standardization)**:
    - 将网页 HTML 转换为符合 EPUB 标准的 XHTML。
    - 生成标准的 `content.opf` (资源清单) 和 `toc.ncx` (目录结构)。
    - **Kindle 兼容性**: 提供了 `--no-kindle` 选项，专门处理 CSS 样式以适配 Kindle 阅读器的渲染特性（如表格和代码块的溢出处理）。

3.  **断点续传 (Resumability)**:
    - 智能检测已下载的章节和资源，避免重复下载，节省时间和带宽。
    - 即使程序意外中断，再次运行也能接着下载。

## 关键业务实体 (Key Entities)

在代码逻辑中，我们主要操作以下几个核心实体：

| 实体 | 描述 | 对应技术概念 |
| :--- | :--- | :--- |
| **凭证 (Credential)** | 用户的邮箱和密码，用于获取访问权限。 | Session, Cookies |
| **书籍 ID (Book ID)** | 唯一标识一本书的数字代码。 | REST API Parameter |
| **章节 (Chapter)** | 书籍的基本组成单元，通常对应一个网页。 | HTML Page |
| **资源 (Asset)** | 书籍中引用的图片和样式表。 | CSS, Images |
| **EPUB 包** | 最终交付给用户的单一文件。 | Zip Archive |

---
*下一篇文档将深入分析核心业务逻辑的实现细节。*
