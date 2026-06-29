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



## flask模块

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