# 🌿 Celery 速成指南

> 内容整理自 [Celery 5.6 官方文档](https://docs.celeryq.dev/en/stable/)，版权归原作者所有，基于 [BSD License](http://www.opensource.org/licenses/BSD-3-Clause) 开源。本页仅供学习参考，不作商业用途。

![Celery](https://img.shields.io/badge/Celery-5.6-37a76f?style=flat-square&logo=celery)
![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-BSD_3--Clause-blue?style=flat-square)

从零开始，**10 分钟**内跑起你的第一个异步任务。

---

## 目录

1. [Celery 是什么？](#1-celery-是什么)
2. [核心概念](#2-核心概念)
3. [第一步：启动 Broker](#3-第一步启动-broker)
4. [第二步：安装 Celery](#4-第二步安装-celery)
5. [第三步：创建任务](#5-第三步创建任务)
6. [第四步：启动 Worker](#6-第四步启动-worker)
7. [第五步：调用任务](#7-第五步调用任务)
8. [第六步：查看结果](#8-第六步查看结果)
9. [基础配置](#9-基础配置)
10. [常见问题](#10-常见问题)
11. [下一步](#11-下一步)

---

## 1. Celery 是什么？

**Celery** 是 Python 生态中最流行的**分布式任务队列**框架。它能让你把耗时的工作（发邮件、处理图片、调用第三方 API……）**从主进程剥离出去**，丢给后台 Worker 异步执行，主程序立刻返回，用户不用傻等。

| 解决什么问题 | 说明 |
|---|---|
| 🚀 非阻塞 | 慢任务不卡主线程 / Web 请求 |
| 📡 分布式 | 任务分发给多台机器并行处理 |
| ⏰ 定时 | 周期性执行任务，类似 cron |
| 🔁 重试 | 任务失败自动重试，可配置策略 |

> Celery 是 Python 写的，但协议可被 Node.js、Go、Rust 等语言的客户端使用。

---

## 2. 核心概念

只需理解三个角色：

```
你的应用（Producer）
      │
      │  发送消息
      ▼
消息中间件（Broker）   ←──  RabbitMQ / Redis
      │
      │  分发任务
      ▼
后台进程（Worker）  ──→  执行任务 ──→  结果存储（Backend，可选）
```

| 角色 | 是什么 | 常见实现 |
|---|---|---|
| **Producer** | 触发任务的一方（你的代码） | 任意 Python 进程 |
| **Broker** | 存放待执行消息的队列 | RabbitMQ、Redis |
| **Worker** | 真正执行任务的后台进程 | `celery worker` |
| **Result Backend** | 存储任务执行结果（可选） | Redis、RPC、数据库 |

---

## 3. 第一步：启动 Broker

Broker 是 Celery 的"邮局"，必须先有它才能传递任务消息。

### 推荐新手：Redis（最省事）

一个命令，同时搞定 Broker + Result Backend：

```bash
docker run -d -p 6379:6379 redis
```

不用 Docker 也可以：
- **Ubuntu/Debian**：`sudo apt-get install redis-server`
- **macOS**：`brew install redis && redis-server`

### 生产环境推荐：RabbitMQ

```bash
docker run -d -p 5672:5672 rabbitmq
```

> 💡 **新手建议**：先用 Redis 把流程跑通，再考虑切换 RabbitMQ。

---

## 4. 第二步：安装 Celery

先创建虚拟环境（强烈建议）：

```bash
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
```

安装 Celery，含 Redis 支持：

```bash
pip install "celery[redis]"
```

用 RabbitMQ 的话：

```bash
pip install celery
```

---

## 5. 第三步：创建任务

新建文件 `tasks.py`：

```python
from celery import Celery

# 创建 Celery 实例
# broker  = 消息从哪来
# backend = 结果存哪里（可选，但推荐配置）
app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/0',
)

# @app.task 把普通函数变成可异步调用的 Celery 任务
@app.task
def add(x, y):
    return x + y
```

就这样，`@app.task` 装饰器告诉 Celery：*"这个函数可以被异步调用"*。

> 如果用 RabbitMQ，把 `broker` 改为 `amqp://guest@localhost//`

---

## 6. 第四步：启动 Worker

打开一个**新终端窗口**，在 `tasks.py` 同目录下运行：

```bash
celery -A tasks worker --loglevel=INFO
```

| 参数 | 含义 |
|---|---|
| `-A tasks` | 指定 Celery 应用所在模块（`tasks.py`） |
| `worker` | 启动 Worker 子命令 |
| `--loglevel=INFO` | 显示详细日志，方便调试 |

看到下面的输出，说明 Worker 启动成功：

```
[tasks]
  . tasks.add

[INFO/MainProcess] celery@hostname ready.
```

> ⚠️ Worker 终端要**一直开着**。每当有任务进来，这里就会打印执行日志。

---

## 7. 第五步：调用任务

再开一个终端，进入 Python 交互式环境：

```python
from tasks import add

# .delay() 是最常用的异步调用方式，立即返回
result = add.delay(4, 4)
# <AsyncResult: 550e8400-e29b-41d4-a716-446655440000>
```

切换到 Worker 终端，你会看到：

```
[INFO] Received task: tasks.add[550e8400...]
[INFO] Task tasks.add[550e8400...] succeeded in 0.001s: 8
```

### 三种调用方式对比

| 方法 | 说明 | 适用场景 |
|---|---|---|
| `add.delay(4, 4)` | 最简洁，异步发出立即返回 | 日常使用 |
| `add.apply_async((4, 4))` | 完整版，支持更多参数 | 需要设倒计时、优先级等 |
| `add(4, 4)` | 直接调用，**不经过队列** | 本地测试 |

### `apply_async` 进阶示例

```python
# 10 秒后才执行
add.apply_async((4, 4), countdown=10)

# 指定在某个时刻执行
from datetime import datetime, timezone
add.apply_async((4, 4), eta=datetime(2025, 1, 1, tzinfo=timezone.utc))

# 指定队列
add.apply_async((4, 4), queue='high-priority')
```

---

## 8. 第六步：查看结果

只要配置了 `backend`，就能追踪任务状态和结果：

```python
result = add.delay(4, 4)

result.ready()        # False → True（是否已完成）
result.get(timeout=5) # 8（获取返回值，会阻塞等待）
result.status         # 'SUCCESS'

# 出错时，不抛异常，安全获取
result.get(propagate=False)
result.traceback      # 查看完整错误堆栈
```

### 任务状态一览

| 状态 | 含义 |
|---|---|
| `PENDING` | 等待中（还没被 Worker 接收） |
| `STARTED` | Worker 已开始执行 |
| `SUCCESS` | ✅ 执行成功 |
| `FAILURE` | ❌ 执行失败，有异常 |
| `RETRY` | 🔁 正在重试 |
| `REVOKED` | 已被撤销 |

> ⚠️ **注意**：每个 `AsyncResult` 用完后，务必调用 `result.get()` 或 `result.forget()`，否则结果会在 Backend 里累积占用空间。

---

## 9. 基础配置

### 内联配置

```python
app.conf.update(
    task_serializer='json',
    result_serializer='json',
    accept_content=['json'],
    timezone='Asia/Shanghai',
    enable_utc=True,
)
```

### 推荐：独立配置文件 `celeryconfig.py`

```python
# celeryconfig.py
broker_url        = 'redis://localhost:6379/0'
result_backend    = 'redis://localhost:6379/0'

task_serializer   = 'json'
result_serializer = 'json'
accept_content    = ['json']

timezone          = 'Asia/Shanghai'  # 中国时区
enable_utc        = True

task_acks_late    = True  # 任务执行完再 ACK，防止 Worker 崩溃丢任务
```

在 `tasks.py` 中加载：

```python
app.config_from_object('celeryconfig')
```

> 💡 **时区建议**：设成 `Asia/Shanghai` + `enable_utc=True`，内部用 UTC，显示用本地时间，避免夏令时问题。

### 任务路由（进阶）

```python
# celeryconfig.py —— 把特定任务发到专属队列
task_routes = {
    'tasks.add': {'queue': 'low-priority'},
}
```

---

## 10. 常见问题

<details>
<summary><b>❌ 任务一直是 PENDING 状态</b></summary>

- 检查 `tasks.py` 里是否配置了 `backend`
- 确认 Worker 和 Producer 连的是**同一个** Broker 地址
- 用以下命令确认 Worker 是否收到任务：

```bash
celery -A tasks inspect active
```

</details>

<details>
<summary><b>❌ 连接 Broker 失败</b></summary>

```bash
# 检查 Redis 是否在跑
redis-cli ping
# 返回 PONG 才正常
```

没有 PONG：
```bash
docker start <容器名>
# 或
redis-server
```

</details>

<details>
<summary><b>❌ Windows 下 Worker 报错 / 无法启动</b></summary>

Celery 在 Windows 上对多进程有限制，本地调试加 `--pool=solo`：

```bash
celery -A tasks worker --loglevel=INFO --pool=solo
```

</details>

<details>
<summary><b>❌ Worker 启动报 Permission Error（Linux）</b></summary>

Debian/Ubuntu 可能限制了 RabbitMQ 相关权限，建议改用 Redis，或以 `sudo` 启动 RabbitMQ。

</details>

---

## 11. 下一步

掌握基础后，推荐按顺序探索：

| 主题 | 说明 | 文档链接 |
|---|---|---|
| 📋 **Next Steps** | 任务链、Canvas 工作流入门 | [→ 阅读](https://docs.celeryq.dev/en/stable/getting-started/next-steps.html) |
| ⏰ **Periodic Tasks** | 定时任务（celery beat） | [→ 阅读](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html) |
| 🗺️ **Canvas** | chain / group / chord 复杂流程 | [→ 阅读](https://docs.celeryq.dev/en/stable/userguide/canvas.html) |
| 🌐 **Django 集成** | 与 Django 配合使用 | [→ 阅读](https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html) |
| 👁️ **Flower 监控** | Web 界面实时监控任务和 Worker | [→ 阅读](https://docs.celeryq.dev/en/stable/userguide/monitoring.html) |
| 🔧 **Tasks 深入** | 重试策略、信号、路由 | [→ 阅读](https://docs.celeryq.dev/en/stable/userguide/tasks.html) |

---

## 参考来源

- 📖 [Celery 官方文档](https://docs.celeryq.dev/en/stable/) — Celery 5.6
- 📖 [First Steps with Celery](https://docs.celeryq.dev/en/stable/getting-started/first-steps-with-celery.html)
- 📖 [Introduction to Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html)

> 本文档内容蒸馏整理自以上官方资料，版权归 [Celery 项目](https://github.com/celery/celery) 所有，基于 BSD 3-Clause License 开源。
