# Pytest Fixture Note


最近在学习实践自动化相关的知识，最终选用 pytest 来组织测试用例。

本文是 pytest 学习笔记的第一篇。

<!--more-->

Fixture 是 pytest 中的一个基本概念，可以简单理解为在测试用例前需要执行的内容，我用来初始化环境、准备数据等工作。

## Fixture

最近在学习实践自动化相关的知识，最终选用 pytest 来组织测试用例，本文是 pytest 学习笔记的第一篇。
Fixture 是 pytest 中的一个基本概念，可以简单理解为在测试用例前需要执行的内容，我用来初始化环境、准备数据等工作。

### Fixture

在被当做 fixture 的函数前面加上` @pytest.fixture`来定义一个 Fixture

```python
@pytest.fixture()
def before():
    print('\nbefore each test')
```

#### Scope

-   function：每个 test 都运行，默认是 function 的 scope
-   class：每个 class 的所有 test 只运行一次
-   module：每个 module 的所有 test 只运行一次
-   session：每个 session 只运行一次

```python
@pytest.fixture(scope="module")
def smtp():
    smtp = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
    yield smtp
    #yield下面是teardown内容
    smtp.close()

def test_ehlo(smtp):
#fixture的function名称，可以直接作为参数，传给需要使用它的测试样例。 在使用时，smtp并非前面定义的function，而是function的返回值，即smtplib.SMTP
    response, msg = smtp.ehlo()
    assert response == 250
    assert b"smtp.gmail.com" in msg
```

#### conftest.py

`conftest.py`是 pytest 的默认配置文件，可以在其中放公用的 fixture 或 plugin。

```bash
tests
├── conftest.py
├── test_a.py
├── test_b.py
└── sub
    ├── __init__.py
    ├── conftest.py
    ├── test_c.py
    └── test_d.py
```

`conftest.py`遵守就近原则，会优先使用层级最近的 conftest 中定义的 Fixture。同时外层的测试用例 a,b 不能使用内层`conftest.py`中定义的 fixture

### Use Fixture

#### 1. 当做参数直接调用

```python
@pytest.fixture(scope="module")
def smtp():
    smtp = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
    yield smtp
    smtp.close()

def test_ehlo(smtp):
    response, msg = smtp.ehlo()
    assert response == 250
    assert b"smtp.gmail.com" in msg
```

fixture 的 function 名称，可以直接作为参数，传给需要使用它的测试样例。 在使用时，`smtp`并非前面定义的 function，而是 function 的返回值，即`smtplib.SMTP`

#### 2. 在函数前用 Fixture Decorator 调用

```python
@pytest.mark.usefixtures("before")
def test_1():
    print('test_1()')

class Test1:
    @pytest.mark.usefixtures("before")
    def test_3(self):
        print('test_1()')

    @pytest.mark.usefixtures("before")
    def test_4(self):
        print('test_2()')

@pytest.mark.usefixtures("before")
class Test2:
    def test_5(self):
        print('test_1()')

    def test_6(self):
        print('test_2()')
```

#### 3. 用 Autouse 调用 Fixture

fixture decorator 一个 optional 的参数是`autouse`, 默认设置为 False。
当默认为 False，就可以选择用上面两种方式来试用 fixture。
当设置为 True 时，在一个 session 内的所有的 test 都会自动调用这个 fixture。

```python
@pytest.fixture(autouse=True)
def before():
    print('\nbefore each test')
```

### Finallizer

```python
@pytest.fixture()
def smtp(request):
    smtp = smtplib.SMTP("smtp.gmail.com")
    def fin():
    #释放函数
        print ("teardown smtp")
        smtp.close()
    request.addfinalizer(fin)
    #测试完成后调用
    return smtp
```

通过`addfinallizer()`注册释放函数

### Parametrizing

fixture 可以通过参数化来循环使用预设的参数

#### 1. params

```python
@pytest.fixture(params=["smtp.gmail.com", "mail.python.org"])
def smtp(request):
    smtp = smtplib.SMTP(request.param, 587, timeout=5)
    yield smtp
    print ("finalizing %s" % smtp)
    smtp.close()
```

在` @pytest.fixture`中，指定参数`params`，就可以利用特殊对象（`request`）来引用`request.param`。 使用以上带参数的 smtp 的测试样例，都会被执行两次。

#### 2. @pytest.mark.parametrize

```python
def add(a, b):
    return a + b

@pytest.mark.parametrize("test_input, expected", [
    ([1, 1], 2),
    ([2, 2], 4),
    ([0, 1], 1),
])
def test_add(test_input, expected):
    assert expected == add(test_input[0], test_input[1])
```

### Build-in Fixture

`pytest --fixtures`可以列出所有可用的 fixture，包括内置的、插件中的、以及当前项目定义的。

#### capsys

`capsys`可以捕捉测试 function 的标准输出

```python
def test_print(capsys):
    print('hello')
    out, err = capsys.readouterr()
    assert 'hello' == out
```

#### tmpdir

`tmpdir`则可以自动创建临时文件夹

```python
def test_path(tmpdir):
    from py._path.local import LocalPath
    assert isinstance(tmpdir, LocalPath)
    from os.path import isdir
    assert isdir(str(tmpdir))
```

### 参考

-   [《Pytest 中的 Fixture》](http://note.qidong.name/2018/01/pytest-fixture/)
-   [《Pytest Fixture》](http://senarukana.github.io/2015/05/29/pytest-fixture/)
-   [《pytest fixtures: explicit, modular, scalable》](https://docs.pytest.org/en/latest/fixture.html#fixture-finalization-executing-teardown-code)

