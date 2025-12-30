---
title: "计算机基础：设计模式（Part 3/6）"
excerpt: '二十二个设计模式，代码审计的思考'

collection: theory
category: basic
permalink: /theory/basic/design
tags: 
  - cs
  - design pattern

layout: single
read_time: true
author_profile: false
comments: true
share: true
related: true
---

![](../../images/theory/basic/design/abstract.png)

## 创建型模式

创建型模式[^1]提供了创建对象的机制， 能够提升已有代码的灵活性和可复用性。

### 1. 单例模式

**定义**：

确保一个类只有一个实例，并提供全局访问点。就像公司只有一个CEO，大家都要通过这个职位来联系CEO[^2]。

**例子**：

数据库连接池
```python
class DatabaseConnection:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.connection = "数据库连接已建立"
        return cls._instance
    
    def query(self, sql):
        print(f"执行查询: {sql}")

# 使用时
db1 = DatabaseConnection()
db2 = DatabaseConnection()
print(db1 is db2)  # True，是同一个实例
```

**代码审计思考**：
- 检查是否真的需要全局唯一实例
- 注意线程安全问题（多线程环境下可能创建多个实例）
- 考虑是否阻碍了单元测试（单例难以被mock替换）

### 2. 工厂方法模式

**定义**：

定义创建对象的接口，但让子类决定实例化哪个类。就像汽车工厂，父类规定造车流程，子类决定造SUV还是轿车。

**例子**：

日志记录器
```python
from abc import ABC, abstractmethod

class Logger(ABC):
    @abstractmethod
    def log(self, message):
        pass

class FileLogger(Logger):
    def log(self, message):
        print(f"文件日志: {message}")

class DatabaseLogger(Logger):
    def log(self, message):
        print(f"数据库日志: {message}")

class LoggerFactory:
    def create_logger(self, logger_type):
        if logger_type == "file":
            return FileLogger()
        elif logger_type == "database":
            return DatabaseLogger()
        else:
            raise ValueError("不支持的日志类型")

# 使用时
factory = LoggerFactory()
logger = factory.create_logger("file")
logger.log("用户登录成功")
```

**代码审计思考**：
- 检查工厂类是否过于复杂（违背单一职责原则）
- 查看新增产品类型时是否需要修改工厂逻辑
- 注意工厂方法可能隐藏了对象创建的复杂度

### 3. 抽象工厂模式

**定义**：

创建一系列相关或依赖的对象家族，而无需指定具体类。就像家具工厂，一次生产配套的椅子、桌子和沙发。

**例子**：

UI主题工厂
```python
class Button:
    pass

class DarkButton(Button):
    def render(self):
        return "深色按钮"

class LightButton(Button):
    def render(self):
        return "浅色按钮"

class ThemeFactory:
    def create_button(self):
        pass

class DarkThemeFactory(ThemeFactory):
    def create_button(self):
        return DarkButton()

class LightThemeFactory(ThemeFactory):
    def create_button(self):
        return LightButton()

# 使用时
theme = "dark"
if theme == "dark":
    factory = DarkThemeFactory()
else:
    factory = LightThemeFactory()

button = factory.create_button()
print(button.render())
```

**代码审计思考**：
- 检查产品族是否真的相关需要一起创建
- 注意抽象工厂增加了系统的复杂度和理解成本
- 查看新增产品族时对现有代码的影响

### 4. 建造者（生成器）模式

**定义**：

将复杂对象的构建与表示分离，使同样的构建过程可以创建不同的表示。就像搭积木，同样的步骤可以搭出房子或城堡。

**例子**：

SQL查询构建器
```python
class SQLQueryBuilder:
    def __init__(self):
        self.query = ""
        
    def select(self, columns):
        self.query += f"SELECT {columns} "
        return self
        
    def from_table(self, table):
        self.query += f"FROM {table} "
        return self
        
    def where(self, condition):
        self.query += f"WHERE {condition} "
        return self
        
    def build(self):
        return self.query.strip() + ";"

# 使用时
builder = SQLQueryBuilder()
query = (builder
         .select("name, age")
         .from_table("users")
         .where("age > 18")
         .build())
print(query)  # SELECT name, age FROM users WHERE age > 18;
```

