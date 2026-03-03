# 生产工单看板系统 - Bug 修复

# 项目地址 
https://github.com/Apustar/interview_pro.git

## 前置准备

1. 复制源代码 `面试代码-app_buggy.py` 为 `app_buggy_fix.py`
2. 激活带 Flask 的开发环境
3. 运行 `python app_buggy_fix.py` 启动服务
4. 访问 http://127.0.0.1:5000 查看看板

## 概述

本文档记录了车间生产工单看板系统（app_buggy.py）中发现的全部 Bug 及修复方案。系统采用 Flask + SQLite 构建，提供工单列表查询、进度更新、汇总统计等核心功能。

---

## 问题 1：GET /api/orders 返回 500

### 1. 如何发现

运行 app_buggy_fix.py 后，在终端日志中可以看到：

```
127.0.0.1 - - [日期] "GET /api/orders HTTP/1.1" 500 -
```

访问首页时，工单列表区域一直显示「加载中...」，无法正常展示数据。

### 2. 如何排查

定位接口：首页会请求 /api/orders 获取工单列表，500 说明该接口内部出错。

查看调用链：get_orders() → 查询数据库 → format_order(r) 处理每一行。

定位异常：在 format_order 中调用 calculate_progress(row["quantity"], row["completed"])。

核对函数签名：calculate_progress(completed, quantity) 的参数顺序是「已完成，计划数量」。

发现问题：调用时传成了 (row["quantity"], row["completed"])，即「计划数量，已完成」，参数顺序反了。

触发异常：测试数据 WO-2026-003 的 completed=0，quantity=150，传入后变成 calculate_progress(150, 0)，即 150/0，引发 ZeroDivisionError，导致 500。

### 3. 如何修复

1. 修正参数顺序：calculate_progress(row["completed"], row["quantity"])
2. 避免除零：在 calculate_progress 中，当 quantity == 0 时直接返回 0.0

---

## 问题 2：整体完成率显示 10666.7%

### 1. 如何发现

前端界面「整体完成率」显示为 10666.7%，明显异常。

### 2. 如何排查

定位接口：/api/summary 接口的 overall_progress 字段计算逻辑。

查看代码（第 199-202 行）：

```python
summary['overall_progress'] = round(
    summary['total_completed'] / summary['total_orders'] * 100, 1
)
```

数据分析：总完成件数 = 120 + 200 + 0 = 320，工单总数 = 3，错误计算：320 / 3 × 100 = 10666.7%

发现问题：分母应为「总计划件数」而非「工单数量」

### 3. 如何修复

1. 将分母从 total_orders 改为 total_quantity
2. 增加分母为 0 的保护逻辑

```python
if summary['total_quantity'] > 0:
    summary['overall_progress'] = round(
        summary['total_completed'] / summary['total_quantity'] * 100, 1
    )
```

---

## 问题 3：进度条显示百分比不正确

### 1. 如何发现

前端进度条未正确显示百分比，视觉上显示异常。

### 2. 如何排查

定位模块：前端 JavaScript 第 306 行，进度条渲染代码。

问题代码：

```javascript
<div class="progress-bar" style="width: " + o.progress + "%"></div>
```

发现问题：模板字符串（反引号）内混用了字符串拼接语法 " + 变量 + "，导致变量未被正确解析。

### 3. 如何修复

使用模板字符串的正确语法 ${变量名}：

```javascript
<div class="progress-bar" style="width: ${o.progress}%"></div>
```

---

## 问题 4：更新进度无反应 + 415 错误

### 1. 如何发现

点击「更新」按钮后数据未更新，控制台报错：PUT /api/orders/1/progress HTTP/1.1" 415

### 2. 如何排查

Bug A - 415 错误：

定位：前端 fetch 请求（第 323-326 行）

根因：缺少 Content-Type: application/json 请求头，Flask 无法解析 JSON 请求体

Bug B - 数据未更新：

定位：后端 UPDATE 语句（第 158-161 行）

根因：缺少 conn.commit()，数据未提交到数据库

### 3. 如何修复

修复 A - 前端添加 Header：

```javascript
const res = await fetch('/api/orders/' + orderId + '/progress', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ completed })
});
```

修复 B - 后端添加 commit：

```python
conn.execute(
    "UPDATE work_orders SET completed=?, status=?, updated_at=? WHERE id=?",
    (new_completed, new_status, now, order_id)
)
conn.commit()  # 添加这行
```

