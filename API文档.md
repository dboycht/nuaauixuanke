# 教务系统选课 API 接口文档

> 南京航空航天大学教务系统（金智教育 eams）自动选课接口逆向分析
> 适用于 `Xuanke_v2.py` 的选课功能

---

## 目录

- [一、整体架构](#一整体架构)
- [二、请求头](#二请求头)
- [三、前置预热](#三前置预热)
- [四、拉取课程列表](#四拉取课程列表)
- [五、批量选课提交（核心）](#五批量选课提交核心)
- [六、响应判断](#六响应判断)
- [七、批量一包优化](#七批量一包优化)
- [八、代码位置索引](#八代码位置索引)

---

## 一、整体架构

```
拉取课程列表        批量选课提交
    │                   │
    ▼                   ▼
GET data.action    POST batchOperator.action
```

| 阶段 | 接口 | 方法 | 作用 |
|------|------|------|------|
| 建立会话 | `/eams/homeExt.action` | GET | 登录态验证 |
| 预热页面 | `/eams/stdElectCourse!defaultPage.action` | GET | 获取合法 Referer |
| 拉取课程 | `/eams/stdElectCourse!data.action` | GET | 获取可选课程列表 |
| 提交选课 | `/eams/stdElectCourse!batchOperator.action` | POST | 执行选课（支持多门） |

> 这是**金智教育（Wisedu）eams 教务系统**的标准接口，NUAA 采用该系统。

---

## 二、请求头

所有请求必须携带以下请求头（来源：`Xuanke_v2.py` 的 `_make_eams_session`）：

```python
headers = {
    "User-Agent": "Mozilla/5.0 ... Chrome/120.0.0.0 Safari/537.36",
    "Cookie": "<登录后的整行 Cookie>",
    "Accept": "application/json, text/javascript, */*; q=0.01",
    "Accept-Encoding": "gzip, deflate, br",
    "Origin": "https://aao-eas.nuaa.edu.cn",
    "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
    "Referer": "https://aao-eas.nuaa.edu.cn/eams/stdElectCourse!defaultPage.action?electionProfile.id=<pid>",
    "X-Requested-With": "XMLHttpRequest",   # ← 关键，标记为 AJAX 请求
}
```

> ⚠️ **`Referer` 和 `X-Requested-With` 缺一不可**
>
> - `Referer` 是选课页面的"来源"，学校网关会校验是否来自合法的选课页面
> - `X-Requested-With: XMLHttpRequest` 标记这是 XHR 请求

---

## 三、前置预热

提交选课前，必须先发送两个 GET 请求：

```
1. GET /eams/homeExt.action
   → 建立会话，确认登录态有效

2. GET /eams/stdElectCourse!defaultPage.action?electionProfile.id=4665
   → 预热选课页面，拿到与提交请求匹配的 Referer
```

### Referer 的匹配逻辑

```python
# 先访问 defaultPage，然后把它作为后续请求的 Referer
referer = f"{BASE}/eams/stdElectCourse!defaultPage.action?electionProfile.id={pid}"
s.get(referer)                    # 预热
s.headers["Referer"] = referer    # 用于后续 data.action 和 batchOperator.action
```

---

## 四、拉取课程列表

### 请求

```
GET https://aao-eas.nuaa.edu.cn/eams/stdElectCourse!data.action?electionProfile.id=4665
```

### 参数

| 参数 | 说明 |
|------|------|
| `electionProfile.id` | 选课档案 ID（网址末尾的数字）|
| `profileId` | 备选参数名（部分系统使用）|

### 参数名自动适配

代码会自动尝试两种参数名：

```python
url1 = f"{BASE}/eams/stdElectCourse!data.action?electionProfile.id={pid}"
url2 = f"{BASE}/eams/stdElectCourse!data.action?profileId={pid}"
```

### 返回格式

返回 JS 对象数组（类似 JSON），每条课程形如：

```js
{
  id: 400785,                    // 课程 ID
  name: '高等数学(一)',           // 课程名称
  code: 'AE101',                  // 课程代码
  credit: '4.0',                  // 学分
  ...
}
```

### 解析方式

代码用正则提取：

```python
find_id   = re.compile(r"id:(\d+),")
find_name = re.compile(r"name:'([^']*)',")

id_list   = find_id.findall(text)
# 通过 text.split("code:") 分段后逐段提取 name
```

---

## 五、批量选课提交（核心）

### 请求

```
POST https://aao-eas.nuaa.edu.cn/eams/stdElectCourse!batchOperator.action?electionProfile.id=4665
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
```

### 参数结构

```python
data = {
    "optype": "true",
    "operator0": "400785:true:0",   # 课程ID:是否选课:标志位
    "lesson0":   "400785",
}
```

### 参数含义

| 参数 | 值 | 含义 |
|------|-----|------|
| `optype` | `"true"` | 操作类型标记，固定为 `true`（执行选课动作）|
| `operator0` | `"课程ID:true:0"` | 第一个操作对象：**课程ID** + 冒号 + **是否选课(`true`)** + 冒号 + **标志位(`0`)** |
| `lesson0` | 课程ID | 教学班/课堂 ID（通常与课程 ID 相同）|

### 🔑 关键：这就是"批量"的精髓

`batchOperator` 支持**一个请求里提交多门课程**——通过递增的下标 `0, 1, 2...`：

```python
# 一次提交 3 门课
data = {
    "optype": "true",
    "operator0": "400785:true:0",   # 第 1 门
    "lesson0":   "400785",
    "operator1": "398319:true:0",   # 第 2 门
    "lesson1":   "398319",
    "operator2": "398891:true:0",   # 第 3 门
    "lesson2":   "398891",
}
```

### 当前实现 vs 接口最大能力

| 方式 | 优点 | 缺点 |
|------|------|------|
| 逐门提交（当前实现）| 单门失败不影响其他；结果可单独统计 | 请求数多，受提交间隔限制 |
| 批量一包（接口支持）| 一次到位，理论上更快 | 一门出问题整包失败；无法单独判断哪门成功 |

---

## 六、响应判断

返回体是 JSON，用中文关键词判断结果：

| 返回内容 | 含义 | 代码处理 |
|----------|------|----------|
| 含 `"成功"` | 选课成功 | ✅ 绿色日志 |
| 含 `"请不要过快点击"` 或 429/503 | 触发限速 | 🟡 退避 3 秒 |
| 含 `"失败"` / `"错误"` / ≥400 | 选课失败 | 🔴 红色日志 |

```python
if "成功" in msg or "选课成功" in msg:
    # 成功
elif "请不要过快点击" in body or resp.status_code in (429, 503):
    # 限速 → 退避
elif "失败" in msg or "错误" in msg or resp.status_code >= 400:
    # 失败
```

---

## 七、批量一包优化

如需将多门课程打包为一次请求：

```python
data = {"optype": "true"}
for i, cid in enumerate(course_ids):
    data[f"operator{i}"] = f"{cid}:true:0"
    data[f"lesson{i}"] = cid

resp = session.post(url, data=data, ...)
```

一次请求搞定所有课程。**建议保留当前的逐门提交作为稳定方案**，批量一包可作为备选策略。

---

## 八、代码位置索引

| 内容 | 位置 |
|------|------|
| 请求头构建 + 前置预热 | `_make_eams_session()` |
| 课程列表拉取 | `FetchCoursesWorker.run()` / `_try_fetch_data()` |
| 提交 URL 构建（双参数名） | `GrabWorker._do_run()` |
| 提交参数构造 | `GrabWorker._do_run()` |
| 响应判断 | `GrabWorker._do_run()` |
| 课程解析正则 | `FetchCoursesWorker.run()` |

---

*仅供学习交流使用。*