**代码审计思考**：
- 检查构建步骤是否合理，顺序是否正确
- 注意构建过程是否完整（可能缺少必要步骤）
- 查看构建器是否支持不同的产品变体

### 5. 原型模式

**定义**：

通过复制现有对象来创建新对象，而不是新建。就像复印机，复制已有文档而不是重写一遍。

**例子**：

游戏角色配置
```python
import copy

class CharacterConfig:
    def __init__(self, health, speed, damage):
        self.health = health
        self.speed = speed
        self.damage = damage
        
    def clone(self):
        return copy.deepcopy(self)
    
    def __str__(self):
        return f"生命:{self.health}, 速度:{self.speed}, 伤害:{self.damage}"

# 使用时
warrior_template = CharacterConfig(100, 5, 20)
archer = warrior_template.clone()
archer.speed = 8  # 弓箭手更快
archer.damage = 15  # 但伤害较低

print(f"战士模板: {warrior_template}")
print(f"弓箭手: {archer}")
```

**代码审计思考**：
- 检查深拷贝和浅拷贝的使用是否恰当
- 注意克隆可能包含不应复制的状态（如ID、连接等）
- 查看原型注册表的管理是否合理

## 结构型模式

结构型模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效。

### 1. 适配器模式

**定义**：

让接口不兼容的对象能够相互合作。就像电源适配器，让美标插头能在中式插座上使用。

**例子**：

第三方支付接口适配
```python
# 旧的支付接口
class LegacyPayment:
    def make_payment(self, amount_in_dollars):
        print(f"支付 ${amount_in_dollars}")

# 新的支付系统（期望人民币）
class NewPaymentSystem:
    def pay(self, amount_in_yuan):
        print(f"支付 ¥{amount_in_yuan}")

# 适配器
class PaymentAdapter:
    def __init__(self, legacy_payment):
        self.legacy_payment = legacy_payment
    
    def pay(self, amount_in_yuan):
        # 人民币转美元（简化汇率）
        amount_in_dollars = amount_in_yuan / 7
        self.legacy_payment.make_payment(amount_in_dollars)

# 使用时
legacy = LegacyPayment()
adapter = PaymentAdapter(legacy)
adapter.pay(700)  # 支付¥700，内部转为$100
```

**代码审计思考**：
- 检查适配器是否增加了不必要的复杂性
- 注意适配器可能隐藏了接口不匹配的问题
- 查看是否应该重构接口而不是使用适配器

### 2. 桥接模式

**定义**：

将抽象与实现分离，使它们可以独立变化。就像遥控器和电视，不同遥控器可以控制不同电视。

**例子**：

形状与渲染器
```python
class Renderer:
    def render_circle(self, radius):
        pass

class VectorRenderer(Renderer):
    def render_circle(self, radius):
        print(f"绘制矢量圆，半径: {radius}")

class RasterRenderer(Renderer):
    def render_circle(self, radius):
        print(f"绘制光栅圆，半径: {radius}")

class Shape:
    def __init__(self, renderer):
        self.renderer = renderer
    
    def draw(self):
        pass

class Circle(Shape):
    def __init__(self, renderer, radius):
        super().__init__(renderer)
        self.radius = radius
    
    def draw(self):
        self.renderer.render_circle(self.radius)

# 使用时
vector = VectorRenderer()
raster = RasterRenderer()

circle1 = Circle(vector, 5)
circle1.draw()  # 绘制矢量圆

circle2 = Circle(raster, 10)
circle2.draw()  # 绘制光栅圆
```

**代码审计思考**：
- 检查抽象和实现是否真的需要独立变化
- 注意桥接模式可能增加系统理解难度
- 查看桥接是否过度设计（简单情况可用继承）

### 3. 组合模式

**定义**：

