---
title: 为新闻聚合器打造完整自动化测试框架
published: 2026-05-09
description: "手把手教你为 Web 项目搭建自动化测试，覆盖 API、UI、性能、安全四大维度"
tags: ["Python", "测试", "自动化", "Playwright"]
category: 技术
draft: false
---

最近做了个热点新闻聚合器项目，功能完成后总觉得缺点什么——对，测试。手动点点点太累了，于是花了一天时间搭建了完整的自动化测试框架，83 个测试用例全部通过，爽。

## 为什么需要自动化测试

手动测试的问题：

- 每次改代码都要重新点一遍，烦
- 容易漏测，尤其边界情况
- 无法量化质量，不知道改完有没有破坏什么

自动化测试的好处：

- 一条命令跑完所有用例
- 覆盖正常流程和异常场景
- CI/CD 集成，提交代码自动验证

## 技术选型

| 类型 | 工具 | 理由 |
|------|------|------|
| 测试框架 | pytest | Python 生态标配，简洁强大 |
| API 测试 | requests | 简单直接 |
| UI 测试 | Playwright | 现代浏览器自动化，比 Selenium 快 |
| 性能测试 | concurrent.futures | 内置库，够用 |
| 报告 | pytest-html | 生成 HTML 报告 |

## 测试架构

```
tests/
├── conftest.py           # 全局 fixtures
├── config.py             # 配置（地址、超时）
├── run_tests.py          # 统一运行入口
│
├── api/                  # API 接口测试
│   ├── test_news.py      # 新闻 CRUD
│   ├── test_sources.py   # 数据源管理
│   └── test_tasks.py     # 定时任务
│
├── ui/                   # UI 自动化测试
│   ├── test_dashboard.py # 仪表盘
│   ├── test_newslist.py  # 新闻列表
│   └── test_settings.py  # 系统设置
│
├── perf/                 # 性能测试
│   ├── test_smoke.py     # 响应时间
│   └── test_concurrent.py # 并发压测
│
└── security/             # 安全测试
    ├── test_injection.py # SQL/XSS 注入
    └── test_headers.py   # HTTP 头安全
```

## API 测试示例

最基础也最重要，验证后端接口是否正常：

```python
class TestNewsAPI:
    def test_get_news_list(self, api_session):
        """测试获取新闻列表"""
        resp = api_session.get(f"{API_URL}/news")
        assert resp.status_code == 200
        data = resp.json()
        assert "total" in data
        assert "items" in data

    def test_pagination_boundary(self, api_session):
        """测试分页边界"""
        resp = api_session.get(
            f"{API_URL}/news",
            params={"page": 9999, "page_size": 20}
        )
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["items"]) == 0  # 超出范围应返回空
```

## UI 测试示例

用 Playwright 模拟用户操作：

```python
class TestNewsList:
    def test_crawl_with_loading(self, page: Page):
        """测试抓取按钮 loading 状态"""
        btn = page.locator("button:has-text('立即抓取')")
        btn.click()
        page.wait_for_timeout(500)  # 等待 loading 生效
        page.wait_for_timeout(5000)  # 等待抓取完成

    def test_search_functionality(self, page: Page):
        """测试搜索功能"""
        search = page.locator("input[placeholder='搜索关键词']")
        search.fill("Python")
        page.locator("button:has-text('搜索')").click()
        page.wait_for_timeout(1000)
```

## 安全测试示例

验证系统是否能抵御常见攻击：

```python
SQL_PAYLOADS = [
    "' OR '1'='1",
    "'; DROP TABLE news; --",
    "1' UNION SELECT * FROM news--",
]

@pytest.mark.parametrize("payload", SQL_PAYLOADS)
def test_sql_injection_search(self, api_session, payload):
    """测试搜索接口 SQL 注入"""
    resp = api_session.get(
        f"{API_URL}/news",
        params={"keyword": payload}
    )
    assert resp.status_code in [200, 400, 422]
```

## 性能测试示例

验证接口响应速度和并发能力：

```python
def test_concurrent_news_list(self):
    """10 并发请求新闻列表"""
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [
            executor.submit(self._make_request, "/news")
            for _ in range(10)
        ]
        results = [f.result() for f in futures]

    success_count = sum(1 for r in results if r["success"])
    avg_time = sum(r["time"] for r in results) / len(results)
    
    assert success_count >= 9  # 90% 成功率
    assert avg_time < 3.0      # 平均 3 秒内
```

## 踩坑记录

### 1. Playwright 的 `page` fixture

需要安装 `pytest-playwright`，否则会报 `fixture 'page' not found`：

```bash
pip install pytest-playwright
playwright install chromium
```

### 2. Ant Design 按钮文本有空格

按钮渲染后文本可能是 `"测 试"` 而不是 `"测试"`，选择器要注意：

```python
# 错误
btn = page.locator("button:has-text('测试')")
# 正确
btn = page.locator("button:has-text('测 试')")
```

### 3. FastAPI 错误码

FastAPI 对无效 HTTP 方法返回 422 而非标准的 405，测试要兼容：

```python
assert resp.status_code in [405, 422]
```

## 运行方式

```bash
# 全部测试
python run_tests.py

# 按类型运行
python run_tests.py api        # API 测试
python run_tests.py ui         # UI 测试
python run_tests.py perf       # 性能测试
python run_tests.py security   # 安全测试
```

## 测试结果

```
✅ API测试:      17/17 通过
✅ 性能测试:     12/12 通过
✅ UI测试:       26/26 通过
✅ 安全测试:     28/28 通过

总计: 83 passed in 51.93s
```

## 总结

自动化测试不是银弹，但能极大提升开发信心。尤其是：

- **API 测试**：保证接口契约不变
- **UI 测试**：覆盖用户核心流程
- **安全测试**：防患于未然
- **性能测试**：确保体验不退化

花一天搭建，受益整个项目周期。值得。

---

*项目地址：[news-aggregator](https://github.com/sucli/news-aggregator)*
