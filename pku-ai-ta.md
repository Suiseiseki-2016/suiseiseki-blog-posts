---
title: 用 LLM 批改作业：PKU AI 助教的实现
slug: pku-ai-ta
summary: 记录为北京大学算法课构建 AI 助教的全过程——从逆向 Blackboard 插件爬取作业，到用 LLM 视觉模型批改扫描版 PDF，再到自动提交成绩。
category: AI, 工程
published_at: 2026-03-19
---

# 用 LLM 批改作业：PKU AI 助教的实现

课程助教最繁琐的工作之一是批改作业：几十份 PDF，逐一打开，对照评分标准，手动录入成绩。对于算法课这类有明确标准答案的课程，这件事几乎是纯体力劳动。

于是我写了一个工具，把这个流程自动化：爬取提交、LLM 批改、人工复核、一键提交成绩。本文记录其中遇到的工程细节。

---

## 一、目标与架构

整个流程分四步：

```
爬取提交 → LLM 打分 → 人工复核（Excel）→ 提交成绩
```

技术选型：
- **爬虫**：`httpx` + 逆向 Blackboard 插件接口
- **LLM**：通过 OpenRouter 调用 `qwen/qwen3.5-397b-a17b`，支持文本和图像
- **批改结果导出**：`openpyxl` 生成 Excel，可疑项高亮黄色
- **成绩提交**：模拟浏览器表单 POST

---

## 二、爬取作业：逆向 Blackboard 插件

北京大学的课程系统基于 Blackboard Learn，但标准的 BB REST API 对助教角色**不开放提交文件内容**。学生作业实际由一个名为 `bb-homeWorkCheck-BBLEARN` 的自研插件托管。

好在这个插件的 HTTP 接口并没有特别的保护，只需要有效的登录 Session 即可访问。

### 认证：IAAA SSO + RSA 加密

2025 年底起，PKU IAAA 统一认证系统要求对密码进行 RSA-PKCS1v15 加密后再提交。流程如下：

```python
# 1. 获取公钥
GET /iaaa/getPublicKey.do

# 2. 用公钥加密密码
from cryptography.hazmat.primitives.asymmetric import padding
encrypted = public_key.encrypt(password.encode(), padding.PKCS1v15())

# 3. 以 appid="blackboard" 登录（不是 course.pku.edu.cn）
POST /iaaa/oauthlogin.do  →  token

# 4. 用 token 换取 BB Session Cookie
GET /B_CAMPUS_LOGIN_URL?token=...
```

这里有两个坑：`appid` 必须是 `"blackboard"` 而不是域名，接口是 `/iaaa/oauthlogin.do` 而不是 `/oauth.jsp`。

另外，PKU 的 CA 证书不在 macOS 信任链里，httpx 要加 `verify=False`。

### 学生列表：两种链接格式

`getStudentWork.do` 返回的 HTML 里，学生提交链接有两种格式并存：

```html
<!-- 老生（查看）-->
<a href="...CheckAloneWork.do?...userId=X&filePk=Y&attemptPk=Z">查看</a>

<!-- 新生（批改）-->
<a onclick="checkWork('userId','filePk','attemptPk')">批改</a>
```

两种格式都要解析，漏掉任何一种就会静默丢失一半的学生。

### 下载文件：双重 URL 编码

下载单个学生文件需要先访问 `CheckWork.do` 提取嵌入 JS 里的 `filePath`，再用双重 URL 编码请求 `api/pdf.do`：

```python
encoded = quote(quote(file_path, safe=""), safe="")
resp = client.get(f"{HW_BASE}/api/pdf.do?path={encoded}")
```

这里不能用 `httpx` 的 `params=` 参数传递——那会再编码一次变成三重编码，服务器会返回 404。

---

## 三、批改：LLM + 视觉模型

### 提示词设计

算法作业的批改有个核心原则：**不确定的地方不能扣分**。如果 LLM 看不清某部分是否正确，应该给满分并标记出来等人工复核，而不是猜测性扣分。

```
Grading philosophy:
- Default to FULL marks. Only deduct when you have clear, direct evidence.
- When in doubt — do NOT deduct. Award full marks and record in uncertain_parts.
- Every deduction MUST be accompanied by an explicit reason and exact points.
```

LLM 返回结构化 JSON：

```json
{
  "total_score": 98,
  "total_max": 100,
  "confidence": 0.85,
  "breakdown": [
    {
      "criterion": "1.6 矩阵乘法加法次数",
      "points_awarded": 8,
      "points_max": 12,
      "reasoning": "deduct 4 pts: 未给出加法操作的计数分析"
    }
  ],
  "uncertain_parts": [...],
  "llm_reasoning": "..."
}
```

`confidence < 0.75` 或存在 `uncertain_parts` 的提交会在 Excel 里高亮为黄色，等待人工复核。