将对象组合成树形结构以表示"部分-整体"层次，使客户端统一对待单个对象和组合对象。就像文件和文件夹，都可以进行"打开"操作。

**例子**：

组织架构
```python
class Employee:
    def __init__(self, name, position):
        self.name = name
        self.position = position
        self.subordinates = []
    
    def add(self, employee):
        self.subordinates.append(employee)
    
    def show_details(self, indent=0):
        print("  " * indent + f"{self.name} - {self.position}")
        for subordinate in self.subordinates:
            subordinate.show_details(indent + 1)

# 使用时
ceo = Employee("张三", "CEO")
cto = Employee("李四", "CTO")
dev1 = Employee("王五", "开发工程师")
dev2 = Employee("赵六", "开发工程师")

cto.add(dev1)
cto.add(dev2)
ceo.add(cto)

ceo.show_details()
# 输出:
# 张三 - CEO
#   李四 - CTO
#     王五 - 开发工程师
#     赵六 - 开发工程师
```

**代码审计思考**：
- 检查组合结构是否过于复杂
- 注意叶节点和组合节点的区别处理
- 查看是否存在循环引用风险

### 4. 装饰器模式

**定义**：

动态地为对象添加新功能，而不改变其结构。就像给手机加保护壳，不改变手机本身，但增加了保护功能。

**例子**：

文本格式化
```python
class Text:
    def render(self):
        return "普通文本"

class TextDecorator:
    def __init__(self, text):
        self.text = text
    
    def render(self):
        return self.text.render()

class BoldDecorator(TextDecorator):
    def render(self):
        return f"<b>{self.text.render()}</b>"

class ItalicDecorator(TextDecorator):
    def render(self):
        return f"<i>{self.text.render()}</i>"

# 使用时
simple_text = Text()
bold_text = BoldDecorator(simple_text)
bold_italic_text = ItalicDecorator(bold_text)

print(simple_text.render())      # 普通文本
print(bold_text.render())        # <b>普通文本</b>
print(bold_italic_text.render()) # <i><b>普通文本</b></i>
```

**代码审计思考**：
- 检查装饰顺序是否正确
- 注意装饰器链可能过长影响性能
- 查看是否应该使用继承替代简单装饰

### 5. 外观模式

**定义**：

为复杂子系统提供一个简单接口。就像酒店前台，客人不需要知道后厨、清洁、维修等细节，只需联系前台。

**例子**：

家庭影院系统
```python
class DVDPlayer:
    def on(self):
        print("DVD播放器打开")
    
    def play(self, movie):
        print(f"播放电影: {movie}")

class Projector:
    def on(self):
        print("投影仪打开")
    
    def wide_screen_mode(self):
        print("宽屏模式")

class HomeTheaterFacade:
    def __init__(self, dvd, projector):
        self.dvd = dvd
        self.projector = projector
    
    def watch_movie(self, movie):
        print("准备看电影...")
        self.projector.on()
        self.projector.wide_screen_mode()
        self.dvd.on()
        self.dvd.play(movie)
        print("电影开始!")

# 使用时
dvd = DVDPlayer()
projector = Projector()
theater = HomeTheaterFacade(dvd, projector)

theater.watch_movie("阿凡达")
```

**代码审计思考**：
- 检查外观是否隐藏了过多细节
- 注意外观可能成为新的"上帝对象"
- 查看子系统变化时对外观的影响

### 6. 享元模式

**定义**：

通过共享大量细粒度对象来节省内存。就像字处理软件，每个字符都是享元，只存储一次字形，多次引用。

**例子**：

