# 🛡️ XSS 跨站脚本攻击学习笔记

> **PortSwigger Web Security Academy 实战记录**
> 
> 学员：Dean | 时间：2026年6月  
> 靶场环境：Kali Linux 2026.1 + Burp Suite CE 2024.2.1

![XSS](https://img.shields.io/badge/XSS-Learning-brightgreen) ![Status](https://img.shields.io/badge/Status-3%2FComplete-blue) ![Platform](https://img.shields.io/badge/Platform-PortSwigger-orange)

---

## 📋 目录

- [一、什么是 XSS](#一什么是-xss)
- [二、靶场环境搭建](#二靶场环境搭建)
- [三、Lab 1: 反射型 XSS — HTML 上下文无过滤](#三lab-1-反射型-xss--html-上下文无过滤)
- [四、Lab 2: 反射型 XSS — 属性注入绕过尖括号过滤](#四lab-2-反射型-xss--属性注入绕过尖括号过滤)
- [五、Lab 3: 存储型 XSS — 评论区 HTML 上下文无过滤](#五lab-3-存储型-xss--评论区-html-上下文无过滤)
- [六、反射型 vs 存储型 XSS 对比](#六反射型-vs-存储型-xss-对比)
- [七、核心技能与工具总结](#七核心技能与工具总结)
- [八、安全红线与法律须知](#八安全红线与法律须知)

---

## 一、什么是 XSS

**XSS**（Cross-Site Scripting，跨站脚本攻击）是一种将恶意 JavaScript 代码注入到网页中的攻击方式。当其他用户访问被注入的页面时，恶意代码会在他们的浏览器中执行，从而**窃取 Cookie、劫持会话、伪造请求、钓鱼**等。

### XSS 的三种类型

| 类型 | 说明 |
|------|------|
| **反射型 XSS** | 恶意脚本通过 URL 参数或表单提交"反射"回页面并执行。攻击者需要诱骗受害者点击特制链接。通常出现在搜索结果、错误提示等即时回显场景。 |
| **存储型 XSS** | 恶意脚本被永久保存在目标服务器上（如数据库、留言板、用户资料），每次用户访问受影响的页面时都会自动执行。比反射型 XSS 危险得多，不需要点击特定链接。 |
| **DOM 型 XSS** | 漏洞存在于客户端 JavaScript 代码中，恶意脚本通过修改 DOM 来执行，不需要与服务器交互。通常出现在使用 `innerHTML`、`document.write` 等不安全 API 的页面。 |

### 核心攻击原理

> XSS 的本质是：**网站把用户的输入当作代码执行了。**

正常情况下，用户输入的 `<script>` 应该被当作普通文字显示。但如果网站没有做过滤和转义，浏览器就会把它当成 HTML 标签来解析和执行。

---

## 二、靶场环境搭建

### 虚拟机配置

| 项目 | 配置 |
|------|------|
| 系统 | Kali Linux 2026.1 虚拟机 |
| 虚拟化软件 | VirtualBox |
| 配置 | 4 GB 内存，2 核 CPU |
| 虚拟机位置 | `E:\VMs\Kali\` |
| 默认登录 | `kali / kali` |

### 核心工具

| 工具 | 版本 | 说明 |
|------|------|------|
| **Burp Suite** | Community Edition 2024.2.1 | 官方 JAR 包，代理设置 `127.0.0.1:8080` |
| **Firefox ESR** | Kali 自带 | 配合 Burp 代理使用 |
| **Kali 软件源** | 清华镜像 | `mirrors.tuna.tsinghua.edu.cn/kali` |

### 练习平台

> 🟢 [PortSwigger Web Security Academy](https://portswigger.net/web-security)（在线靶场，无需本地搭建）

---

## 三、Lab 1: 反射型 XSS — HTML 上下文无过滤

| 信息 | 内容 |
|------|------|
| 难度 | APPRENTICE（学徒）|
| 类型 | Reflected XSS |
| 状态 | ✅ **SOLVED** |

### Lab 名称

**Reflected XSS into HTML context with nothing encoded**  
（反射型 XSS 注入 HTML 上下文 — 无任何编码/过滤）

### 漏洞原理

网站搜索功能将用户输入直接嵌入到 HTML 响应页面中，没有做任何转义或过滤。输入的 `<script>` 标签被浏览器当作代码执行。

### 攻击步骤

1. 进入 Lab 靶场，找到搜索框（`Search the blog...`）
2. 在搜索框直接输入攻击载荷
3. 点击 `Search` 按钮提交
4. 页面弹出 `alert(1)` 弹窗 → 攻击成功

### 攻击载荷 (Payload)

```html
<script>alert(1)</script>
```

### 注入后的页面 HTML

```html
<h1>Search results for: <script>alert(1)</script></h1>
```

### 为什么能成功？

网站没有对用户输入做任何处理。尖括号 `<>` 没有被转义，`<script>` 标签被浏览器完整解析并执行。

### 如何防御？

- **方法 1**：对输出到 HTML 的内容做 HTML 实体编码  
  `<` → `&lt;` | `>` → `&gt;` | `"` → `&quot;`
- **方法 2**：使用模板引擎的自动转义功能（如 Jinja2 的 `{{ }}`）
- **方法 3**：设置 Content-Security-Policy（CSP）HTTP 头

---

## 四、Lab 2: 反射型 XSS — 属性注入绕过尖括号过滤

| 信息 | 内容 |
|------|------|
| 难度 | APPRENTICE（学徒）|
| 类型 | Reflected XSS (attribute) |
| 状态 | ✅ **SOLVED** |

### Lab 名称

**Reflected XSS into attribute with angle brackets HTML-encoded**  
（反射型 XSS 注入 HTML 属性 — 尖括号被 HTML 编码）

### 漏洞原理

用户输入被嵌入到 `<input>` 标签的 `value` 属性中，且尖括号 `<>` 被做了 HTML 实体编码。但双引号没有被过滤，攻击者可以通过**闭合属性**来注入事件处理器。

### 侦查：分析 HTML 结构

先用 `test` 填充，用 F12 查看生成的 HTML：

```html
<input type="text" placeholder="Search the blog..." name="search" value="test">
```

输入在 `value=""` 属性里 → 需要闭合属性来注入。

### 攻击步骤

1. 按 `F12` 打开开发者工具（侦查）
2. 在搜索框输入 `test`，用 Inspect Element 找到 `<input>` 元素
3. 确认输入在 `value=""` 属性中
4. 构造 payload 闭合属性并注入 `onmouseover` 事件
5. 搜索框输入: `" onmouseover="alert(1)`
6. 提交后，鼠标移到搜索框上 → 触发 `alert(1)`

### 攻击载荷 (Payload)

```html
" onmouseover="alert(1)
```

### 注入后的 HTML（模拟）

```html
<input type="text" ... value="" onmouseover="alert(1)">
```

### 为什么 `<script>` 不行？

输入 `<script>alert(1)</script>` 后变成：

```html
<input value="&lt;script&gt;alert(1)&lt;/script&gt;">
```

尖括号被编码了，不会当标签执行。但双引号没被过滤，用 `"` 闭合属性后即可注入新属性。

### 关键技能：属性注入

当标签无法使用时：

- 用 `"` 或 `'` 闭合当前属性
- 用空格分隔，注入新的 HTML 属性
- 使用事件处理器：`onmouseover`, `onclick`, `onfocus`, `onload` 等
- Payload 模式: `" on事件="恶意代码`

### 如何防御？

- 除了 HTML 实体编码外，也要对属性值中的引号做编码：`"` → `&quot;` | `'` → `&#39;`
- 对属性值做严格的输入校验（只允许数字和字母）

---

## 五、Lab 3: 存储型 XSS — 评论区 HTML 上下文无过滤

| 信息 | 内容 |
|------|------|
| 难度 | APPRENTICE（学徒）|
| 类型 | Stored XSS |
| 状态 | ✅ **SOLVED** |

### Lab 名称

**Stored XSS into HTML context with nothing encoded**  
（存储型 XSS 注入 HTML 上下文 — 无编码/过滤）

### 漏洞原理

评论区的输入被服务器保存到数据库中，每次有人访问博客文章，恶意脚本都会在访问者浏览器中执行。攻击者**不需要发送钓鱼链接**——只要帖子存在，所有访客都自动中招。

### 攻击步骤

1. 进入博客文章详情页（点 `View post`）
2. 滚动到底部找到 `Leave a comment` 评论区
3. 在 Comment 框输入载荷，Name 和 Email 随便填
4. 在 Comment 框输入: `<script>alert(1)</script>`
5. 点 `Post Comment` 提交
6. 页面刷新后自动弹出 `alert(1)` → **所有访客都会触发**

### 攻击载荷 (Payload)

```html
<script>alert(1)</script>
```

### 真实场景影响

存储型 XSS 在真实网站中的典型利用方式：

| 攻击方式 | 说明 |
|----------|------|
| **Cookie 窃取** | 窃取 Cookie → 劫持会话 → 冒充用户登录 |
| **钓鱼** | 注入登录表单 → 用户输入账号密码 → 发送到攻击者服务器 |
| **伪造请求** | 调用 API 修改用户设置、发帖、转账等 |
| **网页篡改** | 页面被篡改，影响网站声誉，用户流失 |

### 常见存储型 XSS 注入点

- 论坛帖子 / 博客评论区
- 用户个人资料页（昵称、签名、自我介绍）
- 产品评论和评分
- 即时通讯消息
- 上传文件的文件名

### 如何防御？

- **输出编码**：所有用户提交的内容存储和显示时都必须做 HTML 实体编码
- **输入过滤**：白名单过滤，只允许安全标签
- **HttpOnly**：Cookie 设置 HttpOnly 标志，防止 JS 读取
- **CSP**：Content-Security-Policy 头限制可执行的脚本来源

---

## 六、反射型 vs 存储型 XSS 对比

| 对比维度 | 反射型 XSS | 存储型 XSS |
|----------|-----------|-----------|
| Payload 存储位置 | 不存储（仅存在于 URL/请求中）| 存储在服务器数据库 |
| 触发方式 | 需要用户点击特制链接 | 所有访问者自动触发 |
| 影响范围 | 单个受害者 | 所有访问页面的人 |
| 危险等级 | 中危 | **高危** |
| 常见场景 | 搜索框、错误提示、URL参数回显 | 评论区、留言板、个人资料、论坛 |
| 漏洞发现难度 | 较容易 | 需找到数据保存并回显的入口 |
| SRC 奖金范围 | ¥500 ~ ¥2,000 | ¥2,000 ~ ¥10,000+ |

---

## 七、核心技能与工具总结

### 已掌握的攻击技术

- ✅ 直接注入 `<script>` 标签（无过滤场景）
- ✅ HTML 属性闭合注入（用 `"` 或 `'` 闭合 value 属性）
- ✅ 事件处理器注入（`onmouseover`, `onfocus`, `onclick` 等）
- ✅ 用 F12 开发者工具 Inspect Element 分析注入点

### XSS Payload 速查

```html
<!-- 直接执行 -->
<script>alert(1)</script>

<!-- 属性闭合 + 事件注入 -->
" onmouseover="alert(1)
' onclick='alert(1)

<!-- img 标签 onerror -->
<img src=x onerror=alert(1)>

<!-- a 标签 -->
<a href="javascript:alert(1)">click</a>
```

### 已使用的工具

| 工具 | 用途 |
|------|------|
| Kali Linux 2026.1 | 攻击机操作系统 |
| Burp Suite CE 2024.2.1 | 代理拦截、流量分析 |
| Firefox ESR（F12）| Inspect Element 分析注入点 |
| PortSwigger Academy | 在线靶场练习 |

### 后续要学的工具

| 工具 | 用途 |
|------|------|
| Burp Repeater | 重复发送/修改请求 |
| Burp Intruder | 批量 fuzz payload |
| Burp Decoder | 编码/解码 URL、Base64、HTML Entity |
| Nuclei | 基于模板的自动化漏洞扫描 |
| Python `requests` 库 | 写自动化 PoC 脚本 |

---

## 八、安全红线与法律须知

> ⚠️ **以下三条红线绝对不能触碰！作为大学生，法律风险意识是白帽的第一素质。**

### 红线 1: 只在授权范围内测试

| 行为 | 是否可做 |
|------|---------|
| PortSwigger Academy Lab（官方提供的练习靶场）| ✅ 可以 |
| 自己搭建的 DVWA、Pikachu 等本地靶场 | ✅ 可以 |
| 补天/漏洞盒子 SRC 明确 scope 内的目标 | ✅ 可以 |
| 任何未经授权的网站（包括学校的选课系统、朋友的项目）| ❌ 不能 |
| 百度、淘宝、京东等商业网站（除非有明确授权）| ❌ 不能 |

### 红线 2: 不要用扫描器扫外部站点

Burp Intruder、SQLMap 等工具只能在靶场和 SRC 授权范围内使用。扫描外部站点会被对方服务器记录 IP，可能导致报警或法律追责。

### 红线 3: 拿到数据立刻停手

挖到漏洞后，**验证漏洞存在即可停止**。绝对不要：

- ❌ 翻数据库、下载用户数据
- ❌ 留后门、二次访问
- ❌ 未经授权公开漏洞细节

### 法律依据

- 📖 《网络安全法》第 27 条：不得从事危害网络安全的活动
- 📖 《刑法》第 285 条：非法侵入计算机信息系统罪
- 📖 《刑法》第 286 条：破坏计算机信息系统罪

> **黄金法则：靶场里随便炸，靶场外碰都不要碰。**

---

## 📈 学习进度

```
已完成: 3/???
✅ Lab 1: 反射型 XSS (HTML上下文无过滤)
✅ Lab 2: 反射型 XSS (属性注入)
✅ Lab 3: 存储型 XSS (评论区)
🔜 Lab 4-?: DOM型 XSS、SVG注入、Angular注入...
```

---

## 👨‍💻 作者

**Dean（麦麦提尼牙孜·艾合买提）**  
新疆能源职业技术学院 · 机电一体化技术 · 大三  
📧 2524593811@qq.com  
🔗 [github.com/Scott214278](https://github.com/Scott214278)

---

*笔记持续更新中，后续会补充 DOM 型 XSS、SVG 注入、过滤器绕过等更多内容。*