### 处理扫描版 PDF

很多同学用"扫描全能王"之类的 App 扫描手写作业再上传，pypdf 从这类 PDF 提取到的是水印文字而非作业内容：

```
扫描全能王 创建
```

这种情况下必须走视觉路径。判断逻辑：

```python
def _needs_vision(text: str) -> bool:
    if any(marker in text for marker in _UNREADABLE_MARKERS):
        return True
    return len(text.strip()) < 200  # 水印或空白
```

对于扫描 PDF，用 pymupdf 直接提取 PDF 内嵌的原始 JPEG/PNG 图像（不重新渲染，保留扫描原始质量）：

```python
for page in doc:
    for img_info in page.get_images(full=True):
        xref = img_info[0]
        base_image = doc.extract_image(xref)
        # base_image["image"] 是原始 JPEG/PNG 字节
```

如果 PDF 没有内嵌图像（少数情况），才退回到 `page.get_pixmap()` 渲染。

然后将图像以 base64 编码的 `image_url` content part 发给模型：

```python
{
    "type": "image_url",
    "image_url": {"url": f"data:image/jpeg;base64,{b64}", "detail": "high"}
}
```

### JSON 鲁棒性

模型输出的 reasoning 字段里经常有 LaTeX 公式，比如 `\frac{n}{2}`，这些裸反斜杠会让 `json.loads` 报错。解决方案是在解析前转义：

```python
def _sanitize_json(s: str) -> str:
    # 转义非合法 JSON 转义序列的反斜杠
    return re.sub(r'\\(?!["\\/bfnrtu])', r'\\\\', s)
```

### 多线程加速

每次 LLM 调用要几秒到十几秒，顺序执行 40 份作业需要等很久。用 `ThreadPoolExecutor` 并发：

```python
with ThreadPoolExecutor(max_workers=settings.ta_threads) as executor:
    futures = {executor.submit(score_submission, sub, rubric_text): sub for sub in submissions}
    for future in as_completed(futures):
        all_results.append(future.result())
```

默认 4 线程，可通过 `TA_THREADS` 环境变量调整。

---

## 四、提交成绩：逆向 sendData()

成绩提交是最有意思的部分。我最初尝试用 Blackboard REST API：

```
PATCH /learn/api/public/v1/courses/{courseId}/gradebook/columns/{columnId}/users/{userId}
```

结果每次都返回 403：

```
此请求未与有效会话关联
```

REST API 不接受 Cookie 认证，只接受 OAuth Token。而插件本身有自己的提交表单——`CheckWork.do` 页面上有一个 `gradeAttemptForm`，提交按钮调用 `sendData()` 这个 JS 函数。

抓一下 `sendData()` 的源码：

```javascript
function sendData() {
    const inputVal = document.getElementById('inputBox').value;
    const richContent = document.getElementById('richContent').value;

    const xhr = new XMLHttpRequest();
    xhr.open('POST', 'saveStudentGrade.do', true);
    xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded;charset=UTF-8');

    xhr.send(
        'inputData=' + encodeURIComponent(inputVal) +
        '&attemptPk=' + encodeURIComponent(3385854) +
        '&gradeBookPk=' + encodeURIComponent(423829) +
        '&course_id=' + encodeURIComponent('_98024_1') +
        '&richContent=' + encodeURIComponent(richContent) +
        '&gradePk=' + encodeURIComponent(3220076)  // 每个学生不同！
    );
}
```

关键发现：`gradePk` 是一个**每个学生不同**的服务端注入值，存在于页面 JS 里。这意味着提交前必须为每个学生单独请求一次 `CheckWork.do` 页面来提取这个值：

```python
_GRADE_PK_RE = re.compile(r"gradePk=[^)]*encodeURIComponent\((\d+)\)")

def _fetch_grade_pk(client, course_id, grade_book_pk, user_id, file_pk, attempt_pk, title):
    resp = client.get(f"{HW_BASE}/CheckWork.do", params={...})
    m = _GRADE_PK_RE.search(resp.text)
    return m.group(1) if m else None
```

提取到 `gradePk` 后，POST 到 `saveStudentGrade.do` 即可完成提交。

---

## 五、效果

对第一次算法作业（100 分，8 道题）的测试：

- 约 **80%** 的作业被直接打分，无需人工介入
- **20%** 被标记为需要复核（扫描不清晰、模型不确定）
- 人工复核后，整批作业的批改+提交时间从约 3 小时缩短到 30 分钟

整个工具开源在 GitHub，感兴趣的同学可以参考。

---

## 附：如何找到 course_id 和 column

- **`--course`（课程 ID）**：打开课程任意页面，URL 里的 `course_id=_98024_1` 即是，包括下划线。
- **`--column`（作业 ID）**：在作业列表页点击"查看"，URL 里 `gradeBookPK=423829` 后面的纯数字即是。