---

## 问题 5：SQL 注入漏洞（严重）

### 1. 如何发现

代码审计发现 get_orders() 函数存在 SQL 注入风险。

### 2. 如何排查

定位问题：第 92-94 行

问题代码：

```python
rows = conn.execute(
    f"SELECT * FROM work_orders WHERE status = {status_filter}"
).fetchall()
```

根因：使用 f-string 直接拼接用户输入到 SQL 语句，攻击者可通过 ?status='; DROP TABLE work_orders; -- 执行任意 SQL。

### 3. 如何修复

使用参数化查询：

```python
rows = conn.execute(
    "SELECT * FROM work_orders WHERE status = ?", (status_filter,)
).fetchall()
```

---

## 问题 6：未使用的 import

### 1. 如何发现

代码审查发现导入了未使用的模块。

### 2. 根因

第 10 行 import json 未被使用。Flask 的 jsonify 和 request.get_json() 自带 JSON 处理。

### 3. 如何修复

删除第 10 行的 import json

---

## 问题 7：除零风险

### 1. 如何发现

calculate_progress 函数缺少对 quantity=0 的保护。

### 2. 根因

虽然数据库有 NOT NULL 约束，但业务上应进行防御性编程。

### 3. 如何修复

```python
def calculate_progress(completed, quantity):
    if quantity == 0:
        return 0.0
    return round(completed / quantity * 100, 1)
```

---

## 问题 8：类型检查不完善

### 1. 如何发现

update_progress 函数的类型检查存在漏洞。

### 2. 根因

request.get_json() 可能返回 None（无请求体）。isinstance(True, int) 返回 True，布尔值会被误接受。

### 3. 如何修复

```python
data = request.get_json()
if data is None:
    return jsonify({"success": False, "error": "请求体格式错误"}), 400

new_completed = data.get('completed')
if new_completed is None or not isinstance(new_completed, int) or isinstance(new_completed, bool) or new_completed < 0:
    return jsonify({"success": False, "error": "completed 必须为非负整数"}), 400
```

---

## 问题 9：静默异常（代码质量）

### 1. 如何发现

init_db 函数中使用空 except 块吞掉异常。

### 2. 根因

第 46-47 行：except sqlite3.IntegrityError: pass。重复插入数据时静默失败，不利于调试。

### 3. 建议修复

可保留当前逻辑（因重复插入是预期行为），或添加日志记录：

```python
except sqlite3.IntegrityError:
    pass  # 忽略重复插入，可选：添加日志记录
```

---

## 修复汇总表

| 序号 | 问题 | 严重程度 | 修复位置 |
|------|------|----------|----------|
| 1 | GET /api/orders 500（参数顺序错误） | 高 | 第 73 行 |
| 2 | 整体完成率显示异常 | 高 | 第 201 行 |
| 3 | 进度条显示不正确 | 中 | 第 306 行 |
| 4 | 更新无反应 + 415 错误 | 高 | 第 327 行、第 163 行 |
| 5 | SQL 注入漏洞 | 严重 | 第 92-94 行 |
| 6 | 未使用的 import | 低 | 第 10 行 |
| 7 | 除零风险 | 中 | 第 60-62 行 |
| 8 | 类型检查不完善 | 中 | 第 131-136 行 |
| 9 | 静默异常 | 低 | 第 46-47 行 |

---

## Git 提交建议

### 提交 1：修复核心功能 Bug

```
fix: 修复工单列表查询的 500 错误

- 修正 calculate_progress 参数顺序 (row["completed"], row["quantity"])
- 修正整体完成率计算逻辑，使用 total_quantity 作为分母
- 修复前端进度条模板字符串语法
- 添加 fetch 请求的 Content-Type header
- 后端 UPDATE 后添加 conn.commit() 提交事务
```

### 提交 2：修复安全漏洞

```
security: 修复 SQL 注入漏洞

- 将 f-string 拼接改为参数化查询
- 增强 update_progress 的类型检查
- 添加 calculate_progress 的除零保护
```

### 提交 3：代码优化

```
refactor: 清理未使用的 import

- 删除未使用的 json 模块导入
- 优化异常处理（可选）
```

---

## 修复后的文件

修复后的完整文件为 app_buggy_fix.py，可直接运行：

```bash
python app_buggy_fix.py
```

访问 http://127.0.0.1:5000 查看看板界面。