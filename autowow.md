Dumper	倾倒

Rotation	循环





# typing模块

一句话，用于"标注"类型

1.用于标注变量的类型

2.用于标注函数参数的类型，两者都是变量后面加冒号加类型

```python
#变量名: 类型 = 初始值
name: str = "张三"
#可选变量  Optional
from typing import Optional
user_id: Optional[int] = None
config: Optional[dict] = None

def 函数名(参数名: 类型, 参数名: 类型) :
    pass
```

3.用于标注函数返回值的类型

```python
#标注返回值类型用->，加在冒号前面
def 函数名(参数列表) -> 返回值类型:
    return 值
```

如有多个返回类型，用竖线

```python
def parse_input(value: str) -> int | float | None:
    try:
        return int(value)
    except ValueError:
        try:
            return float(value)
        except ValueError:
            return None
```

详细定义：`typing` 是 Python 的**类型提示（Type Hints）**模块，用于在代码中标注变量、函数参数和返回值的类型。它让代码更易读、更安全，并支持 IDE 自动补全和静态类型检查。

```python
from typing import Any, Callable
```

这行代码导入了 `typing` 模块中的两个重要类型：

- **`Any`**：任意类型

- **`Callable`**：可调用对象类型（函数/方法/类）

  - ```python
    #可调用对象类型的语法
    Callable[[参数类型], 返回值类型]
    [参数类型]：参数列表，用逗号分隔
    返回值类型：函数返回值类型
    
    Callable[[ParamType1, ParamType2, ...], ReturnType]
    ```



# flask模块

from flask import Flask, Response

`Response` 是 Flask 框架封装的**HTTP 响应对象类**，所有浏览器收到的返回内容，底层都会包装成 `Response` 实例。

浏览器发请求 → Flask 路由处理 → Flask 生成 Response 对象 → 底层 WSGI web 服务器(如 Werkzeug 内置服务器）把 Response 转成 HTTP 报文发给浏览器。

日常写 `return "文本"`、`return jsonify()`、`render_template()`，Flask 内部都会自动转换成 Response，手动导入 Response 是为了**自定义响应**。

```python
#Response 标准构造函数格式
Response(
    response=None,        # 响应主体，字符串/字节/生成器(流)
    status=200,           # HTTP状态码，数字或字符串"404 Not Found"
    headers=None,        # 自定义响应头，字典
    mimetype=None,       # 文件类型，等价于Content-Type头
    content_type=None    # 完整Content-Type，优先级高于mimetype
)
```

框架与服务器之间的关系

服务器只做**网络通信**，**不懂业务逻辑，不知道什么是路由、模板、Response**。

框架处理**业务逻辑**

WSGI 是 Python 规定的**服务器 ↔ 框架**中间接口标准，相当于两者的通用翻译官。

- Web 服务器实现 WSGI 服务端：负责收网络请求
- Flask 实现 WSGI 应用端：负责处理业务

交互流程：

1. 浏览器 → HTTP 报文 → Web 服务器
2. 服务器解析 HTTP，按照 WSGI 标准打包请求信息传给 Flask
3. Flask 运行视图函数，生成 Response 对象
4. Flask 把 Response 拆解成 WSGI 规定的三元组：`(状态码, 响应头列表, 响应体迭代器)`
5. 服务器拿到三元组，拼接成完整 HTTP 响应，返回浏览器



# json模块

`json` 是 Python **标准内置库**，不需要额外 pip 安装，直接 `import json` 就能用。

作用：实现程序内部数据和网络传输文本互相转换，即 **Python 对象 ↔ JSON 字符串** 的互相转换。

load()和loads()的区别：这里的s代表string

`json.load(f)`：**从文件流对象读取**JSON，转 Python 对象

- **文件流对象**（通过 open () 打开的文件）

`json.loads(s)`：**从字符串读取**JSON，转 Python 对象



# logging 模块

`logging` 是 Python **内置标准日志模块**，不用安装，直接 `import logging` 使用。

核心用途：**专业日志工具**，替代简单粗暴的 `print()`。

五大日志级别（从低到高）

级别数字越小，打印越详细；设置某个级别后，**高于等于该级别**的日志才会输出。

|                           级别常量                           | 数值 |                用途场景                |
| :----------------------------------------------------------: | :--: | :------------------------------------: |
|                        logging.DEBUG                         |  10  |     开发调试用，打印变量、流程细节     |
| [logging.INFO](https://link.wtturl.cn/?target=https%3A%2F%2Flogging.INFO&scene=im&aid=497858&lang=zh) |  20  |    正常运行记录，接口访问、启动成功    |
|                       logging.WARNING                        |  30  | 不影响运行的异常，如内存不足、参数过时 |
|                        logging.ERROR                         |  40  |   功能出错，接口报错、数据库查询失败   |
|                       logging.CRITICAL                       |  50  |       致命错误，程序无法继续运行       |

默认级别是 `WARNING`，默认只会打印 WARNING / ERROR / CRITICAL。