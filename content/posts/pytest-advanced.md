---
title: "pytest 进阶"
date: 2026-10-06T10:15:00+08:00
tags: [pytest, 测试]
---

上一篇 [pytest 入门](/posts/pytest-getting-started/) 结尾留了张路线图：conftest.py、mock、覆盖率、CI。这篇把它们走完，外加 fixture 作用域和标记这两个进阶篇绕不开的话题。示例统一带上类型注解，惯例的来由见上一篇的「类型注解」一节。

## conftest.py：跨文件共享 fixture

写在测试文件里的 fixture 只能本文件用。`conftest.py` 是 pytest 的约定文件：同目录及子目录下的测试**不需要 import** 就能使用它定义的 fixture。典型的测试布局：

```
myapp/
├── src/myapp/
│   ├── net.py
│   └── mail.py
└── tests/
    ├── conftest.py
    ├── test_net.py
    └── test_mail.py
```

```python
# tests/conftest.py
import pytest
from myapp.net import APIClient

@pytest.fixture
def client() -> APIClient:
    return APIClient(base_url="http://localhost:8000")
```

```python
# tests/test_net.py
from myapp.net import APIClient

def test_ping(client: APIClient) -> None:
    assert client.get("/ping").status_code == 200
```

这里 import 的是类型，不是 fixture——fixture 靠参数名在运行时注入，这正是 conftest.py 的好处。

conftest.py 可以在每层目录各放一个，测试就近取用——子目录里的同名 fixture 会覆盖上层的。利用这一点可以给不同目录配不同环境，比如单元测试用假数据、集成测试连真库。

## fixture 作用域与依赖

默认每个测试函数都要重新执行一次 fixture，耗资源的准备（建连接、起容器）没必要反复来，用 `scope` 拉长生命周期：

```python
from collections.abc import Iterator

@pytest.fixture(scope="session")
def db() -> Iterator[Engine]:
    conn = create_engine("sqlite:///:memory:")
    yield conn
    conn.dispose()
```

`scope` 有三档：`function`（默认，每个测试一次）、`module`（每个文件一次）、`session`（整个测试过程一次）。

fixture 还能依赖 fixture，把准备拆成积木逐层组装：

```python
@pytest.fixture
def user(db: Engine) -> User:
    return User(name="alice", db=db)
```

pytest 会先把 `db` 准备好再创建 `user`，测试里两个都能拿到。另外有个 `@pytest.fixture(autouse=True)`：不用写参数就自动注入所有测试，适合临时目录、屏蔽网络这类全局准备——但它让依赖关系隐身，能不用就不用。

## mock：把外部依赖换掉

网络、时钟、随机数是测试里的不确定因素，测它们之前先换成确定的东西。pytest 内置 `monkeypatch` fixture：它负责「改」，测试结束后负责「还」，不留副作用。

先看被测代码：

```python
# myapp/net.py
import json
from urllib.request import urlopen

def fetch(url: str) -> dict:
    with urlopen(url) as resp:
        return json.loads(resp.read())
```

测试它时不想真的发请求，用 `monkeypatch.setattr` 把 `fetch` 眼里的 `urlopen` 换成假货：

```python
from io import BytesIO

import pytest

from myapp import net

def test_fetch(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setattr(net, "urlopen", lambda url: BytesIO(b'{"ok": true}'))
    assert net.fetch("https://example.com") == {"ok": True}
```

`BytesIO` 自带 `read()`，正好顶上响应对象的位置。`setattr` 的前两个参数是「替换哪个模块的哪个名字」，也可以不 import 模块，写成字符串形式 `monkeypatch.setattr("myapp.net.urlopen", ...)`，两者等价。

关键在于目标要选对：替换发生在名字的**使用处**，不是定义处。`net.py` 里那句 `from urllib.request import urlopen` 已经把名字导进了 `myapp.net` 的命名空间，所以要换 `net.urlopen`；换成 `urllib.request.urlopen` 是拦不住的——`fetch` 查找的名字根本不在那里。

常用的一组方法：

- `setattr` / `delattr` — 替换或删除属性，模块、类、对象实例都行
- `setenv` / `delenv` — 修改环境变量
- `setitem` — 换掉字典里的某个键
- `chdir` — 切换工作目录

`setattr` 在 `test_fetch` 里演示过了，其余每个都给一个最小示例（`delattr` / `delenv` 默认在目标不存在时抛错，传 `raising=False` 可以容忍）：

