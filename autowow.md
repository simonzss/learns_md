Dumper	倾倒

Rotation	循环

# EZPixelDumperX2

## np数组在本程序中的具体形式

```python
import numpy as np

# 你的程序抓到的数据区是 50列 x 16行，每个像素由 RGB 三个通道组成
# 形状为 (16, 50, 3)  —— 16行高，50列宽，3通道
frame = np.array([
    # 第1行（y=0）：对应节点 (1,1) 到 (50,1)
    [[  0,   0,   0], [255, 200, 100], [120, 200, 255], ..., [  0,   0,   0]],  # 行索引0
    # 第2行（y=1）：对应节点 (1,2) 到 (50,2)
    [[199,  86,  36], [ 60, 100, 220], [255, 255, 255], ..., [ 80, 220, 120]],  # 行索引1
    # ... 中间省略 12 行 ...
    # 第16行（y=15）：对应节点 (1,16) 到 (50,16)
    [[  0,   0,   0], [255, 255, 255], [100, 120, 180], ..., [230, 200,  50]],  # 行索引15
], dtype=np.uint8)

print(frame.shape)  # 输出：(16, 50, 3)
print(frame.dtype)  # 输出：uint8
```



## PySide6 QThread + Signal 完整详解

一、基础背景：Qt UI 单线程卡死问题

Qt 的界面（按钮、输入框、窗口）**只能在主线程（UI 线程）操作**。

如果在主线程写耗时逻辑：文件批量读取、网络请求、循环计算、接口请求，界面会**冻结、无响应、点击没反应**。

`QThread` = 用来新开子线程，把耗时任务丢进去，不阻塞 UI；

`Signal` = 线程之间安全通信的信号机制，子线程不能直接操作 UI，只能发信号通知主线程更新界面。

1. QThread 是什么？作用

### 1.1 定义

`QThread` 是 Qt 封装的跨平台线程类，用来创建**独立执行的后台子线程**。

Python 自带 `threading.Thread` 也能开线程，但和 Qt 控件交互会崩溃，必须用 `QThread`。

### 1.2 核心用途

1. 将**耗时阻塞任务**（IO、计算、爬虫、读写大文件）放到后台执行；
2. 保证主线程只负责渲染 UI，窗口流畅不卡顿；
3. 提供线程启停、状态信号、安全退出机制。

### 1.3 QThread 核心内置自带信号（不用自己定义）

- `started()`：线程开始运行时触发
- `finished()`：线程正常执行完毕触发
- `terminated()`：线程强制终止触发（不推荐）
- `errorOccurred()`：线程报错触发

### 1.4 标准正确用法（继承 QThread 重写 run），注意是重写run

```python
from PySide6.QtCore import QThread

class WorkThread(QThread):
    def run(self):
        # run() 是线程真正执行的入口函数
        # 所有耗时代码写在这里
        for i in range(100):
            print(i)
            self.msleep(100)  # 线程休眠，不会卡住UI
```

```python
thread = WorkThread()
thread.start()  # 启动线程，自动执行run()
# thread.wait()  # 阻塞等待线程结束（主线程慎用）
# thread.terminate() # 强制杀死线程（危险，资源不释放）
```

### 1.5 关键规则

1. 只有 `run()` 函数内部代码运行在**子线程**；
2. 类中 `__init__` 构造函数运行在**主线程**，不要在构造里做耗时操作；
3. 禁止子线程直接操作任何 QWidget 控件（按钮、label、窗口），会崩溃 / 异常。

### 2. Signal 信号是什么？作用

完整类：`PySide6.QtCore.Signal`

### 2.1 核心作用

Qt **唯一线程安全的跨线程通信方式**：

- 子线程 → 发 Signal 信号
- 主线程绑定槽函数接收信号，在主线程更新 UI

为什么不能子线程直接改 UI？

Qt UI 组件不是线程安全，多线程同时读写控件会：界面花屏、程序闪退、随机崩溃。

信号内部自带消息队列，自动把执行切回主线程，安全修改界面。

### 2.2 Signal 定义语法

必须定义在**类属性层级**（不能写在 **init** /run 内部），支持传递不同类型数据：

```
# 无参数信号
signal_no_arg = Signal()
# 传递int数字
signal_int = Signal(int)
# 传递字符串
signal_str = Signal(str)
# 同时传int+str多参数
signal_multi = Signal(int, str)
# 传递布尔值
signal_bool = Signal(bool)
```

### 2.3 三要素：信号 Signal + 绑定 connect + 槽函数 slot

