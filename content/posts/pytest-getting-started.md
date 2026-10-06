---
title: "pytest 入门"
date: 2026-10-06T09:50:00+08:00
---

写 Python 免不了写测试。标准库自带 unittest，但用例写起来啰嗦、失败信息也不够直观。[pytest](https://docs.pytest.org/) 用极少的样板代码解决了这些问题，是目前 Python 社区事实上的测试标准。

## 为什么是 pytest

对比一下两种写法，测试"1 + 1 = 2"：

```python
# unittest
import unittest

class TestMath(unittest.TestCase):
    def test_add(self):
        self.assertEqual(1 + 1, 2)
```

```python
# pytest
def test_add():
    assert 1 + 1 == 2
```

没有类、没有 `self`、没有记忆各种 `assertXxx` 方法的负担——直接用 Python 原生的 `assert`。此外 pytest 的失败报告非常详细，插件生态（覆盖率、mock、异步测试等）也很成熟。

## 第一个测试

安装：

```bash
pip install pytest
```

新建 `test_math.py`：

```python
def test_add():
    assert 1 + 1 == 2

def test_divide():
    assert 6 / 2 == 3
```

在文件所在目录运行：

```bash
$ pytest

test_math.py ..                                                  [100%]

============== 2 passed in 0.01s ==============
```

每个用例对应一个点号，`.` 通过，`F` 失败，`s` 被跳过。想看到每个用例的名字，加 `-v`。

## 断言的详细输出

pytest 最大的便利之一：断言失败时，它会展示表达式中每个中间变量的值，而不是只告诉你"不相等"。

```python
def test_string():
    result = "Hello".lower()
    assert result == "hi"
```

失败时输出类似：

```
    def test_string():
        result = "Hello".lower()
>       assert result == "hi"
E       AssertionError: assert 'hello' == 'hi'
E         - hello
E         + hi
```

一眼就能看出实际值是什么，省去了在测试里加 `print` 调试的时间。

## 测试发现规则

pytest 按以下约定自动收集用例：

- 文件名匹配 `test_*.py` 或 `*_test.py`
- 函数名以 `test_` 开头
- 类名以 `Test` 开头，且不带 `__init__` 方法

不符合约定的用例不会被收集，这也是初学者最常踩的坑——函数写好了却"没被跑到"，先检查命名。

## 断言异常

用 `pytest.raises` 检查代码是否抛出了预期的异常，`match` 可以进一步校验异常信息：

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError, match="除数不能为零"):
        divide(1, 0)
```

## 参数化：一组数据跑一个用例

同一个逻辑要验证多组输入时，用 `@pytest.mark.parametrize` 而不是复制粘贴测试函数：

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert a + b == expected
```

pytest 会把每组参数展开成独立的用例分别运行和报告，哪一组失败了清清楚楚。

## fixture：优雅的准备工作

测试经常需要先准备数据、连数据库、起临时文件，跑完再清理。unittest 的做法是 `setUp`/`tearDown`，pytest 用 fixture：

```python
import pytest

@pytest.fixture
def sample_list():
    data = [1, 2, 3]
    yield data
    print("清理资源")  # yield 之后的代码是收尾逻辑

def test_length(sample_list):
    assert len(sample_list) == 3
```

要点：

- 测试函数把 fixture 的名字写成参数，pytest 会自动注入
- `yield` 之前是准备，之后是清理，天然对称
- 多个测试可以复用同一个 fixture；通过 `scope="module"` 等参数还能控制它的生命周期

## 类型注解

pytest 按命名约定收集用例，注解只是普通元数据，不影响收集和运行，可以放心给测试加上类型。

fixture 的类型写在 fixture 函数的返回注解上，`yield` 式 fixture 惯例标成 `Iterator[T]`，测试参数的类型 IDE 会自动推断过去：

```python
from collections.abc import Iterator

@pytest.fixture
def sample_list() -> Iterator[list[int]]:
    data = [1, 2, 3]
    yield data

def test_length(sample_list: list[int]) -> None:
    assert len(sample_list) == 3
```

pytest 自身带完整类型标注，内置 fixture 和 `pytest.raises` 的返回值都有现成类型：

```python
def test_output(capsys: pytest.CaptureFixture[str]) -> None: ...

with pytest.raises(ValueError) as excinfo:   # ExceptionInfo[ValueError]
    divide(1, 0)
```

两点边界：

- fixture 按参数名在运行时注入，mypy/pyright 校验不了参数名是否真有对应的 fixture，名字写错要到跑测试时才报 `fixture not found`
- `@pytest.mark.parametrize` 的数据表不做类型检查

注解的价值主要在 IDE 补全和自查，不能替代跑测试；如果项目开了 `mypy --strict`，测试函数补个 `-> None` 就满足要求了。

## 常用命令行

```bash
pytest                # 运行当前目录下所有测试
pytest test_math.py   # 只跑指定文件
pytest -v             # 显示每个用例的名字
pytest -k "add"       # 只跑名字里含 add 的用例
pytest -x             # 遇到第一个失败就停止
pytest --lf           # 只重跑上次失败的用例
```

日常开发中最顺手的是 `pytest -x -v`：快速定位第一个失败，再对着用例名修。

## 写在最后

到这里，pytest 的核心已经覆盖：裸 `assert`、`raises`、`parametrize`、`fixture`，加上几个常用参数，足够支撑大多数项目的测试了。

进阶可以再了解 `conftest.py`（跨文件共享 fixture）、`pytest-cov`（覆盖率）和 `mocker`（mock 外部依赖），配合 CI 跑起来，测试才算真正落地。