棋盘游戏
```python
class TreeType:
    def __init__(self, name, color, texture):
        self.name = name
        self.color = color
        self.texture = texture
    
    def draw(self, x, y):
        print(f"在({x}, {y})绘制{self.color}的{self.name}")

class TreeFactory:
    _tree_types = {}
    
    @classmethod
    def get_tree_type(cls, name, color, texture):
        key = f"{name}_{color}_{texture}"
        if key not in cls._tree_types:
            cls._tree_types[key] = TreeType(name, color, texture)
        return cls._tree_types[key]

class Tree:
    def __init__(self, x, y, tree_type):
        self.x = x
        self.y = y
        self.type = tree_type
    
    def draw(self):
        self.type.draw(self.x, self.y)

# 使用时
factory = TreeFactory

# 创建1000棵树，但只有2种类型
pine = factory.get_tree_type("松树", "绿色", "松针纹理")
oak = factory.get_tree_type("橡树", "棕色", "橡树纹理")

trees = []
for i in range(500):
    trees.append(Tree(i, i, pine))
    trees.append(Tree(i+1, i, oak))

for tree in trees[:5]:  # 只画前5棵
    tree.draw()
```

**代码审计思考**：
- 检查享元对象是否真的不可变
- 注意享元工厂的内存管理
- 查看共享状态和外部状态的分离是否清晰

### 7. 代理模式

**定义**：

为其他对象提供一个代理以控制对这个对象的访问。就像明星经纪人，控制外界对明星的访问。

**例子**：

图片懒加载
```python
class Image:
    def __init__(self, filename):
        self.filename = filename
        self._load_image()
    
    def _load_image(self):
        print(f"加载大图片: {self.filename}")
    
    def display(self):
        print(f"显示图片: {self.filename}")

class ImageProxy:
    def __init__(self, filename):
        self.filename = filename
        self._image = None
    
    def display(self):
        if self._image is None:
            self._image = Image(self.filename)
        self._image.display()

# 使用时
image1 = ImageProxy("photo1.jpg")
image2 = ImageProxy("photo2.jpg")

# 图片不会立即加载
image1.display()  # 第一次调用时加载
image1.display()  # 已经加载过，直接显示
image2.display()  # 第一次调用时加载
```

**代码审计思考**：
- 检查代理是否增加了不必要的间接层
- 注意代理可能隐藏性能问题
- 查看代理的权限控制是否合理

## 行为模式

行为模式负责对象间的高效沟通和职责委派。

### 1. 责任链模式

**定义**：

将请求沿着处理链传递，直到有对象处理它。就像公司请假流程，从组长到经理再到总监。

**例子**：

日志级别处理
```python
class Logger:
    def __init__(self, level):
        self.level = level
        self.next_logger = None
    
    def set_next(self, logger):
        self.next_logger = logger
        return logger
    
    def log_message(self, level, message):
        if self.level <= level:
            self.write(message)
        if self.next_logger is not None:
            self.next_logger.log_message(level, message)
    
    def write(self, message):
        pass

class ErrorLogger(Logger):
    def __init__(self):
        super().__init__(1)  # ERROR级别
    
    def write(self, message):
        print(f"错误日志: {message}")

class DebugLogger(Logger):
    def __init__(self):
        super().__init__(2)  # DEBUG级别
    
    def write(self, message):
        print(f"调试日志: {message}")

# 使用时
error_logger = ErrorLogger()
debug_logger = DebugLogger()

error_logger.set_next(debug_logger)

error_logger.log_message(1, "系统错误!")
error_logger.log_message(2, "调试信息")
```

**代码审计思考**：
- 检查责任链是否可能无限循环
- 注意请求可能未被任何处理器处理
- 查看链的配置是否正确

### 2. 命令模式

**定义**：

将请求封装为对象，从而支持参数化、队列化、撤销等操作。就像餐厅点单，服务员不需要知道如何做菜，只需传递订单。

**例子**：

文本编辑器命令
```python
class Document:
    def __init__(self):
        self.content = ""
    
    def insert_text(self, text, position):
        self.content = self.content[:position] + text + self.content[position:]
        print(f"插入文本: {text}")
    
    def delete_text(self, position, length):
        deleted = self.content[position:position+length]
        self.content = self.content[:position] + self.content[position+length:]
        print(f"删除文本: {deleted}")

class Command:
    def execute(self):
        pass
    
    def undo(self):
        pass

class InsertCommand(Command):
    def __init__(self, document, text, position):
        self.document = document
        self.text = text
        self.position = position
    
    def execute(self):
        self.document.insert_text(self.text, self.position)
    
    def undo(self):
        self.document.delete_text(self.position, len(self.text))

# 使用时
doc = Document()
commands = []

# 执行命令
cmd1 = InsertCommand(doc, "Hello", 0)
cmd1.execute()
commands.append(cmd1)

cmd2 = InsertCommand(doc, " World", 5)
cmd2.execute()
commands.append(cmd2)

print(f"文档内容: {doc.content}")

# 撤销最后一个命令
if commands:
    last_cmd = commands.pop()
    last_cmd.undo()
    print(f"撤销后: {doc.content}")
```