1. 自定义 Signal：在线程类顶部声明
2. `信号.connect(槽函数)` 绑定接收逻辑
3. `self.信号.emit(参数)` 发射信号，触发槽函数执行

### 完整可运行示例（结合 QThread + Signal）

```python
'''
先回顾三要素
信号   Signal ：子线程里的信号发射器（定义在线程类里）
槽函数 slot：定义在主线程里，接收信号后被调用的函数
connect()：把信号和槽绑定在一起
槽函数可以是任意普通函数、类方法，没有特殊语法标记；
信号发射时带多少个参数，槽函数就要对应接收多少个参数；
例：Signal(int) → 槽必须写 def func(self, int类型的变量):
'''
from PySide6.QtCore import QThread, Signal
from PySide6.QtWidgets import QApplication, QWidget, QLabel, QVBoxLayout


# 1. 自定义工作子线程
class CountThread(QThread):
    # 这两个是【信号 Signal】，自定义信号：传递一个整数
    update_num_signal = Signal(int)
    finish_signal = Signal()

    def run(self):
        # 子线程耗时逻辑
        for num in range(1, 11):
            self.update_num_signal.emit(num)  # emit发射信号，传数字
            self.msleep(300)  # 模拟耗时
        self.finish_signal.emit()  # 执行完成 emit发射信号，发射结束信号

# 2. UI窗口（主线程）
class Window(QWidget):
    def __init__(self):
        super().__init__()
        self.label = QLabel("等待计数...")
        layout = QVBoxLayout(self)
        layout.addWidget(self.label)

        # 创建子线程实例
        self.worker = CountThread()
        # connect(信号, 槽函数)，把信号和槽函数连接到一起，子线程一发信号，主线程就执行槽函数
        self.worker.update_num_signal.connect(self.refresh_label)
        self.worker.finish_signal.connect(self.task_finish)
        # 启动线程
        self.worker.start()

    # 下面两个就是槽函数：主线程执行，安全修改UI
    def refresh_label(self, value):
        self.label.setText(f"当前计数：{value}")

    def task_finish(self):
        self.label.setText("任务全部完成！")

if __name__ == "__main__":
    app = QApplication([])
    win = Window()
    win.show()
    app.exec()
```

运行逻辑拆解

1. `self.worker.start()` → 开启子线程，执行 `run()`；
2. 循环里 `emit(num)` 发射带数字的信号；
3. 信号触发主线程的 `refresh_label`，修改 Label 文字；
4. 全程界面不卡顿。

------

### 3. 核心配套知识点

### 3.1 信号槽三种连接方式（默认自动跨线程）

```
# 自动模式（默认，跨线程自动排队，推荐）
signal.connect(func, Qt.AutoConnection)
# 同线程直接调用，跨线程报错
signal.connect(func, Qt.DirectConnection)
# 放入主线程消息队列排队执行（UI更新专用）
signal.connect(func, Qt.QueuedConnection)
```

你不用手动指定，默认 `AutoConnection` 会自动判断线程，跨线程自动走队列，安全更新 UI。

### 3.2 线程安全注意事项

1. 子线程绝对不能：
   - 直接 `label.setText()` / button.click()
   - 新建窗口、控件
2. 所有 UI 修改逻辑，全部放到**槽函数**，由 Signal 触发；
3. 线程传递大数据：通过 Signal 传参，不要全局变量共享（会竞争锁）。

### 3.3 线程退出规范（避免内存泄漏）

绑定线程自带 `finished` 信号，结束后销毁：

```
self.worker.finished.connect(self.worker.deleteLater)
```

禁止频繁 `terminate()` 强制杀线程，会造成文件句柄、网络连接无法释放。





## typing模块

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



## json模块

`json` 是 Python **标准内置库**，不需要额外 pip 安装，直接 `import json` 就能用。

作用：实现程序内部数据和网络传输文本互相转换，即 **Python 对象 ↔ JSON 字符串** 的互相转换。

load()和loads()的区别：这里的s代表string

`json.load(f)`：**从文件流对象读取**JSON，转 Python 对象

- **文件流对象**（通过 open () 打开的文件）

`json.loads(s)`：**从字符串读取**JSON，转 Python 对象



## logging 模块

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



## 回调函数

#### 主函数里的参数callback

- 看到一个函数的参数是callback，就说明这个函数是接收回调函数的主函数，回调函数在另外的地方

将一个函数 A，作为参数传递给另一个函数 B，由函数 B 在内部、合适时机自动调用函数 A，A 就叫做回调函数。

