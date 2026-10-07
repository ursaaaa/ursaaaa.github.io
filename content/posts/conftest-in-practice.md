---
title: "conftest.py 实战"
date: 2026-10-06T10:40:00+08:00
tags: [pytest, 测试]
series: [pytest]
series_order: 3
---

系列第三篇：[pytest 入门](/posts/pytest-getting-started/) 写基础用法，[pytest 进阶](/posts/pytest-advanced/) 讲到 conftest 的就近覆盖。这篇让 conftest.py 当主角，分享几个我在真实项目里反复用到的配方。

## 全局一层，专用一层

一个中型项目的测试布局通常长这样：

```
myapp/
├── conftest.py            # 全局：命令行参数、hook、插件注册
├── pyproject.toml
└── tests/
    ├── conftest.py        # 各目录共享的 fixture
    ├── fixtures/
    │   ├── __init__.py
    │   ├── db.py
    │   └── api.py
    ├── unit/
    │   └── test_logic.py
    └── integration/
        ├── conftest.py    # 集成测试专用
        └── test_api.py
```

划分原则一句话：**命令行参数和 hook 放最外层，共享 fixture 放 tests/，专用的贴着测试放**。`pytest_addoption` 这类 hook 只在最外层 conftest.py 里被调用，写在子目录是无效的——这是 conftest 的第一个坑。

## 配方一：fixture 拆到独立文件

测试多了以后，conftest.py 会膨胀成几百行。把 fixture 按主题拆成模块，用 `pytest_plugins` 注册（同样只能写在最外层）：

```python
# conftest.py（项目根目录）
pytest_plugins = ["tests.fixtures.db", "tests.fixtures.api"]
```

```python
# tests/fixtures/db.py
from collections.abc import Iterator

import pytest
from myapp.db import create_connection

@pytest.fixture
def db() -> Iterator[Connection]:
    conn = create_connection()
    yield conn
    conn.close()
```

```python
# tests/fixtures/api.py
import pytest
from myapp.net import APIClient

@pytest.fixture
def api_client(db: Connection) -> APIClient:
    return APIClient(db=db)
```

测试照常把名字写成参数，不用关心 fixture 定义在哪个文件。前提是 `tests/` 是包（有 `__init__.py`），`pytest_plugins` 里的模块路径才能被导入。

## 配方二：给测试加命令行参数

测哪个环境、用什么账号——这些不该写死在代码里。`pytest_addoption` 注册参数，fixture 把值递给测试：

```python
# conftest.py（项目根目录）
import pytest

def pytest_addoption(parser: pytest.Parser) -> None:
    parser.addoption("--base-url", action="store", default="http://localhost:8000")

@pytest.fixture
def base_url(request: pytest.FixtureRequest) -> str:
    return request.config.getoption("--base-url")
```

```python
# tests/integration/test_api.py
def test_ping(base_url: str) -> None:
    assert base_url.startswith("http")
```

```bash
pytest                                        # 用默认值，测本地
pytest --base-url=https://staging.example.com # 换个环境跑
```

参数从命令行流进 fixture，同一套测试就能在本地和预发环境之间切换。

## 配方三：--runslow，把慢测试藏起来

进阶篇注册过 `slow` 标记，现在让"日常跳过慢测试、需要时一键打开"自动化：

```python
# conftest.py（项目根目录）
import pytest

def pytest_addoption(parser: pytest.Parser) -> None:
    parser.addoption("--runslow", action="store_true", default=False, help="包含 slow 标记的测试")

def pytest_collection_modifyitems(config: pytest.Config, items: list[pytest.Item]) -> None:
    if config.getoption("--runslow"):
        return
    skip_slow = pytest.mark.skip(reason="加 --runslow 才运行")
    for item in items:
        if "slow" in item.keywords:
            item.add_marker(skip_slow)
```

```
$ pytest

tests/test_backup.py::test_full_backup SKIPPED (加 --runslow 才运行)

$ pytest --runslow

tests/test_backup.py::test_full_backup PASSED
```

日常 `pytest` 几秒跑完，发版前 `pytest --runslow` 全量过一遍。

## 配方四：在子目录里"扩展"父级 fixture

进阶篇说过，子目录的同名 fixture 会挡住上层。想**扩展**而不是替换时，让子 fixture 接收一个同名参数——pytest 会先调用上层 fixture，把结果交给你加工：

```python
# tests/conftest.py
from typing import Any

import pytest

@pytest.fixture
def client() -> dict[str, Any]:
    return {"timeout": 1}
```

```python
# tests/integration/conftest.py
from typing import Any

import pytest

@pytest.fixture
def client(client: dict[str, Any]) -> dict[str, Any]:  # 参数名触发上层 fixture
    client["timeout"] = 5
    return client
```

```python
# tests/integration/test_api.py
from typing import Any

def test_slow_network(client: dict[str, Any]) -> None:
    assert client["timeout"] == 5
```

```python
# tests/unit/test_logic.py
from typing import Any

def test_default_client(client: dict[str, Any]) -> None:
    assert client["timeout"] == 1
```

unit 拿到原始值，integration 拿到加工后的值，各自成立、互不干扰。这个"同名包裹"是在分层目录里定制 fixture 的标准做法。

## 用 conftest 的几条军规

- **不要 `from conftest import x`** —— pytest 会把每层 conftest 当作独立模块加载，import 进来的可能是另一份副本；要共享就写成 fixture
- **同名覆盖是替换，不是合并** —— 想扩展就用配方四的同名包裹
- **hook 只认最外层** —— `pytest_addoption`、`pytest_plugins` 都写在项目根的 conftest.py
- **conftest 报错是大事** —— 里面 import 失败，整个目录的测试都收集不到；改完顺手 `pytest --collect-only` 看一眼
- **autouse 少用** —— 进阶篇说过，这里再念一遍

## 写在最后

五个配方覆盖了 conftest.py 的绝大多数日常：拆文件、收参数、控标记、分层定制。等有一天 conftest.py 写不下了，该考虑的不是继续往里塞，而是把它升级成本地插件（`-p` 参数加载），或者上 `pytest-xdist` 把慢套件并行化——这两篇有空再写。