**代码审计思考**：
- 检查命令对象是否过大（存储过多状态）
- 注意命令队列的内存管理
- 查看撤销/重做逻辑是否正确

### 3. 迭代器模式

**定义**：

提供一种方法顺序访问聚合对象的元素，而不暴露其内部表示。就像遥控器换台，不需要知道电视内部如何工作。

**例子**：

自定义集合迭代
```python
class Book:
    def __init__(self, title):
        self.title = title

class BookShelf:
    def __init__(self):
        self._books = []
    
    def add_book(self, book):
        self._books.append(book)
    
    def __iter__(self):
        return BookShelfIterator(self)

class BookShelfIterator:
    def __init__(self, book_shelf):
        self._book_shelf = book_shelf
        self._index = 0
    
    def __next__(self):
        if self._index < len(self._book_shelf._books):
            book = self._book_shelf._books[self._index]
            self._index += 1
            return book
        raise StopIteration

# 使用时
shelf = BookShelf()
shelf.add_book(Book("设计模式"))
shelf.add_book(Book("代码整洁之道"))
shelf.add_book(Book("重构"))

for book in shelf:
    print(f"书籍: {book.title}")
```

**代码审计思考**：
- 检查迭代器是否支持并发修改
- 注意迭代器的资源释放
- 查看是否应该使用语言内置的迭代机制

### 4. 中介者模式

**定义**：

用一个中介对象封装一系列对象交互，使对象间不需要显式相互引用。就像机场控制塔，协调所有飞机的起降。

**例子**：

聊天室
```python
class User:
    def __init__(self, name, mediator):
        self.name = name
        self.mediator = mediator
    
    def send(self, message):
        print(f"{self.name} 发送: {message}")
        self.mediator.send_message(message, self)
    
    def receive(self, message):
        print(f"{self.name} 收到: {message}")

class ChatRoom:
    def __init__(self):
        self.users = []
    
    def add_user(self, user):
        self.users.append(user)
    
    def send_message(self, message, sender):
        for user in self.users:
            if user != sender:
                user.receive(message)

# 使用时
chat_room = ChatRoom()

alice = User("Alice", chat_room)
bob = User("Bob", chat_room)
charlie = User("Charlie", chat_room)

chat_room.add_user(alice)
chat_room.add_user(bob)
chat_room.add_user(charlie)

alice.send("大家好!")
bob.send("你好Alice!")
```

**代码审计思考**：
- 检查中介者是否过于复杂（成为新的"上帝对象"）
- 注意中介者可能成为性能瓶颈
- 查看对象间通信是否真的需要中介

### 5. 备忘录模式

**定义**：

在不破坏封装的前提下，捕获对象内部状态并在外部保存，以便后续恢复。就像游戏存档，保存当前进度，随时可以读档。

**例子**：

文本编辑器状态保存
```python
class EditorState:
    def __init__(self, content):
        self.content = content

class TextEditor:
    def __init__(self):
        self.content = ""
    
    def set_content(self, content):
        self.content = content
    
    def get_content(self):
        return self.content
    
    def create_state(self):
        return EditorState(self.content)
    
    def restore_state(self, state):
        self.content = state.content

class History:
    def __init__(self):
        self.states = []
    
    def push(self, state):
        self.states.append(state)
    
    def pop(self):
        if self.states:
            return self.states.pop()
        return None

# 使用时
editor = TextEditor()
history = History()

editor.set_content("第一行")
history.push(editor.create_state())

editor.set_content("第二行")
history.push(editor.create_state())

editor.set_content("第三行")
print(f"当前内容: {editor.get_content()}")  # 第三行

# 撤销
state = history.pop()
if state:
    editor.restore_state(state)
    print(f"撤销后: {editor.get_content()}")  # 第二行
```