```python
# 这是回调函数
def say_hello(name):
    print(f"你好，{name}！")

# 接收回调的主函数，参数callback用来接收函数
def handle_user(name, callback):
    print("开始处理用户信息...")
    # 在内部主动调用传入的回调函数
    callback(name)
    print("处理完成")

# 使用：把say_hello函数本身传进去，不要加()
handle_user("小明", say_hello)

'''
运行输出：
开始处理用户信息...
你好，小明！
处理完成
'''
```

以上的同步回调是立刻执行，**异步回调才是回调诞生的核心用途**：等待耗时操作完成后再执行回调函数。

模拟网络请求、文件读取、定时任务：

```python
import time

# 耗时操作：模拟网络请求
def request_data(callback):
    print("正在请求数据，等待2秒...")
    time.sleep(2)  # 模拟网络延迟
    data = {"code": 200, "msg": "请求成功", "data": [10,20,30]}
    # 请求完成，自动调用传入的回调，把结果传给回调
    callback(data)

# 回调函数：拿到请求后的结果做后续处理
def handle_response(res):
    if res["code"] == 200:
        print("接收数据：", res["data"])

# 发起请求，传入回调
request_data(handle_response)
print("主线程继续运行，不会阻塞")
```

运行逻辑：

1. 调用`request_data`，把处理结果的函数传进去
2. 程序等待 2 秒（耗时操作）
3. 耗时操作结束，内部自动执行回调，把数据丢给回调处理

> 这就是回调最经典场景：**耗时任务完成后自动执行后续逻辑**





## array 、 list、tuple的 完整区别

先分清两个东西：

1. `list`：Python 内置列表，`[]`，无限制，可以混合存放任意类型：数字、字符串、列表、对象都行

2. `array.array`：标准库数组，需要 `import array`，**只能存同一种基础数字类型**，创建时必须指定类型码

3. `list`：每个元素存的是对象指针，内存开销大，大量数字运算慢

   `array.array`：底层是连续 C 原生数值，不存对象，内存更小、读写更快，十万、百万级数字时差距非常明显。

   list 适用：日常所有通用场景：数据混杂、需要频繁增删排序、存字符串 / 复合数据。list 方法丰富（支持增删改、排序）

   array.array 适用：只存大批量纯数字、需要节省内存、二进制文件读写、追求更快遍历速度。但**没有 sort、reverse 等内置排序**

元组是 `tuple ()`，一旦定义，**元素不能修改、不能新增、不能删除**

- 特例：元组里如果包含列表，列表内部能改，但元组本身结构不变

- 2. 写法符号

  - 列表（数组）：方括号 

    ```
    arr = [10,20,30]
    ```

  - 元组：圆括号 ，单个元素必须加逗号

    ```python
    t = (10,20,30)
    t = (5,)    #单个元组，逗号不能少
    ```

      

## `__init__` 的作用

一看到`__init__` ，就要想到实例

`__init__` 是 Python 类的**构造方法（Constructor）**，在创建类的实例时自动调用。

## 关键区别：`__new__` vs `__init__`

理解 `__init__` 的真正角色，需要知道完整的对象创建流程：

```Python
class Person:
    def __new__(cls, name):     # ← 真正的构造函数
        print(f"1. 创建对象（分配内存）并且返回实例")
        instance = super().__new__(cls)
        return instance
    
    def __init__(self, name):   # ← 初始化方法
        print(f"2. 初始化对象（填充数据）")
        self.name = name

# 创建对象
p = Person("张三")

# 输出：
# 1. 创建对象（分配内存）
# 2. 初始化对象（填充数据）
```

### `__init__` 其正式名称是**初始化方法（Initialization Method）**

其中的self强制作为第一个参数，`self` 是实例的"身份证"，每个对象都有唯一的 `self`

### 1. 实例变量（通过 self 访问）

```
class CameraWorker:
    def __init__(self, camera):
        self.camera = camera  # ← 实例变量，每个对象独立
        
worker1 = CameraWorker("cam1")
worker2 = CameraWorker("cam2")
print(worker1.camera)  # "cam1"
print(worker2.camera)  # "cam2"（不同对象，不同值）
```



### 2. 类变量（直接定义在类中）

```
class CameraWorker:
    default_fps = 30  # ← 类变量，所有实例共享
    
    def __init__(self, camera):
        self.camera = camera

worker1 = CameraWorker("cam1")
worker2 = CameraWorker("cam2")
print(worker1.default_fps)  # 30
print(worker2.default_fps)  # 30（共享）
```



### 3. 局部变量（在方法内定义）

```
class CameraWorker:
    def __init__(self, camera):
        local_var = "局部"  # ← 局部变量，只在方法内有效
        self.camera = camera
        
    def run(self):
        print(local_var)  # ❌ 错误！局部变量在 __init__ 外不可见
```



