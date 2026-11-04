# Bright Data 爬虫工作室 (Node.js)

一个精简的 Node.js 入门项目，用于通过数据采集 API 运行 [Bright Data 爬虫工作室](https://www.bright.cn/products/web-scraper/studio)采集器：使用 URL 列表触发任务并下载结果。

[![Bright Data 推广](https://github.com/bright-cn/Awesome-Web-Scraping/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner%20CN.png)](https://www.bright.cn/)

<a href="https://githubbox.com/bright-cn/bright-data-scraper-studio-nodejs-project?file=index.js" target="_blank">在 CodeSandbox 中打开</a>，使用 GitHub 登录，然后复刻该代码仓库即可开始修改。

---

## 目录

- [概述](#概述)
- [功能](#功能)
- [前提条件](#前提条件)
- [安装](#安装)
- [使用方法](#使用方法)
- [配置](#配置)
- [工作原理](#工作原理)
- [示例](#示例)
- [输出](#输出)
- [安全](#安全)
- [支持](#支持)
- [许可证](#许可证)

---

## 概述

[Bright Data 爬虫工作室](https://www.bright.cn/products/web-scraper/studio)是一个低代码 IDE，用于在 Bright Data 平台上构建自定义网页爬虫采集器。采集器发布后，会提供两个 HTTP 端点：

| 步骤 | 端点 | 用途 |
| --- | --- | --- |
| 1 | `POST /dca/trigger?collector=<id>` | 将一个或多个输入加入采集器队列 |
| 2 | `GET  /dca/dataset?id=<snapshot_id>` | 数据准备就绪后下载采集的数据 |

此代码仓库使用约 150 行 Node.js 代码封装了这两个调用，方便你复制、粘贴并投入使用。

---

## 功能

- 通过 `/dca/trigger` 端点触发爬虫工作室采集器
- 轮询 `/dca/dataset`，直到结果准备就绪
- 通过 `.env` 使用环境变量进行配置（代码中不包含密钥）
- 对临时错误（5xx 和网络错误）使用指数退避重试；遇到 4xx 错误时立即失败
- 库辅助函数：`triggerWithUrl`、`triggerWithUrls`、`runScraper`
- 将原始 JSON 响应保存到带时间戳的文件中
- ES 模块，Node 18+

---

## 前提条件

- Node.js v18 或更高版本
- 拥有 [API 令牌](https://www.bright.cn/cp/setting)的 Bright Data 账户
- 在[爬虫工作室](https://www.bright.cn/cp/scrapers)中拥有已发布的采集器；复制其 **采集器 ID**（以 `c_` 开头）

---

## 安装

```sh
git clone https://github.com/bright-cn/bright-data-scraper-studio-nodejs-project.git
cd bright-data-scraper-studio-nodejs-project
npm install
cp .env.example .env       # then edit .env with your token and collector ID
```

---

## 使用方法

```sh
npm start
# or
node index.js
```

结果将写入项目目录中的 `scraper_studio_results_<timestamp>.json` 文件。

---

## 配置

需要两个环境变量。你可以在 `.env`、shell 中设置它们，也可以将其硬编码到 [`index.js`](index.js) 中：

| 变量 | 获取位置 |
| --- | --- |
| `BRIGHT_DATA_API_TOKEN`    | Bright Data 控制面板，[账户设置 &rarr; API 令牌](https://www.bright.cn/cp/setting) |
| `BRIGHT_DATA_COLLECTOR_ID` | 爬虫工作室：打开你的采集器，从 URL 中复制 ID（以 `c_` 开头） |

你还可以在 `index.js` 顶部调整轮询和重试行为：

```js
const POLL_INTERVAL_MS  = 5000;  // delay between dataset checks
const MAX_POLL_ATTEMPTS = 60;    // give up after ~5 minutes
const MAX_RETRIES       = 3;     // for transient HTTP failures
```

`SAMPLE_URLS` 的结构必须与你在爬虫工作室中定义的**输入架构**一致。默认示例假定只有一个 `url` 字段。如果你的采集器使用不同的输入（例如 `keyword`、`zip_code`、`category`），请相应更新对象。

---

## 工作原理

```text
       +-----------------+      POST /dca/trigger      +-------------------+
       |  Your script    | --------------------------> |  Scraper Studio   |
       |  (index.js)     | <-- { collection_id } ----- |  Collector        |
       +-----------------+                             +-------------------+
                |                                                |
                |  GET /dca/dataset?id=<snapshot_id>             |
                |  (poll every 5s, retry 5xx with backoff)       |
                |  <--- [ { ...record... }, ... ] -------------- |
                v
       scraper_studio_results_<timestamp>.json
```

该脚本每五秒轮询一次 `/dca/dataset`，最长持续五分钟。非空 JSON 数组会被视为已完成的快照。临时错误（5xx 和网络错误）会使用指数退避重试（1 秒、2 秒、4 秒）；遇到 4xx 错误时会立即失败，以便你修复请求，而不是继续重试。

---

## 示例

### 使用你自己的 URL 运行

替换 [`index.js`](index.js) 中的 `SAMPLE_URLS`：

```js
const SAMPLE_URLS = [
    { url: 'https://example.com/product/1' },
    { url: 'https://example.com/product/2' }
];
```

### 自定义输入架构

如果你的采集器需要 `url` 以外的内容，请传入其定义的相应字段：

```js
const inputs = [
    { keyword: 'wireless headphones', country: 'US' },
    { keyword: 'standing desk',       country: 'DE' }
];
await runScraper(inputs);
```

### 作为库使用

`runScraper`、`triggerWithUrl`、`triggerWithUrls` 和 `saveResults` 均已导出：

```js
import { triggerWithUrls, saveResults } from './index.js';

const data = await triggerWithUrls([
    'https://example.com/page-1',
    'https://example.com/page-2'
]);
saveResults(data, 'my_run.json');
```

---

## 输出

- 结果将保存为名为 `scraper_studio_results_<ISO timestamp>.json` 的 JSON 文件。
- 该文件包含采集器的原始输出：默认情况下，每个输入 URL 对应一条记录。

---

## 安全

切勿提交你的 `.env` 文件。随附的 `.gitignore` 会阻止提交 `.env` 和 `.env.local`。

如果你不慎提交了真实的 `BRIGHT_DATA_API_TOKEN`：

1. 立即前往 [www.bright.cn/cp/setting](https://www.bright.cn/cp/setting) 轮换令牌。
2. 使用 [`git filter-repo`](https://github.com/newren/git-filter-repo) 或 [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) 从历史记录中移除密钥。
3. 强制推送，并通知所有可能已克隆泄露内容的人员。

---

## 支持

- [Bright Data 帮助中心](https://brightdata.com/help)
- [爬虫工作室快速入门](https://docs.brightdata.com/api-reference/scraper-studio-api/Getting_started_with_the_API)
- [联系支持团队](https://brightdata.com/contact-us)

---

## 许可证

本项目采用 MIT 许可证。详情请参阅 [LICENSE](LICENSE)。