**代码审计思考**：
- 检查备忘录可能占用大量内存
- 注意状态序列化和反序列化的安全性
- 查看状态恢复是否完整

### 6. 观察者模式

**定义**：

定义对象间的一对多依赖关系，当一个对象状态改变时，所有依赖它的对象都会收到通知。就像微信公众号，订阅者收到新文章通知。

**例子**：

股票价格监控
```python
class Stock:
    def __init__(self, symbol, price):
        self.symbol = symbol
        self._price = price
        self._investors = []
    
    @property
    def price(self):
        return self._price
    
    @price.setter
    def price(self, value):
        if self._price != value:
            self._price = value
            self.notify_investors()
    
    def attach(self, investor):
        self._investors.append(investor)
    
    def notify_investors(self):
        for investor in self._investors:
            investor.update(self)

class Investor:
    def __init__(self, name):
        self.name = name
    
    def update(self, stock):
        print(f"{self.name}: {stock.symbol} 价格变为 {stock.price}")

# 使用时
apple = Stock("AAPL", 150)

investor1 = Investor("张三")
investor2 = Investor("李四")

apple.attach(investor1)
apple.attach(investor2)

apple.price = 155  # 所有投资者收到通知
```

**代码审计思考**：
- 检查观察者是否及时取消注册（避免内存泄漏）
- 注意通知顺序是否重要
- 查看是否存在循环通知问题

### 7. 状态模式

**定义**：

允许对象在内部状态改变时改变其行为，对象看起来好像修改了它的类。就像水在不同温度下呈现不同状态（固态、液态、气态）。

**例子**：

订单状态
```python
class OrderState:
    def process(self, order):
        pass
    
    def cancel(self, order):
        pass

class NewOrderState(OrderState):
    def process(self, order):
        print("新订单处理中...")
        order.set_state(ProcessingOrderState())
    
    def cancel(self, order):
        print("订单已取消")
        order.set_state(CancelledOrderState())

class ProcessingOrderState(OrderState):
    def process(self, order):
        print("订单处理完成!")
        order.set_state(CompletedOrderState())
    
    def cancel(self, order):
        print("处理中的订单无法取消")

class CompletedOrderState(OrderState):
    def process(self, order):
        print("订单已完成，无法再次处理")
    
    def cancel(self, order):
        print("订单已完成，无法取消")

class Order:
    def __init__(self):
        self.state = NewOrderState()
    
    def set_state(self, state):
        self.state = state
    
    def process(self):
        self.state.process(self)
    
    def cancel(self):
        self.state.cancel(self)

# 使用时
order = Order()
order.process()  # 新订单处理中...
order.process()  # 订单处理完成!
order.cancel()   # 订单已完成，无法取消
```

**代码审计思考**：
- 检查状态转换是否正确完整
- 注意状态对象是否应该是无状态的（可共享）
- 查看状态是否过多导致复杂

### 8. 策略模式

**定义**：

定义一系列算法，封装每个算法，并使它们可以相互替换。就像出行方式，根据情况选择公交、开车或步行。

**例子**：

支付策略
```python
class PaymentStrategy:
    def pay(self, amount):
        pass

class CreditCardPayment(PaymentStrategy):
    def __init__(self, card_number):
        self.card_number = card_number
    
    def pay(self, amount):
        print(f"信用卡支付 {amount}元，卡号: {self.card_number[-4:]}")

class AlipayPayment(PaymentStrategy):
    def __init__(self, account):
        self.account = account
    
    def pay(self, amount):
        print(f"支付宝支付 {amount}元，账户: {self.account}")

class ShoppingCart:
    def __init__(self):
        self.items = []
        self.payment_strategy = None
    
    def add_item(self, item, price):
        self.items.append((item, price))
    
    def set_payment_strategy(self, strategy):
        self.payment_strategy = strategy
    
    def checkout(self):
        total = sum(price for _, price in self.items)
        print(f"总计: {total}元")
        if self.payment_strategy:
            self.payment_strategy.pay(total)
        else:
            print("请选择支付方式")

# 使用时
cart = ShoppingCart()
cart.add_item("手机", 2999)
cart.add_item("耳机", 399)

cart.set_payment_strategy(CreditCardPayment("1234-5678-9876-5432"))
cart.checkout()
```