## `super()` "代理"与"实例"的区别

```python
# 以下都是为了理解super().__init__()

class QThread:
    def __init__(self):
        print("QThread 初始化")

class CameraWorker(QThread):
    def __init__(self):
        # super() 返回的不是 QThread 的实例
        proxy = super()  # ← 这是一个"代理对象"
        
        print(type(proxy))        # <class 'super'>
        print(isinstance(proxy, QThread))  # False！不是 QThread 实例 
        
        super().__init__()  # self 才是 CameraWorker 实例
        # 上句等价于：QThread.__init__(self)，注意self代表的是子类CameraWorker的实例
        # 核心是：在子类实例self上添加了父类才有的属性和方法
        # 父类QThread并没有被实例化
```

| **代理是什么** | 一个"导航员"对象，帮你找到父类的方法  |
| -------------- | ------------------------------------- |
| **是实例吗**   | 不是！它只是一个特殊的工具对象        |
| **主要作用**   | 调用父类方法时自动传递 self           |
| **为什么需要** | 处理继承链、支持多重继承、避免硬编码  |
| **返回什么**   | 一个 `super` 类型的对象，不是父类实例 |

**`super().__init__()` 的本质就是：在”子类实例“上执行父类的初始化代码，让父类为”子类实例“添加父类的属性和方法(在子类实例上完成了父类对”子类实例“的改造)**。

整个过程中，父类没有实例化，即没有父类的实例





## Flask模块

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

### 创建Flask应用

##### 第一步是通过\_\_name\_\_确定根目录

```python
from flask import Flask

# 创建 Flask 应用
app = Flask(__name__)

# Flask 内部会：
# 1. 根据 __name__ 确定根目录
# 2. 查找 templates/ 文件夹（存放HTML模板）
# 3. 查找 static/ 文件夹（存放CSS/JS/图片）

# 例如，如果 __name__ = 'Worker'：
# ├── worker.py (当前文件)
# ├── templates/    ← Flask 会在这里找模板
# │   └── index.html
# └── static/       ← Flask 会在这里找静态文件
#     ├── style.css
#     └── script.js

# 在 worker.py 文件中
# __name__ 的值：
# ┌────────────────────────────────────────────────────────┐
# │ 如果 worker.py 是主程序：                            │
# │   python worker.py  → __name__ = '__main__'         │
# │                                                       │
# │ 如果 worker.py 被导入：                              │
# │   from Worker import WebServerWorker                 │
# │   → __name__ = 'Worker' (模块名)                    │
# └────────────────────────────────────────────────────────┘
```

### 装饰器的本质是"函数替换"

**装饰器本质上是一个函数，它接受一个函数A作为参数，并返回一个新的函数B**

这时若执行函数A，那么实际上执行的函数就被”替换“为B，从而实现**在不修改原函数代码的情况下，为原函数添加额外的功能。**

其语法是@后面跟装饰器函数

```python
# 这是一个装饰器函数
def my_decorator(func):
    def wrapper():
        print("装饰器：在函数执行前做点什么")
        func()  # 执行原始函数
        print("装饰器：在函数执行后做点什么")
    return wrapper

# 使用装饰器
@my_decorator
def say_hello():
    print("Hello!")

# 调用函数
say_hello()
# 完全等价于：
# 这里的say_hello = my_decorator(say_hello)  # 函数被替换了！say_hello = wrapper 
# 于是say_hello()= my_decorator(say_hello)() # 调用say_hello() 时，实际执行的是 wrapper()

# 输出：
# 装饰器：在函数执行前做点什么
# Hello!
# 装饰器：在函数执行后做点什么
```

图解装饰器执行过程

```
原始函数: say_hello
    ↓
@my_decorator
    ↓
my_decorator(say_hello)  ← 调用装饰器
    ↓
返回 wrapper 函数
    ↓
say_hello = wrapper  ← 函数被替换
    ↓
调用 say_hello() 时，实际执行的是 wrapper()
```

### Flask 路由核心机制：URL 变量提取

Flask 路由的核心功能之一就是：**从 URL 中提取被 `<>` 包裹东西作为变量，并将这些变量作为参数传递给视图函数。**

```python
@app.route('/user/<username>') # name 默认是字符串类型，完整为'/user/<string:username>'
def user_profile(username):  # ← username 来自 URL
    return f"用户: {username}"

# 访问 /user/alice → username = 'alice'
# 访问 /user/bob → username = 'bob'
```

指定类型转换器