```python
from pathlib import Path

import pytest

from myapp import net

# delattr —— 删掉属性，模拟"这个配置项不存在"
def test_missing_config(monkeypatch: pytest.MonkeyPatch) -> None:
    from types import SimpleNamespace
    conf = SimpleNamespace(host="localhost", port=8000)
    monkeypatch.delattr(conf, "port")
    assert getattr(conf, "port", 80) == 80   # 代码里用 getattr(…, 默认值) 取值

# setenv / delenv —— 环境变量与它的默认值
def test_data_dir(monkeypatch: pytest.MonkeyPatch, tmp_path: Path) -> None:
    monkeypatch.setenv("DATA_DIR", str(tmp_path))
    assert myapp.data_dir() == tmp_path

def test_default_data_dir(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.delenv("DATA_DIR", raising=False)
    assert myapp.data_dir() == Path("/tmp/default")

# setitem —— 换掉配置字典里的键
def test_retry(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setitem(myapp.settings, "retries", 1)   # 原本默认 3
    assert myapp.max_retries() == 1

# chdir —— 切到临时目录，文件读写不会弄脏真实目录
def test_export(monkeypatch: pytest.MonkeyPatch, tmp_path: Path) -> None:
    monkeypatch.chdir(tmp_path)
    net.export("out.txt")
    assert (tmp_path / "out.txt").exists()
```

所有改动都记在 monkeypatch 这个对象上，当前测试结束（无论通过还是失败）pytest 会逐项撤销——每个测试都从干净环境开始，不会互相污染。

如果还想断言"它被调用过、参数对不对"，装 [pytest-mock](https://pytest-mock.readthedocs.io/) 用 `mocker`：

```python
# pip install pytest-mock
from pytest_mock import MockerFixture

def test_notify(mocker: MockerFixture) -> None:
    send = mocker.patch("myapp.mail.send")
    myapp.notify("发布成功")
    send.assert_called_once_with("发布成功")
```

两者的分工：`monkeypatch` 适合"换掉实现"，`mocker` 适合"还要检查怎么被调用的"。

## 标记：skip、xfail 与自定义

有些用例天生不该在某些环境下跑，或者明知道会失败：

```python
import sys
import pytest

@pytest.mark.skip(reason="缺陷还没修好")
def test_known_bug() -> None: ...

@pytest.mark.skipif(sys.platform == "win32", reason="符号链接行为不同")
def test_symlink() -> None: ...

@pytest.mark.xfail(reason="上游接口未恢复")
def test_upstream() -> None: ...
```

`skip` 直接跳过；`xfail` 仍然会跑，但失败被记作"预期失败"而不算挂——如果哪天它反而通过了，会记作 xpassed，提醒你来更新这条测试。

自定义标记（比如把慢测试挑出来）需要先注册，否则 pytest 会发警告；配上 `--strict-markers` 则直接报错，防止拼写错误的标记静默通过：

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["slow: 耗时较长的测试"]
```

```python
@pytest.mark.slow
def test_full_backup() -> None: ...
```

```bash
pytest -m "not slow"   # 日常开发，跳过慢测试
pytest -m slow         # 发布前专门跑一遍
```

## 覆盖率：pytest-cov

```bash
pip install pytest-cov
pytest --cov=myapp --cov-report=term-missing
```

```
Name             Stmts   Miss  Cover   Missing
-------------------------------------------------
myapp/net.py        42      3    93%   18-20
myapp/mail.py       30      0   100%
-------------------------------------------------
TOTAL              120      5    96%
```

`Missing` 列直接指出哪些行没被测到，比一个总百分比有用得多。覆盖率不必追 100%——覆盖率高不等于测得好，但没覆盖到的分支一定没测。

## 接入 CI：GitHub Actions

测试的价值在于每次提交都被运行，而不是躺在本地。加一个 `.github/workflows/tests.yml`：

```yaml
name: tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python: ["3.12", "3.13", "3.14"]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: ${{ matrix.python }}
      - run: pip install pytest pytest-cov
      - run: pytest --cov=myapp
```

每次 push 和 PR，GitHub 会在三个 Python 版本上各跑一遍测试，哪个挂了一眼可见。公开仓库用标准 runner 是免费的，私有仓库则消耗每月的免费额度。

## 写在最后

到这里，pytest 的主线基本走完：写用例、断言、参数化、fixture、mock、覆盖率、CI。再往下是锦上添花：`pytest-xdist`（`-n auto` 多进程并行提速）、`pytest-randomly`（打乱执行顺序，暴露用例间的隐藏依赖）、fixture 工厂和自定义插件。

最后推荐一个值得养成的习惯：修 bug 之前，先写一个能复现它的失败测试，修完看着它变绿——这个 bug 就再也不会悄悄回来了。