**代码审计思考**：
- 检查策略是否真的可以互相替换
- 注意客户端是否知道选择合适的策略
- 查看策略对象是否应该是无状态的

### 9. **模板方法模式

**定义**：

在父类中定义算法的骨架，将某些步骤延迟到子类实现。就像泡茶和泡咖啡，流程相同（烧水、冲泡、倒入杯中），但具体步骤不同。

**例子**：

数据导入流程
```python
from abc import ABC, abstractmethod

class DataImporter(ABC):
    def import_data(self):
        """模板方法：定义导入流程"""
        self.validate_file()
        self.parse_data()
        self.validate_data()
        self.save_to_database()
        self.cleanup()
    
    def validate_file(self):
        print("验证文件格式...")
    
    @abstractmethod
    def parse_data(self):
        pass
    
    def validate_data(self):
        print("验证数据完整性...")
    
    def save_to_database(self):
        print("保存到数据库...")
    
    def cleanup(self):
        print("清理临时文件...")

class CSVImporter(DataImporter):
    def parse_data(self):
        print("解析CSV文件...")

class ExcelImporter(DataImporter):
    def parse_data(self):
        print("解析Excel文件...")

# 使用时
csv_importer = CSVImporter()
csv_importer.import_data()

print("\n---\n")

excel_importer = ExcelImporter()
excel_importer.import_data()
```

**代码审计思考**：
- 检查模板方法是否过于严格（限制子类灵活性）
- 注意钩子方法的使用是否合理
- 查看是否应该使用策略模式替代

### 10. 访问者模式

**定义**：

将算法与对象结构分离，在不修改对象结构的情况下为对象添加新操作。就像税务审计员，访问不同公司进行不同的审计。

**例子**：

文档元素导出
```python
class DocumentElement:
    def accept(self, visitor):
        pass

class TextElement(DocumentElement):
    def __init__(self, content):
        self.content = content
    
    def accept(self, visitor):
        visitor.visit_text(self)

class ImageElement(DocumentElement):
    def __init__(self, src, alt):
        self.src = src
        self.alt = alt
    
    def accept(self, visitor):
        visitor.visit_image(self)

class Visitor:
    def visit_text(self, text):
        pass
    
    def visit_image(self, image):
        pass

class HTMLExportVisitor(Visitor):
    def visit_text(self, text):
        print(f"<p>{text.content}</p>")
    
    def visit_image(self, image):
        print(f'<img src="{image.src}" alt="{image.alt}">')

class PlainTextExportVisitor(Visitor):
    def visit_text(self, text):
        print(text.content)
    
    def visit_image(self, image):
        print(f"[图片: {image.alt}]")

# 使用时
elements = [
    TextElement("欢迎访问我们的网站"),
    ImageElement("logo.png", "公司Logo"),
    TextElement("感谢您的支持")
]

html_visitor = HTMLExportVisitor()
plain_visitor = PlainTextExportVisitor()

print("HTML导出:")
for element in elements:
    element.accept(html_visitor)

print("\n纯文本导出:")
for element in elements:
    element.accept(plain_visitor)
```

**代码审计思考**：
- 检查访问者是否破坏了封装（需要暴露内部细节）
- 注意新增元素类型时对访问者的影响
- 查看是否应该使用迭代器或组合模式替代

[^1]: Refactoring Guru https://refactoringguru.cn/design-patterns/catalog
[^2]: DeepSeek AI辅助生成