```python
# 整数转换器
@app.route('/user/<int:user_id>')
def user_detail(user_id):
    # user_id 是整数类型
    return f"用户ID: {user_id}"

# /user/123 → user_id = 123 (int)
# /user/abc → 404 错误

# 浮点数转换器
@app.route('/price/<float:value>')
def show_price(value):
    # value 是浮点数类型
    return f"价格: {value}"

# /price/19.99 → value = 19.99 (float)

# 路径转换器（包含斜杠）
@app.route('/files/<path:filepath>')
def get_file(filepath):
    # filepath 可以包含斜杠
    return f"文件路径: {filepath}"

# /files/folder/subfolder/file.txt 的结果是filepath = 'folder/subfolder/file.txt'

# UUID 转换器
@app.route('/uuid/<uuid:uid>')
def get_uuid(uid):
    return f"UUID: {uid}"

# /uuid/123e4567-e89b-12d3-a456-426614174000 → 有效
```

```python
@self._app.route('/', defaults={'path': ''})  
@self._app.route('/<path:path>')  #这句的意思是根目录后面的所有URL用路径转换器提取，上面一行因为只有根目录，而根目录的path是空值，没法再写path转换器，因此必须用defaults指定path的值为空
```



### 视图函数

核心理解

```python
def _setup_routes(self) -> None:
   """设置Flask路由。"""
	@self._app.route('/', defaults={'path': ''})
	@self._app.route('/<path:path>')
	def catch_all(path: str) -> Response:
```



> **`catch_all` 是视图函数（View Function），不是普通函数。**
>
> - 普通函数：你定义，你调用
> - 视图函数：你定义，Flask 调用
> - Flask会根据URL查找对应路由表，自动调用对应的视图函数，不需要程序员再显式调用。

### 端点名

**端点名（Endpoint）是 Flask 内部给视图函数起的"别名"或"标识符"，用于在框架内部唯一标识一个视图函数。**

端点名在 Flask 中相当于一个"内部名字"，方便你通过 `url_for()` 函数反向生成 URL，也方便框架内部管理路由。当你不指定端点名时，Flask 会自动使用**函数名**作为端点名：

------

1. 端点名的本质

```
# Flask 内部的路由映射结构（简化）
url_map = {
    '/': {
        'endpoint': 'catch_all',      # ← 端点名
        'view_func': catch_all,       # ← 视图函数
        'methods': ['GET']
    },
    '/<path:path>': {
        'endpoint': 'catch_all',      # ← 相同端点名
        'view_func': catch_all,       # ← 相同视图函数
        'methods': ['GET']
    }
}
```



2. 默认端点名

当你不指定端点名时，Flask 会自动使用**函数名**作为端点名：





### 多个装饰器

#### 多个装饰器的执行规则（非Flask场景）

在普通的 Python 代码中，被多个装饰器装饰的函数，调用时**所有装饰器都会被执行**，且执行顺序是**从下往上**（从最靠近函数的装饰器开始）。

但需要明确区分两个阶段：

1. **定义阶段**：装饰器函数本身被执行
2. **调用阶段**：被装饰的函数被调用时，装饰器的包装代码被执行

------

1. 多个装饰器的执行顺序

```python
def decorator_a(func):
    print("A: 定义阶段")
    def wrapper(*args, **kwargs):
        print("A: 调用阶段 - 执行前")
        result = func(*args, **kwargs)
        print("A: 调用阶段 - 执行后")
        return result
    return wrapper

def decorator_b(func):
    print("B: 定义阶段")
    def wrapper(*args, **kwargs):
        print("B: 调用阶段 - 执行前")
        result = func(*args, **kwargs)
        print("B: 调用阶段 - 执行后")
        return result
    return wrapper

@decorator_a
@decorator_b
def say_hello():
    print("Hello!")

# 输出（定义阶段）：
# B: 定义阶段
# A: 定义阶段

# 调用函数
say_hello()

# 输出（调用阶段）：
# A: 调用阶段 - 执行前
# B: 调用阶段 - 执行前
# Hello!
# B: 调用阶段 - 执行后
# A: 调用阶段 - 执行后
```

#### Flask情景下的多个路由装饰器

Flask的路由装饰器是注册型装饰器（即装饰器函数中不执行被装饰函数），路由装饰器的执行顺序是从下往上，但这**只发生在程序启动时的注册阶段**。

Flask的route装饰器有几个，就会在路由表注册几次，但给定一个URL时，试图函数的调用就只根据路由表调用一次

当一个 HTTP 请求到达时，Flask 会遍历所有注册的路由，**匹配到第一个符合条件的路由就会执行对应的视图函数，随后中止匹配，不会按装饰器的顺序再去匹配别的路由。**

