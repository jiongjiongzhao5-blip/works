# 作业1：类关系、UML、SOLID设计原则

## 1. 类与类之间的关系有哪几种？从语义（is / use / has）和耦合程度两个维度分别说明，并按耦合度从低到高排序。

**原题：**
> 简答：类与类之间的关系有哪几种？从语义（is / use / has）和耦合程度两个维度分别说明，并按耦合度从低到高排序。

**答案：**

UML 中类与类之间的关系共 **6 种**（依赖、关联、聚合、组合、实现、继承）。从语义（is / use / has）和耦合度两个维度归纳如下：

| 关系                       | 语义                           | 生命周期/所有权                        | 耦合程度 |
| -------------------------- | ------------------------------ | -------------------------------------- | -------- |
| 依赖 Dependency            | **use-a**（临时使用）          | 无持有关系，仅函数参数/局部变量/返回值 | 最低     |
| 关联 Association           | **has-a / use-a**（长期认识）  | 双方各自独立，持有对方指针/引用        | 较低     |
| 聚合 Aggregation           | **has-a**（整体-部分，弱拥有） | 整体不负责部分的生命周期，"同生不共死" | 中       |
| 组合 Composition           | **has-a**（整体-部分，强拥有） | 整体负责部分的生命周期，"同生共死"     | 较高     |
| 实现 Realization           | **is-a**（接口契约）           | 类实现接口，耦合到接口抽象             | 高       |
| 继承（泛化）Generalization | **is-a**（"是…一种"）          | 派生类强耦合基类实现                   | 最高     |

**按耦合度从低到高排序：**

```
依赖 < 关联 < 聚合 < 组合 < 实现 < 继承
```

**通俗解释：**
- **依赖**：就像你去餐厅"用"了筷子，吃完饭筷子跟你没关系了——最弱。
- **关联**：你"认识"你的同事，长期保持联系，但彼此独立生活。
- **聚合**：大雁群里有大雁，大雁离开雁群照样活——整体不负责部分的生死。
- **组合**：人由心脏组成，心脏离开人就没法独立存在——整体负责部分生死。
- **实现/继承**：你"是"人类（实现"会说话"这个接口、继承"人类"的所有属性）——血缘关系最强，改爹影响儿子，耦合最高。

**设计启示**：能用依赖/关联解决就不要用继承；能用组合就不要用继承（这正好呼应下面的"组合复用原则"）。

---

## 2. 画出继承、关联（双向）、聚合、组合、依赖的UML类图示意

**原题：**
> 画图题：画出继承、关联（双向）、聚合、组合、依赖的UML类图示意（用箭头、菱形等符号）。

**答案：**

UML 符号约定：**实线 + 空心三角 = 继承（泛化）**；**虚线 + 空心三角 = 实现**；**实线 + 空心菱形 = 聚合**；**实线 + 实心菱形 = 组合**；**虚线 + 普通箭头 = 依赖**；**实线无箭头/两端箭头 = 关联**。

**① 继承（实线 + 空心三角箭头指向基类）**
```
        ┌──────────┐
        │  Shape   │            ← 基类
        └──────────┘
             ▲
             │  实线 + 空心三角
   ┌─────────┴─────────┐
   │                   │
┌───────┐          ┌───────┐
│ Rect  │          │ Circle│   ← 派生类
└───────┘          └───────┘
```

**② 关联-双向（实线，两端各带普通箭头，或两端不带箭头）**
```
┌─────────┐                   ┌─────────┐
│  Student│◄─────────────────►│ Teacher │
└─────────┘   选课/授课双向引用 └─────────┘
```
（`1..*` 等数字标在连线两端表示多重性：一个老师教多个学生等。）

**③ 聚合（实线 + 空心菱形，菱形画在"整体"一端）**
```
┌─────────┐                ┌─────────┐
│   Car   │◇───────────────│  Wheel  │
└─────────┘   1        4   └─────────┘
```
（整车"拥有"车轮，但车轮拆下来还能单独存在 → 空心菱形。）

**④ 组合（实线 + 实心菱形，菱形画在"整体"一端）**
```
┌─────────┐                ┌─────────┐
│  House  │◆───────────────│  Room   │
└─────────┘   1         *  └─────────┘
```
（房间离开房子就无意义，和房子同生共死 → 实心菱形。）

**⑤ 依赖（虚线 + 普通箭头指向被依赖者）**
```
┌─────────┐             ┌─────────┐
│   A     │┄┄┄┄┄┄┄┄┄┄┄▶│    B    │
└─────────┘   A 的函数参数使用了 B
```

**通俗解释**：判断"菱形"还是"三角"就看一句话——"能不能单独活"。部分离开整体还能活（轮胎、大雁）→ 空心菱形（聚合）；离开整体就死（房间、心脏）→ 实心菱形（组合）。

---

## 3. 代码辨析：判断下列代码属于哪种类间关系，并说明理由

**原题：**
> 代码辨析：给出以下代码片段，判断属于哪种类间关系，并说明理由：
> - `class B; class A { B* b; };`
> - `class A { void func(B b); };`
> - `class A { B b; };`
> - `class A : public B {};`

**答案：**

**(a) `class B; class A { B* b; };` → 关联 / 聚合（has-a，弱持有）**
理由：A 通过**裸指针**持有 B。指针可以指向任意生命周期、可以为空，A 不负责（也不必然负责）B 的创建与销毁。这与"整体认识部分但不管理其生死"的关联/聚合语义一致。具体是关联还是聚合要看 A 是否"拥有"B：若 b 由外部传入、A 只使用 → 关联；若 A 内部创建 B 且由 A 管理 → 聚合。

**(b) `class A { void func(B b); };` → 依赖（use-a，最弱）**
理由：B 仅作为**函数参数**被临时使用，函数结束后 A 与 B 不再有任何关系，A 也不持有 B 的任何成员。这是典型的"用到才认识"的依赖关系。

**(c) `class A { B b; };` → 组合（has-a，强拥有）**
理由：B 作为 A 的**值成员**，A 的构造会创建 B、A 的析构会自动销毁 B，二者生命周期严格一致（"同生共死"），这是最典型、最强的组合关系。

**(d) `class A : public B {};` → 继承（is-a）**
理由：A 公有继承 B，A 是 B 的一种，A 拥有 B 的全部公有接口并可以覆盖（override）其虚函数，这是明确的泛化/继承关系。

**通俗解释**：判断口诀——**看指针还是值、看是否长期持有、看是否负责生死**。指针成员≈关联/聚合（弱），值成员≈组合（强），函数参数≈依赖（最弱），冒号继承≈继承（is-a）。

---

## 4. 分别解释七个设计原则的核心思想，并解释"开闭原则是目标，里氏替换是基础，依赖倒置是手段"

**原题：**
> 原则阐述：分别解释单一职责、开闭原则、里氏替换、依赖倒置、接口分离、迪米特、组合复用原则的核心思想。为什么说"开闭原则是目标，里氏替换是基础，依赖倒置是手段"？

**答案：**

| 原则       | 英文缩写 | 核心思想                                                     | 一句话记忆       |
| ---------- | -------- | ------------------------------------------------------------ | ---------------- |
| 单一职责   | SRP      | 一个类只负责**一项**职责，只有一个引起它变化的原因           | 一个类只干一件事 |
| 开闭原则   | OCP      | 对**扩展**开放、对**修改**关闭：新增功能靠增加新类，不靠改老代码 | 加功能不改老代码 |
| 里氏替换   | LSP      | 派生类必须能**安全替换**基类，且不破坏程序正确性；派生类不能放宽基类的契约 | 儿子能替父亲干活 |
| 依赖倒置   | DIP      | 高层模块不依赖低层模块，二者都依赖**抽象**；抽象不依赖细节，细节依赖抽象 | 面向接口编程     |
| 接口分离   | ISP      | 客户端**不应该依赖它用不到的接口**，胖接口要拆成多个专用小接口 | 接口要"小而专"   |
| 迪米特法则 | LoD      | 一个对象应对其他对象保持**最少了解**，只与直接朋友通信       | 少管闲事         |
| 组合复用   | CRP      | **优先用组合**而非继承来复用功能                             | 能组合就不继承   |

**为什么说"开闭原则是目标，里氏替换是基础，依赖倒置是手段"？**

- **开闭原则是"目标"**：所有原则最终都是为了实现"对扩展开放、对修改关闭"这个终极目标——软件能优雅地增加新功能，而不破坏已有稳定代码。
- **里氏替换是"基础"**：要用多态实现开闭（比如新增一个子类替换基类就能扩展系统），前提是子类**必须能安全替换**基类。如果里氏替换被破坏（子类改了父类的行为契约），那么"扩展"就会变成"破坏"，开闭原则就无从谈起。所以 LSP 是 OCP 的地基。
- **依赖倒置是"手段"**：让代码**依赖抽象接口而不是具体实现**，正是实现开闭原则的具体手段。当新增功能时，只需新增一个实现抽象接口的类，客户端代码（依赖抽象）一行不改，就完成了"扩展不修改"。

**通俗解释**：好比"供电标准"——插座（抽象）规定了电压接口，任何新电器（新类）只要按标准做插头就能直接用（开闭）；前提是所有电器都遵守"插头插得进去、电不会烧坏"（里氏替换）；而"大家都按插座标准来设计"（依赖倒置）就是实现这一点的办法。

---

## 5. 下面代码违反哪些设计原则？如何修改？

**原题：**
> 代码违反原则分析：下面代码违反哪些设计原则？如何修改？
> ```cpp
> class GraphicEditor {
> public:
>  void drawShape(Shape* s) {
>      if (s->type == RECT) drawRect();
>      else if (s->type == CIRCLE) drawCircle();
>  }
> };
> ```

**答案：**

**违反的原则：**

1. **开闭原则（OCP）**——每次新增图形（三角形、椭圆…）都要回来改 `drawShape` 里的 if-else，违背"对扩展开放、对修改关闭"。
2. **单一职责（SRP）**——`GraphicEditor` 承担了**所有**图形的绘制逻辑，一旦某个图形的画法变化，都会导致这个类被修改（一个类有多个变化原因）。
3. **依赖倒置（DIP）**——`GraphicEditor` 依赖 `Shape` 的**具体类型标签**（`type` 枚举）来做分支，而不是依赖抽象行为接口（虚函数）。

**修改方案：用多态替代分支（策略 + 多态）**

```cpp
class Shape {
public:
    virtual void draw() const = 0;   // 抽象行为：每个图形自己会画
    virtual ~Shape() = default;
};

class Rect : public Shape {
public:
    void draw() const override { /* 画矩形 */ }
};

class Circle : public Shape {
public:
    void draw() const override { /* 画圆形 */ }
};

class GraphicEditor {
public:
    void drawShape(Shape* s) {
        s->draw();      // 一个调用，完事！新增图形只加新类，这里永不修改
    }
};
```

**通俗解释**：原来的写法像"前台总机"，每来一种图形就让你手动转接一次；改成多态后，**每个图形自己知道怎么画自己**，前台只负责一句"请开始你的表演"。以后新增三角形，只新建 `Triangle` 类、实现 `draw()`，编辑器代码一行不动——开闭原则达成。

---

## 6. 里氏替换陷阱：基类有 `void consume(int)` 非虚函数，派生类定义了同名的 `void consume(int)`（非覆盖）

**原题：**
> 里氏替换陷阱：基类有 `void consume(int)` 非虚函数，派生类定义了同名的 `void consume(int)`（非覆盖）。为什么说这违反了里氏替换？应如何避免？

**答案：**

**为什么会违反里氏替换：**

基类的 `consume(int)` 是**非虚**函数，派生类再定义一个同名同参的 `consume(int)`，这不是"覆盖"（override），而是 **名字隐藏（name hiding）**。后果是**同一个对象的同一个调用，会因"静态类型"不同而得到不同行为**：

```cpp
struct Base  { void consume(int n) { std::cout << "Base::consume\n"; } };
struct Derived : Base { void consume(int n) { std::cout << "Derived::consume\n"; } };

Derived d;
d.consume(1);        // 输出 Derived::consume   （静态类型 Derived）
Base* pb = &d;
pb->consume(1);      // 输出 Base::consume      （静态类型 Base，但对象明明是 Derived！）
```

里氏替换要求：**凡是能用基类对象的地方，用派生类对象替换后行为应当一致、不破坏正确性**。而这里"同一个 `Derived` 对象"在通过基类指针访问时悄悄换了实现，程序员的意图（覆盖）与真实行为（隐藏）不一致，产生隐蔽 bug——这恰恰是 LSP 的破坏（派生类不能按基类契约一致工作）。

**如何避免：**

1. **要多态，就明确虚函数 + `override`**：把基类函数声明为 `virtual`，派生类用 `virtual ... override` 显式标注，让编译器帮你检查。
2. **不要多态，就改名**：派生类若不想覆盖基类行为，就不要定义同名同参函数，换个名字或加参数。
3. **用 `final` 封死误覆盖**：基类可以 `virtual void consume(int) final`，禁止派生类再覆盖。
4. **开编译器警告**：`-Woverloaded-virtual`（GCC/Clang）能在派生类隐藏基类虚函数时给出警告。
5. **用 `override` 关键字**是 C++11 的硬性语法检查：基类没有对应虚函数，编译直接报错。

**通俗解释**：就像"爸爸的手机号打过去是爸爸接，儿子自称继承了爸爸的号码，但你拨他（以为是他爸）时接电话的却是另一个人"——同一件事换个角度看行为就变了，这就是里氏替换被破坏。正确做法是：要么真的办"呼叫转移"（虚函数 + override），要么儿子别用同一个号码（改名）。

---

# 作业2：工厂模式 & 观察者模式

## 1. 画出简单工厂模式、工厂方法模式的类图，并标注每个角色

**原题：**
> 类图绘制：画出简单工厂模式、工厂方法模式的类图，并标注每个角色。

**答案：**

**① 简单工厂模式（Simple Factory）**

```
                      ┌────────────────────┐
                      │   SimpleFactory    │  ← 工厂角色（负责 if/switch 创建产品）
                      │ + create(type)     │
                      └────────────────────┘
                                 │ 创建并返回
                                 ▼
                      ┌────────────────────┐
                      │      Shape         │  ← 抽象产品角色（接口）
                      │ + virtual draw()=0 │
                      └────────────────────┘
                           ▲           ▲
                           │           │  实现
                 ┌─────────┴───┐   ┌───┴──────────┐
                 │   Circle    │   │  Rectangle   │  ← 具体产品角色
                 └─────────────┘   └──────────────┘

   Client(客户端) → 调用 SimpleFactory::create("circle")，依赖工厂而非具体产品
```

**② 工厂方法模式（Factory Method）**

```
   ┌─────────────────────┐                ┌─────────────────────┐
   │     Factory         │                │      Product        │  ← 抽象产品
   │ (抽象工厂/接口)      │──依赖──────▶   │ + virtual use()=0   │
   │ + virtual create()=0│                └─────────────────────┘
   └─────────────────────┘                       ▲
             ▲  实现                              │  实现
             │                              ┌─────┴─────┐
   ┌─────────────────────┐                  │  Concrete │
   │ ConcreteFactory     │  创建            │  Product  │  ← 具体产品
   │ + create() override │────────────────▶└───────────┘
   └─────────────────────┘   （返回具体产品指针）
            角色说明：抽象工厂/具体工厂/抽象产品/具体产品/客户端(Client)
```

**通俗解释**：简单工厂是"一个总厂按名字造各种货"（工厂类职责重，加货要改总厂）；工厂方法是"每种货配一个分厂"（加货只需新开分厂 + 新货，老分厂不动）——这就是两种模式最核心的差别。

---

## 2. 从设计原则角度分析简单工厂模式的三大缺点；工厂方法如何解决？工厂方法的新缺点是什么？

**原题：**
> 优缺点对比：从设计原则角度，分析简单工厂模式的三大缺点；工厂方法模式如何解决这些缺点？工厂方法的新缺点是什么？

**答案：**

**简单工厂模式三大缺点（从设计原则角度）：**

1. **违反开闭原则（OCP）**：新增产品必须回到工厂类的 if-else/switch 里加一个分支，**改已有代码**才能扩展。
2. **违反单一职责（SRP）/ 工厂职责过重**：所有产品的创建逻辑集中在一个工厂类里，一个工厂承担了 N 种产品的创建职责，改动频繁、易出错。
3. **客户端与具体产品仍然耦合**：客户端必须知道产品"名字"（字符串/枚举），并且要自己处理返回类型转换，并没有真正做到完全面向抽象。

**工厂方法如何解决这三大缺点：**

1. **满足开闭原则**：新增产品 = 新增一个具体产品类 + 新增一个对应的具体工厂类，**老工厂代码一行不改**，通过"多态延迟"把创建逻辑下沉到子类。
2. **职责单一**：每个具体工厂只负责创建一种产品，各工厂互不影响。
3. **面向抽象**：客户端依赖抽象工厂 `Factory` 和抽象产品 `Product`，不需要知道具体产品类型。

**工厂方法的新缺点：**

1. **类数量爆炸**：每增加一种产品就要配套增加一个具体工厂类，类个数翻倍，系统结构膨胀、维护成本上升。
2. **引入抽象层后更复杂**：增加了工厂抽象接口，简单场景下属于**过度设计**。
3. **客户端仍需知道具体工厂类**：工厂方法只是把"创建逻辑"延迟，客户端选择"哪个具体工厂"这一步骤依然存在（通常靠配置文件/注册表解决）。

**通俗解释**：简单工厂像"中央食堂"——所有菜谱集中在一个厨师手里，上新菜要改总菜谱（违反开闭、职责重）；工厂方法像"每个菜一个独立小店"——上新菜就是新开一家店，老店不动（满足开闭），但店的数量翻倍了（类爆炸）。

---

## 3. 场景选型：多数据库系统，未来可能新增数据库类型，且每个数据库操作也可能新增版本，应选哪种工厂模式？

**原题：**
> 场景选型：有一个系统需要支持多种数据库（MySQL、Oracle、SQLite），且每种数据库都有连接、查询、事务等操作。如果未来可能新增数据库类型，同时每个数据库内部的操作也可能增加新版本（如MySQL 8.0特有功能），应选用哪种工厂模式？画出类图简要说明。

**答案：**

**选型：抽象工厂模式（Abstract Factory）为主，配合"工厂方法 + 配置文件/注册表"处理版本维度。**

原因：这里有**两个变化轴**——
- **轴1：数据库类型**（MySQL / Oracle / SQLite）——每个类型是一组**相关的产品族**（连接、查询命令、事务），这正好是抽象工厂的用武之地：每个"具体工厂"（MySQLFactory、OracleFactory…）负责创建某一族内的全部相关产品。
- **轴2：数据库内部的操作/版本**（如 MySQL 8.0 特有功能）——这属于**同一族内部的纵向扩展**，用**工厂方法**在族内继续延迟，或把"版本特性"做成**策略/装饰**，并用**配置文件 + 注册表**在运行期选择，避免为"类型×版本"建立 M×N 个工厂类。

```
   ┌──────────────────────┐  创建族    ┌──────────────────┐
   │   DBFactory(抽象工厂) │──────────▶│   Connection(抽象)│
   │ + createConnection() │           │ + connect()=0    │
   │ + createCommand()    │           └──────────────────┘
   │ + createTransaction()│           ┌──────────────────┐
   └──────────────────────┘        ┌▶│   Command(抽象)   │
             ▲                      │ │ + execute()=0    │
             │ 实现                  │ └──────────────────┘
   ┌──────────────────────┐         │ ┌──────────────────┐
   │  MySQLFactory        │─────────┘▶│  Transaction(抽象)│
   │  OracleFactory       │           └──────────────────┘
   │  SQLiteFactory       │
   └──────────────────────┘
       每个具体工厂负责为"某一种数据库"创建一族产品：
       MySQLFactory → MySQLConnection / MySQLCommand / MySQLTransaction
       新增数据库类型 = 新增一个具体工厂 + 一族具体产品（满足开闭原则，不改旧代码）
       版本功能（MySQL 8.0 特有）→ 在该族内部用工厂方法/策略扩展
```

**通俗解释**：抽象工厂解决"一整族相关产品一起造、还要能整体换族"的问题。加一种数据库就像"加一个品牌的全套家电生产线"；而某个品牌内部升级型号（8.0 特有功能），则是在这条生产线内部加"新工艺"，不需要再造一条线。

---

## 4. 观察者模式：画出类图，说明 Subject 和 Observer 的关系；C++ 中如何用智能指针避免生命周期问题？

**原题：**
> 观察者模式：画出观察者模式的类图，说明Subject和Observer的关系。在C++中，如何用智能指针避免观察者生命周期问题？

**答案：**

**类图：**

```
    ┌───────────────────────────┐        ┌──────────────────────┐
    │ Subject（被观察者/主题）     │        │   Observer（观察者）    │
    │ - observers: List<Observer>│        │ + virtual update()=0  │
    │ + attach(obs)             │        └──────────────────────┘
    │ + detach(obs)             │                 ▲
    │ + notify()                │                 │ 实现
    │ ──────────────────────────│        ┌──────────────────────┐
    │ notify: 遍历 observers，  │◀───────│  ConcreteObserver    │
    │   调用每个 obs->update()  │  注册    │  + update() override  │
    └───────────────────────────┘        └──────────────────────┘
    关系：Subject 持有 Observer 列表（关联），Observer 不持有 Subject；
         Subject 是"一"，Observer 是"多"（一对多，依赖倒置：双方都依赖抽象）。
```

**Subject 与 Observer 的关系：**
- **一对多关联**：一个 Subject 维护多个 Observer 的列表，`attach` 注册、`detach` 退订、`notify` 广播。
- **依赖倒置**：Subject 只依赖抽象的 `Observer`（接口），不知道具体观察者是谁——这正是 DIP 的体现（可扩展新观察者而不改 Subject）。
- **方向**：Observer 通过 `update()` 被动接收通知，Subject 主动推送。

**C++ 中用智能指针避免生命周期问题：**

```cpp
class Observer {
public:
    virtual ~Observer() = default;
    virtual void update(const std::string& msg) = 0;
};

class Subject {
public:
    void attach(const std::shared_ptr<Observer>& o) { observers_.push_back(o); }
    void notify(const std::string& msg) {
        // 用 weak_ptr 保存，lock() 检查观察者是否还活着
        for (auto it = observers_.begin(); it != observers_.end();) {
            if (auto sp = it->lock()) {          // 观察者还活着 → 安全调用
                sp->update(msg);
                ++it;
            } else {                              // 观察者已析构 → 顺手清理
                it = observers_.erase(it);
            }
        }
    }
private:
    std::vector<std::weak_ptr<Observer>> observers_;   // 弱引用，不延长观察者生命
};

// 使用：
auto obs = std::make_shared<ConcreteObserver>();
subject.attach(obs);
obs.reset();          // 观察者析构；subject 里 weak_ptr 自动失效，notify 不会野指针
```

**要点：**
1. **Subject 持有 `std::weak_ptr<Observer>`** 而非裸指针或 shared_ptr：裸指针会野指针，shared_ptr 会造成 Subject 与 Observer 循环引用无法释放。
2. **notify 时 `lock()`** 提升为临时 shared_ptr，若为空说明观察者已析构，跳过。
3. 若 Observer 也需要访问 Subject，Observer 侧同样用 **weak_ptr** 打破循环引用（否则两边互持 shared_ptr 会导致内存泄漏）。

**通俗解释**：观察者就像"订阅公众号"——公众号（Subject）只存读者的"会员号"（weak_ptr），读者注销了（析构），会员号自动失效，群发时看到失效号就跳过，绝不会给"已注销的人"发消息导致崩溃。

---

## 5. 代码填空：补充 create 函数使其能根据字符串创建 Circle 或 Rectangle，并指出如何通过配置文件进一步解耦

**原题：**
> 代码填空：给出简单工厂的部分代码，补充create函数使其能根据字符串创建Circle或Rectangle，并指出如何通过配置文件进一步解耦。

**答案：**

**补充 create 函数：**

```cpp
#include <memory>
#include <string>

class Shape {
public:
    virtual void draw() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    void draw() const override { /* 画圆 */ }
};

class Rectangle : public Shape {
public:
    void draw() const override { /* 画矩形 */ }
};

class ShapeFactory {
public:
    // 根据字符串创建对应图形；不认识的名字返回 nullptr
    static std::unique_ptr<Shape> create(const std::string& type) {
        if (type == "circle") {
            return std::make_unique<Circle>();
        } else if (type == "rectangle") {
            return std::make_unique<Rectangle>();
        }
        return nullptr;
    }
};
```

**如何通过配置文件进一步解耦：**

简单工厂的 if-else 仍然在代码里写死了"名字→类"的映射，新增图形还是要改 `create`。进一步解耦的做法是：**注册表 + 配置文件**。

1. **定义注册表**：用 `std::map<std::string, 创建函数>` 保存"字符串 → 工厂函数"的映射。
2. **注册**：启动时把每个具体产品的创建函数注册进注册表（或用一个宏/静态注册表）。
3. **读配置文件**：把"当前系统要使用哪些图形"写在 XML/JSON/ini 里，程序读取配置，只从注册表创建配置中列出的类型。

```cpp
class ShapeFactory {
public:
    using Creator = std::function<std::unique_ptr<Shape>()>;
    static bool registerType(const std::string& name, Creator c) {
        creators()[name] = std::move(c);
        return true;
    }
    static std::unique_ptr<Shape> create(const std::string& name) {
        auto it = creators().find(name);
        return it == creators().end() ? nullptr : it->second();
    }
private:
    static std::map<std::string, Creator>& creators() {
        static std::map<std::string, Creator> m;
        return m;
    }
};

// 启动阶段"注册"：
static bool regCircle = ShapeFactory::registerType("circle", [] { return std::make_unique<Circle>(); });
// 之后 create("circle") 不再有 if-else；新增图形 = 新写一个类 + 一行注册，create 不再改动
```

这样**新增图形不再修改工厂内部逻辑**（配合配置文件甚至可以在不改代码的情况下切换使用哪些图形），把"开闭原则"落实得更彻底。

**通俗解释**：if-else 工厂像"电话总机人工转接"，接线表写死在总机里；注册表 + 配置就像"查电话号码簿"，新增一个人只需在电话簿里加一条记录，总机程序本身不用改。

---

## 6. 如何利用 std::weak_ptr 解决观察者生命周期延长与安全退订问题？

**原题：**
> 在观察者模式或多线程回调中，观察者先于被观察者析构会引发野指针崩溃。如何利用std::weak_ptr解决对象生命周期延长与安全退订（weak_ptr.lock()）问题？

**答案：**

**问题本质**：被观察者（Subject）持有观察者（Observer）的裸指针，观察者先析构后，Subject 在 `notify()` 时对已释放的内存调用 `update()` → **野指针 / use-after-free 崩溃**。多线程场景下更隐蔽：通知回调执行到一半，另一个线程把观察者销毁了。

**std::weak_ptr 解决方案：**

`std::weak_ptr` 是 `std::shared_ptr` 的"旁观者"：**它不增加引用计数、不延长对象生命周期**，但能通过 `lock()` 临时"提升"为一个 shared_ptr——**提升成功的瞬间，对象就安全地活着**（引用计数 +1），直到这段提升的 shared_ptr 用完为止，期间谁也别想销毁它。这就是"生命周期延长 + 安全退订"两个问题的统一解法：

```cpp
class Subject {
public:
    void attach(const std::shared_ptr<Observer>& o) { observers_.push_back(o); }
    void notify() {
        for (auto it = observers_.begin(); it != observers_.end();) {
            if (auto sp = it->lock()) {   // ① 提升：观察者活着 → 本次通知期内保证存活
                sp->update();             // ② 安全调用（即使别的线程此刻 reset，也不会崩）
                ++it;
            } else {
                it = observers_.erase(it); // ③ 已析构：lock 返回空，自动退订清理
            }
        }
    }
private:
    std::vector<std::weak_ptr<Observer>> observers_;
};
```

**几个关键细节（多线程下）：**

1. **lock() 是原子的**：`weak_ptr::lock()` 内部原子地检查引用计数并 +1，不会出现"检查到活着、正要调用却被析构"的竞态窗口。
2. **延长的是"调用期间"的生命**：`lock()` 返回的临时 shared_ptr 在该作用域内保持观察者存活，这就是"生命周期延长"。
3. **退订**：观察者析构时，weak_ptr 自动过期（expired），`lock()` 返回空 → 无需显式调用 detach 也能安全跳过，并顺手把失效条目从容器中清除。
4. **显式主动退订**（可选）：Observer 内部保存指向 Subject 的 `std::weak_ptr<Subject>`，析构时 `if (auto s = subjectWeak_.lock()) s->detach(shared_from_this());`，实现"退订"。

**通俗解释**：weak_ptr 像"考勤登记"——Subject 只登记观察者的"编号"（weak_ptr），不发工资（不加引用计数）。群发消息前先按编号**核对人还活着**（lock() 成功）才通知；已经离职（析构）的编号自动失效，直接跳过。而且核对成功的那一刻起，这个人在这条通知期间绝不会被辞退（引用计数 +1），所以永远不会出现"对已离职的人讲话"的崩溃。

---

# 作业3：C++多线程基础

## 1. 线程入口函数传递方式：至少6种可调用对象类型

**原题：**
> 线程入口函数传递方式：列出C++11创建线程时，可以传递的至少6种可调用对象类型（如普通函数、函数指针、lambda等），并各给出一行代码示例。

**答案：**

`std::thread` 的构造函数接收**任何可调用对象（Callable）+ 参数列表**。至少 6 种：

**① 普通函数（函数名即函数指针）**
```cpp
void work() {}
std::thread t1(work);
```

**② lambda 表达式（C++11 最常用）**
```cpp
std::thread t2([]{ std::cout << "hi" << std::endl; });
```

**③ 函数对象（重载 operator() 的类，functor）**
```cpp
struct Worker { void operator()(int n) const { /* ... */ } };
std::thread t3(Worker(), 42);
```

**④ 类成员函数（需绑定对象指针/引用作为第一个参数）**
```cpp
class MyClass { public: void run(int x) {} };
MyClass obj;
std::thread t4(&MyClass::run, &obj, 100);
```

**⑤ 类静态成员函数（无需对象）**
```cpp
class MyClass { public: static void go(int x) {} };
std::thread t5(&MyClass::go, 100);
```

**⑥ std::bind 绑定后的可调用对象**
```cpp
void f(int a, int b) {}
std::thread t6(std::bind(f, 1, std::placeholders::_2), 2);   // _2 用实参 2 填充
```

**⑦ std::function 包装的可调用对象**
```cpp
std::function<void()> fn = []{ /* ... */ };
std::thread t7(fn);
```

**⑧ 可移动/拷贝的任意自定义 Callable（如 lambda 捕获成员、引用包装器 std::ref）**
```cpp
auto l = [&obj]{ obj.run(1); };
std::thread t8(l);
```

> 补充（更现代 C++20）：`std::jthread` 可传 `std::stop_token`，实现协作式取消：
> ```cpp
> std::jthread t9([](std::stop_token st){ while (!st.stop_requested()) { /* ... */ } });
> ```

**通俗解释**：`std::thread` 只要求你给它"一个能被 `()` 调用、并且可移动/可拷贝的东西"——普通函数、lambda、仿函数、成员函数绑定、`std::bind`、`std::function` 都行。把它们想成"一张写好的菜谱纸条"，线程拿过去照着执行。

---

## 2. std::mutex、lock_guard、unique_lock 的区别；为什么 unique_lock 更灵活但效率更低；RAII 如何体现？

**原题：**
> 锁的区别：std::mutex、lock_guard、unique_lock三者有何区别？为什么说unique_lock更灵活但效率更低？RAII思想在此处如何体现？

**答案：**

**三者区别：**

| 类型               | 本质                    | 加锁方式                                                     | 灵活性                        |
| ------------------ | ----------------------- | ------------------------------------------------------------ | ----------------------------- |
| `std::mutex`       | **裸互斥锁**            | 必须手动 `lock()` / `unlock()`                               | 最低，忘 unlock 就死锁        |
| `std::lock_guard`  | RAII 包装器             | 构造时 `lock()`，析构自动 `unlock()`                         | 中，作用域内无法手动解锁/重锁 |
| `std::unique_lock` | RAII 包装器（功能最强） | 可延迟加锁(`defer_lock`)、`try_lock`、手动 `lock/unlock`、可移动、与条件变量配合 | 最高                          |

**`unique_lock` 更灵活的具体点：**
- 支持 `std::defer_lock`（先构造不锁，稍后手动锁）
- 支持 `try_lock()` / `try_lock_for()` / `try_lock_until()`
- 可以中途 `unlock()` 再 `lock()`（配合 `std::condition_variable::wait` 是**必需**的——wait 需要释放锁）
- 支持移动语义（可把锁的"所有权"转移给别的 unique_lock）

**为什么 unique_lock 效率更低：**
`unique_lock` 内部需要维护**"是否拥有锁"的状态标志**，每次 `lock/unlock` 都要多一次状态检查和可能的标志位读写；而 `lock_guard` 是"零额外状态"的栈对象（构造加锁、析构解锁，无成员标志）。在频繁加解锁的高并发热点路径上，unique_lock 的开销略高。

**RAII 思想在此处的体现：**
RAII = **Resource Acquisition Is Initialization**（资源获取即初始化）。`lock_guard`/`unique_lock` 把"互斥锁"这个资源绑定到对象的**生命周期**：
- **构造** → 获取锁（获取资源）
- **析构** → 自动释放锁（释放资源，无论正常 return 还是抛异常）

这样**永远不需要手写 `unlock()`**，即使函数中途抛异常、提前 return，析构函数也会兜底解锁，从根源上避免"忘记解锁导致死锁"。

**通俗解释**：`mutex` 是"一把裸锁"，你得记着开锁关锁，忘了就锁死；`lock_guard` 是"房卡"，进房自动锁门、出房（作用域结束）自动开门；`unique_lock` 是"智能房卡"，不但自动开门，还能让你中途出去再进来、甚至把卡转给别人——功能多，卡片本身也"重"一点（多了状态记录），所以效率略低。

---

## 3. 死锁分析：以下代码为什么可能死锁？如何改进？

**原题：**
> 死锁分析：以下代码为什么可能死锁？如何用lock_guard或unique_lock改进？
> ```cpp
> mutex m1, m2;
> void f1() { m1.lock(); m2.lock(); /*...*/ m1.unlock(); m2.unlock(); }
> void f2() { m2.lock(); m1.lock(); /*...*/ m2.unlock(); m1.unlock(); }
> ```

**答案：**

**为什么会死锁：加锁顺序不一致（交叉加锁）**

- `f1` 先锁 `m1` 再锁 `m2`
- `f2` 先锁 `m2` 再锁 `m1`

当两个线程并发执行时，可能发生：线程 A 拿到 `m1`，线程 B 拿到 `m2`，然后 A 想拿 `m2`（被 B 占着）、B 想拿 `m1`（被 A 占着），**双方都在等对方释放自己需要的锁，谁都不让 → 永久阻塞 → 死锁**。另外注意：如果 `/*...*/` 中间抛异常，`unlock()` 根本执行不到，裸 `mutex` 也会锁死（这属于异常安全导致的死锁）。

**改进方案：**

**方案1：统一加锁顺序（最简单、最推荐）**
```cpp
// 所有函数都"先 m1 后 m2"，锁顺序一致就不可能交叉等待
void f1() { std::lock_guard<std::mutex> lk1(m1); std::lock_guard<std::mutex> lk2(m2); /* ... */ }
void f2() { std::lock_guard<std::mutex> lk1(m1); std::lock_guard<std::mutex> lk2(m2); /* ... */ }
```

**方案2：用 `std::lock()` 一次性同时锁多个锁（C++11）**
```cpp
void f1() {
    std::unique_lock<std::mutex> lk1(m1, std::defer_lock);
    std::unique_lock<std::mutex> lk2(m2, std::defer_lock);
    std::lock(lk1, lk2);   // 内部保证以不产生死锁的顺序锁定
    /* ... */
}
```

**方案3（C++17 更现代）用 `std::scoped_lock` 一行搞定**
```cpp
void f1() { std::scoped_lock lock(m1, m2); /* ... */ }   // 变参，内部用 std::lock 策略，自动解锁
void f2() { std::scoped_lock lock(m1, m2); /* ... */ }
```

**方案4：try_lock + 回退（超时机制）**
```cpp
void f2() {
    std::unique_lock<std::mutex> lk2(m2, std::defer_lock);
    while (!lk2.try_lock()) { /* 拿不到就稍等重试，绝不无限等待 */ }
    std::unique_lock<std::mutex> lk1(m1);
    /* ... */
}
```

**通俗解释**：死锁就是"**你等我、我等你**"。两个人都要先借到"1号钥匙"再借"2号钥匙"，结果一人拿了一把，都等对方先还另一把——卡死。解决办法就三条路：**排队顺序统一**（都先拿1号）、**一次同时申请多把**（std::lock/scoped_lock 内部避免交叉）、**申请不到就超时重试**（try_lock）。

---

## 4. atomic 为什么能不加锁实现线程安全？简述 CAS 机制的核心步骤

**原题：**
> 原子变量原理：atomic为什么能不加锁实现线程安全？简述CAS机制的核心步骤。

**答案：**

**为什么 atomic 能不加锁实现线程安全：**

核心原因是**硬件提供了原子指令**。普通的 `i++` 不是原子的（读-改-写三步，可能被中断/其他线程插队），而 `std::atomic` 会编译成带 **LOCK 前缀**的 CPU 原子指令（如 x86 的 `LOCK INC`、`LOCK XADD`、`LOCK CMPXCHG`），或者依赖 CPU 的 cache 一致性协议（MESI）与总线锁，**保证"读-改-写"这一整段在硬件层面不可分割**，因此不需要操作系统互斥锁。现代 `std::atomic` 在 `is_always_lock_free()` 为 true 的平台上就是**无锁**的（不借助 mutex）。

**CAS（Compare-And-Swap，比较并交换）的核心步骤：**

```
循环：
  ① 读：  把内存中当前值读出来，记为 old
  ② 比较：判断 old 是否等于"期望值 expect"
  ③ 交换：若相等  → 把"新值 new"原子地写入内存，返回成功(true)
          若不相等 → 什么都不写，返回失败(false)，把当前值带出来
  ④ 若失败，用最新的当前值更新 expect，回到①重试（直到成功）
```

对应 x86 指令就是 `LOCK CMPXCHG`：它在一个不可分割的指令里完成"比较 + 有条件地交换"。典型用法（CAS 自旋）：

```cpp
std::atomic<int> counter{0};
int expected = counter.load();
while (!counter.compare_exchange_weak(expected, expected + 1)) {
    // 失败说明被别人改了，expected 已被更新为最新值，重试
}
```

**补充**：CAS 存在 **ABA 问题**（值从 A 变成 B 又变回 A，CAS 误以为没变过），通常用带版本号的 CAS（如 `std::atomic<std::pair<ptr,version>>`，或 SSO 用 `std::atomic<std::shared_ptr>`）解决。

**通俗解释**：普通变量像"白板"，别人可以随时改一半；`atomic` 像"银行柜台的单据"，整个"核对-修改-签收"过程在一个窗口一次办完，别的人只能等你办完才能办。CAS 就是"**先看一眼现在是多少，若还是我记的那个数，我就把它改成新值；如果不是，说明被抢改了，我就重新看再试**"——全程不锁门，靠"失败重试"保证最终正确。

---

## 5. 写出 TaskQueue 的 push 和 pop 关键代码，说明为什么用 while 而非 if；解释 wait 的上半部与下半部

**原题：**
> 生产者消费者实现：写出TaskQueue的push和pop函数关键代码，说明为什么使用while而不是if来检查条件变量？解释wait的上半部（解锁+睡眠）和下半部（唤醒+加锁）过程。

**答案：**

**TaskQueue 的 push / pop 关键代码：**

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>

template <typename T>
class TaskQueue {
public:
    explicit TaskQueue(size_t cap) : capacity_(cap) {}

    void push(const T& task) {
        std::unique_lock<std::mutex> lock(mutex_);
        notFull_.wait(lock, [this] { return queue_.size() < capacity_; }); // 满则等
        queue_.push(task);
        notEmpty_.notify_one();          // 唤醒一个消费者
    }

    T pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        notEmpty_.wait(lock, [this] { return !queue_.empty(); });          // 空则等
        T task = std::move(queue_.front());
        queue_.pop();
        notFull_.notify_one();           // 唤醒一个生产者
        return task;
    }

private:
    std::queue<T> queue_;
    size_t capacity_;
    std::mutex mutex_;
    std::condition_variable notFull_;
    std::condition_variable notEmpty_;
};
```

**为什么用 `while` 而不是 `if` 检查条件：**

1. **假唤醒（spurious wakeup）**：标准规定条件变量可能在没有 `notify` 的情况下自行醒来（底层信号/中断导致）。如果用 `if`，醒了一次就直接往下走，但此时条件（队列非空/未满）**未必成立**，就会拿错数据或越过容量。
2. **多个等待者被同时唤醒**：`notify` 可能唤醒多个线程，但只有一个抢到资源，其余醒来后发现条件又被别人改回不满足——必须再检查。

所以正确写法是**循环检查**：`while (!条件) wait(...)`。`wait(lock, predicate)` 这个带谓词的重载**内部就是 while 循环**，所以用它最安全。

**wait 的上半部（解锁 + 睡眠）与下半部（唤醒 + 加锁）过程：**

```
上半部（进入 wait）：
  ① 原子地：把锁释放（unlock）+ 让线程进入睡眠/挂起。
     关键点：这两步必须"原子"完成——如果先解锁、后睡眠，
     别的线程可能在这个间隙把条件改好并 notify，而本线程还没睡，
     就永远错过通知（丢失唤醒 lost wakeup）。
  ② 本线程进入条件变量的等待队列，等待被 notify。

下半部（被 notify 唤醒）：
  ① 线程被唤醒，从等待队列移出。
  ② 尝试重新获取（加锁）之前释放的 mutex（可能还要排队等别人先拿）。
  ③ 拿到锁后 wait 才返回，继续执行 while 条件判断。
     注意：唤醒 ≠ 立即加锁成功，加锁本身也可能阻塞。
```

**通俗解释**：`wait` 就是"**我先放下手里的锁、趴桌上睡，等有人叫我**"——放下锁和趴下是"一个动作"（上半部，防止漏叫）；被叫醒后"**先重新把锁抓回手里才睁眼**"（下半部）。至于为什么用 while：服务员（notify）喊"有座了！"，结果你醒来看见座位又被别人抢了——所以要醒来再看一眼，没座就接着睡（循环）。

---

## 6. 完整代码题：实现一个简单的生产者-消费者模型，使用 unique_lock 和条件变量，队列容量为5

**原题：**
> 完整代码题：实现一个简单的生产者‑消费者模型，使用unique_lock和条件变量，生产者和消费者各为一个线程，队列容量为5。

**答案：**

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <chrono>

class TaskQueue {
public:
    explicit TaskQueue(size_t capacity) : capacity_(capacity) {}

    void push(int value) {
        std::unique_lock<std::mutex> lock(mutex_);
        notFull_.wait(lock, [this] { return queue_.size() < capacity_; }); // 满则等待
        queue_.push(value);
        notEmpty_.notify_one();          // 唤醒一个消费者
    }

    int pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        notEmpty_.wait(lock, [this] { return !queue_.empty(); });          // 空则等待
        int value = queue_.front();
        queue_.pop();
        notFull_.notify_one();           // 唤醒一个生产者
        return value;
    }

private:
    std::queue<int> queue_;
    size_t capacity_;
    std::mutex mutex_;
    std::condition_variable notFull_;
    std::condition_variable notEmpty_;
};

int main() {
    TaskQueue queue(5);   // 队列容量为 5

    std::thread producer([&queue] {
        for (int i = 0; i < 20; ++i) {
            queue.push(i);
            std::cout << "生产: " << i << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
        }
    });

    std::thread consumer([&queue] {
        for (int i = 0; i < 20; ++i) {
            int value = queue.pop();
            std::cout << "消费: " << value << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(120));
        }
    });

    producer.join();
    consumer.join();
    return 0;
}
```

**要点回顾：**
- `push` 用 `notFull_` 条件变量（容量满等待），`pop` 用 `notEmpty_` 条件变量（队列空等待）；
- 都使用 `std::unique_lock`（因为 `condition_variable::wait` 需要它能解锁/重锁的能力，`lock_guard` 做不到）；
- `wait` 带谓词 → 自动 `while` 循环检查，杜绝假唤醒；
- 生产/消费各一个线程，`join()` 保证两个线程跑完再退出。

> 更现代（C++20）写法：用 `std::jthread`（析构自动 join）替代 `std::thread` + 手写 `join()`，更不易泄漏。

**通俗解释**：这就是"餐厅后厨"：队列是出餐台（最多摆 5 盘），厨师（生产者）做满 5 盘就等客人吃；服务员（消费者）把台子拿空就等厨师上菜。台满时厨师等（notFull_），台空时服务员等（notEmpty_），双方靠条件变量打暗号互相叫醒，互不空转、互不越界。

---

# 作业4：线程池

## 1. 面向对象线程池的 start() 函数做了什么？为什么会出现"任务执行不完成程序就退出"？解决办法是什么？

**原题：**
> 问题分析：面向对象线程池的启动start()函数做了什么？为什么会出现"任务执行不完成程序就退出"的问题？解决办法是什么？

**答案：**

**start() 做了什么：**

`start()` 负责"拉起工作线程"。典型实现：

```cpp
void ThreadPool::start() {
    _isExit = false;                       // 复位退出标志
    for (size_t i = 0; i < _threadNum; ++i) {
        _threads.emplace_back([this] {     // 创建 N 个后台工作线程
            while (!_isExit) {             // 每个线程循环：取任务 → 执行
                Task task = _queue.pop();
                if (task) task();
            }
        });
    }
}
```

它做三件事：① 复位退出标志；② 创建并启动 N 个工作线程；③ 每个工作线程进入"**取任务 → 执行 → 再取任务**"的循环，任务队列空时阻塞在条件变量上等待。

**为什么会出现"任务执行不完成程序就退出"：**

原因：`std::thread` 创建的线程是**后台线程**。主线程（main）执行完自己的代码后直接 `return 0`，此时 `main` 退出 → 进程终止，**操作系统会直接结束所有线程**（包括那些还在执行任务的工作线程），任务自然来不及跑完。这是线程池使用最常见的错误：**没等线程池把任务消费完就退出进程**。

**解决办法：**

在 `main` 结束前**优雅停机**：
1. 调用 `stop()`：置 `_isExit = true`；
2. **唤醒**所有睡眠中的工作线程（`notify_all`，否则它们永远阻塞在条件变量上）；
3. 对每个工作线程调用 `join()`，等待它们把手头任务执行完、退出循环；
4. 全部 join 完再返回 `main` → 任务必然执行完毕。

```cpp
ThreadPool pool;
pool.start();
pool.addTask(...);          // 投递任务
pool.stop();                // 优雅停机：等所有任务执行完再退出
```

**通俗解释**：工作线程是"临时工"，`main` 是"老板"。老板如果说完话就走人（return），整个公司（进程）直接关门，临时工手里的活（任务）全被打断。正确做法是老板走之前先宣布"今天下班"（stop：置标志 + 喊醒所有人），再等每个人把手头活干完、签退（join），才关灯锁门。

---

## 2. 为什么 stop() 仅设置 _isExit = true 可能导致子线程永远睡眠？wakeup() 如何解决？为什么 pop 中要增加 _flag 判断？

**原题：**
> 线程池退出难题：为什么在线程池的stop()中仅设置_isExit = true可能导致子线程永远睡眠？wakeup()函数如何解决？为什么在pop中要增加_flag判断？

**答案：**

**为什么仅设置 `_isExit = true` 会导致子线程永远睡眠：**

工作线程的循环是"`pop()` 取任务 → 执行"。当任务队列**为空**时，`pop()` 会调用 `notEmpty_.wait(...)` 阻塞在条件变量上**等待被 notify**。此时主线程在 `stop()` 里只改了一个 `_isExit = true`——**没有唤醒等待的线程**，那些 worker 就继续在 `wait()` 上睡死过去：
- 若 `stop()` 随后 `join()`：会**永远阻塞**（线程不醒，join 等不到它结束）；
- 若不 join 直接退出：进程粗暴结束线程，行为未定义。

**`wakeup()` 如何解决：**

`wakeup()` 的本质是对**条件变量 `notify_all()`**，把当前所有阻塞在 `wait()` 上的工作线程全部唤醒。线程醒来后会**重新检查退出标志 `_isExit`**，发现为 true 就退出 `while` 循环、结束线程，`join()` 便能顺利返回：

```cpp
void TaskQueue::wakeup() {
    notEmpty_.notify_all();   // 唤醒所有沉睡的 worker
}

void ThreadPool::stop() {
    _isExit = true;           // 置退出标志
    _queue.wakeup();          // 必须唤醒，否则 worker 睡死
    for (auto& th : _threads) th.join();
}
```

**为什么 `pop()` 中要增加 `_flag`（退出标志）判断：**

单纯被 `notify_all` 唤醒还不够——`wait` 醒来后默认行为是"**继续取任务**"。如果不加退出标志判断，worker 醒来后看到队列空（或又有新任务）可能继续 `wait` 或继续干活，根本不会退出。所以在 `pop()` 里要同时检查"队列是否有任务"和"是否已要求退出"：

```cpp
Task TaskQueue::pop(bool& exitFlag) {
    std::unique_lock<std::mutex> lock(mutex_);
    // 队列空 且 未要求退出 → 才继续等；退出标志置位时不再等待
    while (queue_.empty() && !exitFlag) {
        notEmpty_.wait(lock);
    }
    if (queue_.empty()) {        // 已要求退出且队列也空了 → 返回空任务让线程结束
        return Task();           // 空 Task（可调用对象为 null）
    }
    Task t = std::move(queue_.front());
    queue_.pop();
    return t;
}
```

这样语义清晰：**"队列有任务"永远优先执行完（停机会先把积压任务跑完）；"没任务且已要求退出"就返回空任务结束线程**。这就是 `_flag` 判断的意义——让"被唤醒"真正转化为"退出"，而不是空转或死等。

**通俗解释**：`_isExit = true` 就像老板口头说"下班了"但**没拉响下班铃**——睡着的员工听不见，还在打盹（wait），怎么等都等不到人走（join 卡死）。`wakeup()`（notify_all）就是**拉响下班铃**，把人叫醒；而 `pop` 里的 `_flag` 判断就是让醒来的员工确认"哦，真下班了，那我收拾走人"，而不是醒了又去抢活干。

---

## 3. 基于对象线程池：任务添加方式有什么不同？解释 addTask(bind(&MyTask::process, ptask.get(), 100)) 如何传递业务逻辑

**原题：**
> 基于对象线程池：与面向对象线程池相比，基于对象的线程池在任务添加方式上有什么不同？解释addTask(bind(&MyTask::process, ptask.get(), 100))是如何将业务逻辑传递给工作线程的。

**答案：**

**两种线程池任务添加方式的不同：**

- **面向对象线程池**：定义一个抽象 `Task` **基类**（含纯虚 `process()`），业务类继承 Task、实现 `process()`。addTask 接收 `Task*`，工作线程取出后调用 `task->process()`。缺点是：**每个业务都要继承 Task，业务与线程池基类强耦合**，且一个任务必须是一个"类"。

- **基于对象（面向回调）线程池**：任务不再要求继承基类，而是把任务封装成**可调用对象**（`std::function<void()>`）。`addTask` 接收任意可调用对象，工作线程取出后直接 `task()` 调用。业务逻辑以**闭包/绑定函数**的形式"注入"，**与线程池完全解耦**（只要是个能被 `()` 调用的东西就能当任务）。

```cpp
// 面向对象版
class MyTask : public Task { void process() override {...} };
pool.addTask(new MyTask());

// 基于对象版（addTask 接收 std::function<void()>）
pool.addTask(std::bind(&MyTask::process, ptask.get(), 100));
pool.addTask([]{ doSomething(); });      // lambda 直接投递，无需任何基类
```

**解释 `addTask(std::bind(&MyTask::process, ptask.get(), 100))`：**

这条语句把"**在 ptask 指向的对象上调用其成员函数 process，参数为 100**"这件事整体封装成了一个**不带参数的 `std::function<void()>`**：

1. `&MyTask::process` —— 取 `MyTask` 的成员函数 `process` 的指针（签名是 `void MyTask::process(int)`）；
2. `ptask.get()` —— 传**对象指针**作为成员函数的隐式 `this` 参数（通常 `ptask` 是 `std::unique_ptr`/`shared_ptr`）；
3. `100` —— 绑定 `process` 的实参；
4. `std::bind` —— 把"对象 + 成员函数 + 参数"打包成一个可调用对象，放进 `addTask`；
5. 线程池内部把它保存为 `std::function<void()>`，放入任务队列；
6. 某个工作线程 `pop()` 取出后执行 `task()`，等价于调用 `ptask->process(100)`——**业务逻辑就这样从主线程传递到工作线程执行**。

**通俗解释**：面向对象版是"所有任务都得按公司模板填表格（继承 Task）"；基于对象版是"任何纸条都能当任务（std::function），只要上面写了怎么干活"。`std::bind` 就像**把"张三、他的身份证号、给 100 元"写在一张授权书上**，交给任何一个员工（worker）照办——员工不关心张三是谁，只要按授权书执行就行。

> 补充：C++11 更常用 lambda 替代 bind：`pool.addTask([p = ptask.get()]{ p->process(100); });`，可读性更好。C++20 里还能用 `std::move_only_function`。

---

## 4. 设计题：让线程池支持任务返回值（类似 std::future），在哪个环节添加？简述思路

**原题：**
> 设计题：如果要让线程池支持任务返回值（类似std::future），你会在哪个环节添加？简述思路。

**答案：**

**添加环节：`addTask` 接口层（任务封装环节）。**

具体思路：让 `addTask` 返回一个 `std::future<T>`。利用 C++11 的 **`std::packaged_task` + `std::promise`** 机制——把用户传入的可调用对象包进 `packaged_task`，它内部自带一个 `promise`，执行结束后自动把返回值写入 promise，调用方通过 `future` 拿结果：

```cpp
class ThreadPool {
public:
    // 核心：接收任意可调用对象，返回 future，任务完成时 future 里就有返回值
    template <typename F, typename... Args>
    auto addTask(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>> {   // C++17；C++11 用 decltype

        using Ret = std::invoke_result_t<F, Args...>;

        // 1. 把用户任务包进 packaged_task（自带 promise，能产出 future）
        std::packaged_task<Ret()> task(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));

        // 2. 取出 future 返回给调用方
        std::future<Ret> res = task.get_future();

        // 3. packaged_task 不可拷贝，必须 move 进队列（包一层 std::function 要求可拷贝）
        {
            std::lock_guard<std::mutex> lk(mutex_);
            queue_.emplace(std::move(task));   // queue_ 类型: std::queue<std::packaged_task<Ret()>>
        }
        cond_.notify_one();
        return res;
    }
};
```

**关键点：**
1. **`std::packaged_task`**：把"可调用对象 + promise"绑在一起，执行完自动 `set_value`（或捕获异常 `set_exception`）；
2. **`std::future`**：调用方用来**阻塞等待结果**（`future.get()`）或轮询（`wait_for`）；
3. **packaged_task 不可拷贝**：入队要用 `emplace` + `move`（这是工程上最常踩的坑）；
4. **工作线程**：`pop()` 出 packaged_task 后 `task()` 执行即可，无需知道返回值——返回值自动流入 promise/future；
5. **异常处理**：任务抛异常时 promise 自动携带异常，`future.get()` 会重新抛出，调用方能捕获。

> 补充（C++20）：`std::packaged_task` 在 C++20 中被标记为 deprecated，推荐自封装或用 `std::promise` + `std::future` 直接组合；但绝大多数课程/面试仍以 `packaged_task` 作答。

**通俗解释**：`packaged_task` 像"**带回执的快递单**"——你把任务（包裹）交给线程池（快递员），同时拿到一张回执（future）；快递员送完（任务执行完）会在回执上写明结果（promise 存值）；你随时可以拿着回执查结果（future.get()），结果没到就先等着（阻塞）。所以"支持返回值"要做的就是把**普通任务升级成"带回执的任务"**，这个升级就发生在 addTask 的封装环节。

---

# 作业5：计算机网络基础

## 1. OSI 七层、TCP/IP 四层如何对应？每层举一个常见协议

**原题：**
> 分层模型：OSI七层模型是哪七层？TCP/IP四层模型如何对应？每层举一个常见协议。

**答案：**

**OSI 七层模型（自下而上）：**

1. **物理层**（Physical）
2. **数据链路层**（Data Link）
3. **网络层**（Network）
4. **传输层**（Transport）
5. **会话层**（Session）
6. **表示层**（Presentation）
7. **应用层**（Application）

（记忆口诀："物、数、网、传、会、表、应"）

**与 TCP/IP 四层模型的对应关系：**

| OSI 七层                 | TCP/IP 四层    | 常见协议举例                                      |
| ------------------------ | -------------- | ------------------------------------------------- |
| 应用层 + 表示层 + 会话层 | **应用层**     | HTTP / HTTPS、DNS、FTP、SMTP、SSH、Telnet         |
| 传输层                   | **传输层**     | TCP、UDP                                          |
| 网络层                   | **网际层**     | IP、ICMP、IGMP、IPsec                             |
| 数据链路层 + 物理层      | **网络接口层** | Ethernet（以太网）、ARP/RARP、Wi-Fi (802.11)、PPP |

（教学上常把 TCP/IP 拆成"五层模型"：物理层、数据链路层、网络层、传输层、应用层，只是把 OSI 的会话/表示合进应用层、数据链路与物理分开而已。）

**通俗解释**：OSI 七层是"理论教科书"（分得细但没人完全照做），TCP/IP 四层是"实际工程"（干活的真家伙）。理解核心就抓住三层：**IP 负责"送到家"（网际层）、TCP/UDP 负责"送到人手上并保证质量"（传输层）、HTTP 等负责"人说什么话"（应用层）**。

---

## 2. TCP 特性的实际通信含义

**原题：**
> TCP特性：TCP是面向连接、可靠、全双工、字节流、流量控制的协议。分别解释这些特性在实际通信中的含义。

**答案：**

1. **面向连接**：通信前必须先**三次握手**建立连接（双方确认彼此可达、分配资源），结束后**四次挥手**释放连接。实际含义：TCP 不像 UDP 那样"发了就不管"，而是"先打通电话再说话，说完挂电话"。

2. **可靠**：通过**序号 + 确认应答（ACK）+ 超时重传 + 校验和 + 去重排序**保证数据**不丢、不错、不乱**。实际含义：发方发出数据后等 ACK，超时未收到就重传；接收方按序号拼装，即使网络丢包乱序，最终交给应用层的字节流也是完整有序的。

3. **全双工**：一条连接上双方**可以同时收发**。实际含义：收发各用一组序号/窗口，互不干扰；所以关闭连接要"两个方向各自关"（这也是四次挥手的原因）。

4. **字节流**：TCP 只保证"字节的先后顺序"，**不保留消息边界**。实际含义：应用层 write 一次的数据，对端可能 read 到半截（拆包）或多段拼一起（粘包）——**必须由应用层自己定义消息边界**（定长/包头+长度/分隔符）。这是网络编程最容易踩的坑。

5. **流量控制**：接收方通过**通告窗口**告诉发送方"我还能收多少"，发送方据此调整发送速率，防止**把接收方缓冲区撑爆导致丢包**。实际含义：就像水管前有个"漏斗"，接收方说"你慢点倒，我这边装不下了"，发送方就限流。

> 补充：TCP 还有**拥塞控制**（慢启动、拥塞避免、快重传、快恢复），解决的是"网络本身太堵"的问题，与流量控制（接收方太慢）是两个概念。

**通俗解释**：TCP 像"**可靠的快递**"——寄之前先确认地址（握手）、每件包裹编号（序号）、签收要回执（ACK）、丢了重新寄（重传）、到了按编号排好序、收件人装不下就让你少寄点（流量控制），并且支持"你寄我收、我寄你收"同时进行（全双工）。但它不保证"一次寄的就是一整件"——快递会把大包裹分箱，也会把多个小包裹装同一车（粘包/拆包），拆箱装车的规则要你自己定。

---

## 3. 三次握手、四次挥手时序图；为什么四次；半关闭；2MSL 的作用

**原题：**
> 三次握手与四次挥手：画出三次握手和四次挥手的时序图，标注SYN、ACK、FIN等标志位。为什么关闭连接需要四次？什么是半关闭状态？2MSL的作用是什么？

**答案：**

**三次握手（建立连接）：**

```
客户端 Client                         服务器 Server
   │                                      │
   │   SYN=1, seq=x  ────────────────────▶│  ① 客户端发起连接（同步请求）
   │                                      │
   │   SYN=1, ACK=1, seq=y, ack=x+1 ◀─────│  ② 服务器同意 + 自己也请求连接
   │                                      │
   │   ACK=1, seq=x+1, ack=y+1 ──────────▶│  ③ 客户端确认，连接建立
   │                                      │
   └────────── ESTABLISHED ────────────────┘
```
- ① 客户端发 `SYN=1, seq=x`（我申请连接）；
- ② 服务器回 `SYN=1, ACK=1, seq=y, ack=x+1`（同意你 + 我也要连你）；
- ③ 客户端回 `ACK=1, seq=x+1, ack=y+1`（确认收到）。
- 前两次 SYN 其实是"确认双方**收发能力**都正常"：①证明客户端能发、②证明服务器能发能收、③证明客户端能收。

**四次挥手（关闭连接）：**

```
客户端（主动关闭方）                   服务器（被动关闭方）
   │                                      │
   │   FIN=1, seq=u  ────────────────────▶│  ① 我这边不再发送数据了（半关闭开始）
   │                                      │
   │   ACK=1, ack=u+1  ◀───────────────── │  ② 收到你的 FIN，但我的数据还没发完
   │          （半关闭：客户端→服务器方向关闭）│
   │   （服务器继续发送剩余数据...）          │
   │                                      │
   │   FIN=1, seq=v  ◀─────────────────── │  ③ 我这边数据也发完了，我也要关
   │                                      │
   │   ACK=1, ack=v+1  ──────────────────▶│  ④ 确认，进入 TIME_WAIT（等 2MSL）
   │                                      │
```

**为什么关闭需要四次：**
TCP 是全双工的，**两个方向的关闭相互独立**。主动方发 `FIN` 只代表"我这边**发**完了"，但被动方可能还有数据没发完，所以被动方先回一个 `ACK`（确认收到你的 FIN），等**自己**的数据发完后，再单独发一个 `FIN`（我这边也关）。于是"确认"和"我的 FIN"被拆成两条消息 → 比建立连接（SYN+ACK 合并）多了一次。

**半关闭（half-close）状态：**
一个方向已经关闭、另一个方向仍在传输的状态。即：主动方发了 FIN 并收到 ACK 后，**它不再发送，但仍可以接收**服务器剩余的数据。可用 `shutdown(SHUT_WR)` 显式实现（只关发送方向，保留接收）。

**2MSL 的作用（TIME_WAIT 状态）：**
主动关闭方在发完最后一个 ACK 后进入 `TIME_WAIT`，持续 **2 × MSL**（MSL = 报文最大生存时间，通常 30s–2min）：
1. **确保最后的 ACK 能到达对方**：若该 ACK 丢失，对方会重发 FIN，TIME_WAIT 期间本端还能再回 ACK；若立即关闭，对方重发的 FIN 将无人应答。
2. **让本连接产生的所有旧报文在网络中自然消失**：防止本连接"迟到"的旧包混入后续使用同一四元组（IP+端口）的新连接，造成数据错乱。

**通俗解释**：握手是"你好—你好—收到"，三次就能互证双方都能说能听；挥手是"我要挂了—好，等我话说完—我说完了—收到"，**因为两个人要分别确认对方把话说完了**，所以多一次。2MSL 是"挂电话后等两分钟"，一是怕最后一句"拜拜"没送到要补一句，二是让线上"回声"散干净，别串到下一通电话里。

---

## 4. TCP 的 11 种状态、从 LISTEN 到 ESTABLISHED 的状态路径、主动/被动关闭路径

**原题：**
> 状态迁移：列出TCP的11种状态，说明从LISTEN到ESTABLISHED经过哪些状态？主动关闭和被动关闭的状态路径分别是什么？

**答案：**

**TCP 的 11 种状态：**

```
CLOSED      （初始：无连接）
LISTEN      （服务器监听，等待连接）
SYN_SENT    （客户端已发 SYN，等待确认）
SYN_RCVD    （服务器收到 SYN，已回 SYN+ACK）
ESTABLISHED （连接已建立，正常通信）
FIN_WAIT_1  （主动关闭方已发 FIN）
FIN_WAIT_2  （主动关闭方已收到对端 ACK，等待对端 FIN）
CLOSE_WAIT  （被动关闭方收到 FIN，已回 ACK，等待自己关闭）
CLOSING     （双方同时关闭，两边的 FIN 交叉）
LAST_ACK    （被动关闭方已发 FIN，等待最后 ACK）
TIME_WAIT   （主动关闭方发完最后 ACK，等 2MSL 后关闭）
```

**从 LISTEN 到 ESTABLISHED 经过哪些状态：**

- **服务器（被动方）**：`LISTEN → SYN_RCVD → ESTABLISHED`
  - 收到客户端 SYN → 进入 `SYN_RCVD`，回 SYN+ACK；
  - 收到客户端 ACK → 进入 `ESTABLISHED`。
- **客户端（主动方）**：`CLOSED → SYN_SENT → ESTABLISHED`
  - 发 SYN → `SYN_SENT`；
  - 收到服务器 SYN+ACK、回 ACK → `ESTABLISHED`。

**主动关闭和被动关闭的状态路径：**

```
主动关闭方：
  ESTABLISHED → FIN_WAIT_1（发FIN）→ FIN_WAIT_2（收到ACK）
      →（收到对端FIN）→ TIME_WAIT（回最后ACK，等2MSL）→ CLOSED
    * 若双方同时发 FIN：FIN_WAIT_1 → CLOSING → TIME_WAIT → CLOSED

被动关闭方：
  ESTABLISHED → CLOSE_WAIT（收到FIN、回ACK）→ LAST_ACK（发FIN）
      →（收到最后ACK）→ CLOSED
```

**通俗解释**：状态机就是 TCP 的"**台账**"，每一句话（SYN/ACK/FIN）都对应一次记账。关连接时主动方要"过账"四次（FIN_WAIT_1→FIN_WAIT_2→TIME_WAIT→CLOSED），被动方只要"挂账"两次（CLOSE_WAIT→LAST_ACK→CLOSED）。`netstat`/`ss` 看到的正是这些状态，排障时看 CLOSE_WAIT 堆积（对方没关干净）和 TIME_WAIT 堆积（主动方等 2MSL）最常见。

---

## 5. 大端/小端、网络字节序、htonl/htons，代码填充 sockaddr_in

**原题：**
> 字节序：大端和小端的区别是什么？网络字节序采用哪种？给出htonl、htons等函数的作用，并写一段代码将本机IP 192.168.1.100和端口8080填充到sockaddr_in中（注意转换）。

**答案：**

**大端 vs 小端：**
- **大端（Big-Endian）**：**高字节存低地址**（高位在前，像人写字一样从左到右）。
- **小端（Little-Endian）**：**低字节存低地址**（低位在前）。
- 例：32 位整数 `0x12345678`，假设地址从 `0x100` 开始：
  - 大端：`0x100=12 0x101=34 0x102=56 0x103=78`
  - 小端：`0x100=78 0x101=56 0x102=34 0x103=12`

**网络字节序采用哪种：** **大端（Big-Endian）**。TCP/IP 协议规定所有协议头中的多字节数值（端口、IP 等）一律用大端在网络上传输，保证异构主机之间能正确解析。

**htonl / htons 等函数的作用：**

| 函数    | 全称                  | 作用                                       |
| ------- | --------------------- | ------------------------------------------ |
| `htonl` | Host TO Network Long  | 主机字节序 → 网络字节序（32 位，用于 IP）  |
| `htons` | Host TO Network Short | 主机字节序 → 网络字节序（16 位，用于端口） |
| `ntohl` | Network TO Host Long  | 网络字节序 → 主机字节序（32 位）           |
| `ntohs` | Network TO Host Short | 网络字节序 → 主机字节序（16 位）           |

在小端机器上这些函数会**交换字节**；在大端机器上是**空操作**。所以正确写法永远要调用它们，才能做到跨平台。

**代码：将 IP 192.168.1.100 和端口 8080 填充到 sockaddr_in：**

```cpp
#include <cstring>        // memset
#include <arpa/inet.h>    // htons / inet_pton / sockaddr_in

struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));          // 清零（推荐，避免残留脏数据）

addr.sin_family = AF_INET;               // IPv4 地址族
addr.sin_port   = htons(8080);           // 端口：主机序 → 网络序（大端）

// IP：推荐 inet_pton（IPv4/IPv6 通用，且能校验格式，优于 inet_addr）
inet_pton(AF_INET, "192.168.1.100", &addr.sin_addr);
// 等价旧写法：
// addr.sin_addr.s_addr = inet_addr("192.168.1.100");  // inet_addr 已返回网络序
```

> 注意：`inet_pton` 写入 `sin_addr` 的已经是**网络字节序**；端口必须显式 `htons()`。忘了 `htons` 是在小端机器上最常见的 bug（表现为连不上的端口号错乱）。

**通俗解释**：字节序就是"**数字在内存里正着存还是倒着存**"。网络规定统一"正着存"（大端），所以你在自己机器上造报文时，凡是端口/IP 这类多字节数字都要"翻成正的"再发出去（hton 系列），收到再翻回来（ntoh 系列）。

---

## 6. socket、bind、listen、accept 的作用；accept 返回的套接字与监听套接字的区别

**原题：**
> 网络编程函数：服务器端socket、bind、listen、accept的作用分别是什么？accept返回的套接字与监听套接字有何不同？

**答案：**

| 函数       | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| `socket()` | 创建一个套接字（网络通信的"文件描述符"），指定协议族、类型（SOCK_STREAM/SOCK_DGRAM）、协议 |
| `bind()`   | 把**本地地址（IP + 端口）**绑定到该套接字上，让操作系统知道"这个 fd 代表哪个 IP:端口" |
| `listen()` | 将套接字从"主动连接"变为"被动监听"，并设置**已完成连接队列**（backlog，最多同时排队的连接数） |
| `accept()` | 从已完成连接队列中**取出一个连接**，返回一个**新的已连接套接字 fd**，用于与该客户端通信 |

```cpp
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);   // 1. 创建
bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)); // 2. 绑定 IP:端口
listen(listen_fd, SOMAXCONN);                      // 3. 监听（backlog）
while (true) {
    int conn_fd = accept(listen_fd, ...);          // 4. 接受连接，得到新 fd
    // 用 conn_fd 与客户端收发数据
}
```

**accept 返回的套接字 vs 监听套接字的区别：**

|      | 监听套接字（listen_fd）      | 已连接套接字（conn_fd）                |
| ---- | ---------------------------- | -------------------------------------- |
| 职责 | **只负责"接客"**——接受新连接 | **只负责"陪客"**——与某个特定客户端通信 |
| 读写 | 不参与数据读写               | 负责该客户端的读写                     |
| 数量 | 服务器通常只有一个           | 每来一个客户端 accept 产生一个         |
| 关联 | 不变，持续监听               | 每个连接独立，与具体客户端四元组对应   |

> 高并发要点：`accept` 是阻塞的（无连接时阻塞），且一个监听 fd 只能服务一个 accept；**Reactor/epoll 模型**就是把监听 fd 和所有已连接 fd 一起交给 epoll 监听，accept 只在监听 fd 可读时被调用一次，从而支撑成千上万连接。

**通俗解释**：监听套接字是**前台总机**（只负责接电话、转接），accept 返回的套接字是**专门这条通话的分机**（只跟这个客户通话）。前台一个就够了，分机来一个客户开一部。用 epoll 做高并发时，前台和每部分机都登记在"来电提醒表"里，谁响铃处理谁。

---

# 作业6：IO多路复用 & ReactorV1‑V2

## 1. 写出 select、poll、epoll 的核心函数原型并说明参数含义

**原题：**
> 函数接口：写出select、poll、epoll的核心函数原型，并说明各参数含义。

**答案：**

**select：**
```c
int select(int nfds,
           fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
           struct timeval *timeout);
```
- `nfds`：要监视的最大文件描述符 **+1**；
- `readfds/writefds/exceptfds`：可读/可写/异常集合。**输入输出双用**——传入要监听哪些 fd，返回时被内核改写成"哪些就绪"（就绪集合直接覆盖监听集合）；
- `timeout`：超时（NULL 永久阻塞，0 立即返回，否则最多等这么久）；
- 返回：就绪 fd 个数，超时返回 0，出错返回 -1。

**poll：**
```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
struct pollfd {
    int fd;          // 要监听的 fd
    short events;    // 输入：关心的事件（POLLIN/POLLOUT...）
    short revents;   // 输出：实际就绪的事件（内核回填）
};
```
- `fds`：pollfd 数组（**输入输出分离**：events 是输入、revents 是输出）；
- `nfds`：数组元素个数；
- `timeout`：毫秒（-1 永久阻塞）。

**epoll：**
```c
int epoll_create1(int flags);                     // 创建 epoll 实例（flags: EPOLL_CLOEXEC）
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

struct epoll_event {
    uint32_t events;   // EPOLLIN / EPOLLOUT / EPOLLET ...
    epoll_data_t data; // union: fd / ptr / u32 ...
};
```
- `epoll_ctl`：对 epoll 实例**增删改** fd，`op` 为 `EPOLL_CTL_ADD/MOD/DEL`，`event` 描述关心的事件；
- `epoll_wait`：等待就绪事件，**就绪的事件写入用户提供的 `events` 数组**，返回就绪个数；`maxevents` 是数组容量；`timeout` 毫秒。

**通俗解释**：select/poll 每次调用都要"**把一整叠名单交给内核，让内核从头到尾翻一遍**再还回来"；epoll 则是"**一开始就登记名单（epoll_ctl），平时内核只通知有动静的（epoll_wait 返回就绪列表）**"——所以 epoll 在人多的场景下省事得多。

---

## 2. select / poll / epoll 优缺点对比

**原题：**
> 优缺点对比：从以下角度对比三种IO复用：
> - 文件描述符上限
> - 监听集合与就绪集合是否分离
> - 就绪检测效率（轮询 vs 回调）
> - 适用场景（连接数多少、活跃度高低）

**答案：**

| 对比维度                       | select                                                       | poll                                               | epoll                                                        |
| ------------------------------ | ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| **fd 上限**                    | 受 `FD_SETSIZE`（通常 **1024**）硬限制                       | 受系统级 `ulimit -n` 限制（一般几万~百万）         | 受系统级限制，**无 select 式硬上限**，可支撑百万级           |
| **监听集合与就绪集合是否分离** | **不分离**：fd_set 输入输出共用，每次调用都要重建/拷贝，且就绪会**覆盖**监听集 | **分离**：`events`（输入）与 `revents`（输出）分开 | **分离**：监听集合由内核红黑树维护（注册一次即可），epoll_wait 只输出就绪数组 |
| **就绪检测效率**               | **内核线性扫描**全部 fd，O(n)，n 越大越慢                    | **内核线性扫描**全部 fd，O(n)                      | **回调驱动**：有事件的 fd 由内核回调挂进就绪链表，epoll_wait 只取就绪项，**O(就绪数)** |
| **适用场景**                   | 连接数少（<1024）、简单场景                                  | 连接数中等                                         | **连接数多、活跃度低**的高并发服务器（大量连接长时间无事件） |

**选型结论：**
- 连接少（几十~几百）且简单：select 够用；
- 连接中等：poll（解决 fd 上限）；
- **高并发（上万连接、低活跃）**：必须 epoll——"**多路复用 + 回调就绪链表**"把复杂度从 O(全部连接) 降到 O(有事件的连接)。

**通俗解释**：select 是"人工点名"，每次把所有学生（fd）从头点一遍，人多了累死（O(n)），而且名单一次只能写 1024 个；poll 允许名单更长、更整齐，但还是要"全校点名"；epoll 是"**装了门铃**"——平时没人敲门（有事件）就睡觉，谁按门铃谁响，只处理响的（O(就绪数)），人再多也不怕。

---

## 3. epoll 原理：红黑树 + 就绪队列，epoll_ctl 和 epoll_wait 的工作过程，为什么说 epoll 是事件驱动

**原题：**
> epoll原理：epoll使用红黑树和就绪队列，解释epoll_ctl和epoll_wait的工作过程，为什么说epoll是事件驱动？

**答案：**

**内核数据结构（一个 epoll 实例 = 一个 eventpoll 对象）：**
- **红黑树**：存放所有**注册（被监听）的 fd** 及其关注事件，用于快速增删查（O(log n)）；
- **就绪链表（就绪队列）**：存放**有事件发生、等待被取走的 fd**；
- **等待队列**：存放正在 `epoll_wait` 上睡眠的进程。

**epoll_ctl 的工作过程：**
```
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev)
  → 在内核红黑树中插入/查找/删除该 fd 的节点
  → 建立 fd 与该事件对应的"回调函数"绑定（ep_poll_callback）
  → 注册到该 fd 所在的设备（网卡驱动）的事件回调上
```
之后每次 `epoll_wait` 时，内核**不需要再遍历**这些 fd——因为"关注事件"已经登记在树里。

**有事件发生时（内核回调）：**
当某个 fd 有数据可读/可写（比如网卡驱动收到数据包），会调用事先注册的 **`ep_poll_callback` 回调函数**，把该 fd 挂到**就绪链表**尾部；如果此时有进程在 `epoll_wait` 上睡觉，还会**唤醒它**。

**epoll_wait 的工作过程：**
```
epoll_wait(epfd, events, maxevents, timeout)
  → 检查就绪链表是否为空
  → 非空：把就绪链表中的 fd 及其事件拷贝到用户 events 数组，返回个数（拷贝后可以清空）
  → 为空：把当前进程挂入等待队列睡眠；直到有事件回调唤醒，再重复上面步骤
```

**为什么说 epoll 是"事件驱动"：**
因为它是**"有事件才处理"**的——内核在事件发生**那一刻**通过回调把 fd 丢进就绪队列，`epoll_wait` 只返回**真正就绪**的 fd，且复杂度只与**就绪数**有关，与**总连接数**无关。对比 select/poll 需要**主动轮询全部 fd**（即使 9999 个都没事，也要扫 10000 次），epoll 是典型的"**被动等事件**"模型。

**通俗解释**：红黑树是"**登记册**"（谁被关注、关注什么，一查即得）；就绪队列是"**叫号屏**"（谁有动静谁上屏）。有人（网卡）喊"号！"，系统就把号挂上叫号屏并摇醒服务员（epoll_wait），服务员只看屏上有哪几个号就服务哪几个——**服务量 = 喊号的个数，而不是全部客人的个数**。这就是"事件驱动"。

---

## 4. 五种 IO 模型：画出模型图，指出每个模型在数据准备和数据拷贝两个阶段是否阻塞

**原题：**
> 五种IO模型：画出阻塞式IO、非阻塞式IO、IO多路复用、信号驱动IO、异步IO的模型图，并指出每个模型在数据准备和数据拷贝两个阶段是否阻塞。

**答案：**

**IO 两个阶段**（以 read 为例）：
- **阶段1 数据准备**：等待数据到达内核缓冲区（如等待网卡收包）；
- **阶段2 数据拷贝**：把数据从内核缓冲区拷到用户缓冲区。

**① 阻塞式 IO（Blocking IO）**
```
应用线程:  read() ──── 全程阻塞 ────▶ 返回数据
           [阶段1 等待数据: 阻塞] [阶段2 拷贝: 阻塞]
```
- 数据准备：**阻塞**；数据拷贝：**阻塞**（同步）。

**② 非阻塞式 IO（Non-blocking IO）**
```
应用线程:  read() → EAGAIN(没数据，立刻返回) → 忙轮询/睡一下 → read() → 有数据 → 拷贝 → 返回
           [阶段1 等待数据: 不阻塞进程（轮询，但耗CPU）] [阶段2 拷贝: 阻塞]
```
- 数据准备：**不阻塞**（立即返回 EAGAIN，靠用户态轮询）；数据拷贝：**阻塞**（同步）。

**③ IO 多路复用（select/poll/epoll）**
```
应用线程:  select/epoll_wait(阻塞等"多个fd中任一个有数据") → 就绪 → read() → 拷贝 → 返回
           [阶段1 等待数据: 阻塞在select/epoll_wait] [阶段2 拷贝: 阻塞(仍由应用线程read)]
```
- 数据准备：**阻塞**（但能同时等很多 fd）；数据拷贝：**阻塞**（同步）。

**④ 信号驱动 IO（SIGIO）**
```
应用线程:  注册SIGIO → 干别的 → 内核发SIGIO信号"数据好了" → read() → 拷贝 → 返回
           [阶段1 等待数据: 不阻塞（信号通知）] [阶段2 拷贝: 阻塞]
```
- 数据准备：**不阻塞**（内核用信号通知）；数据拷贝：**阻塞**（同步）。

**⑤ 异步 IO（AIO / io_uring）**
```
应用线程:  aio_read(告诉内核"拷完再通知我") → 干别的 → 内核完成"准备+拷贝" → 回调/事件通知 → 直接拿数据
           [阶段1 等待数据: 不阻塞] [阶段2 拷贝: 不阻塞（内核完成，真正的异步）]
```
- 数据准备：**不阻塞**；数据拷贝：**不阻塞**（异步）。

**对比表：**

| 模型        |  数据准备是否阻塞  | 数据拷贝是否阻塞 | 本质     |
| ----------- | :----------------: | :--------------: | -------- |
| 阻塞 IO     |       ✅ 阻塞       |      ✅ 阻塞      | 同步     |
| 非阻塞 IO   |     ❌（轮询）      |      ✅ 阻塞      | 同步     |
| IO 多路复用 | ✅ 阻塞（等一批fd） |      ✅ 阻塞      | 同步     |
| 信号驱动 IO |   ❌（信号通知）    |      ✅ 阻塞      | 同步     |
| 异步 IO     |         ❌          |        ❌         | **异步** |

**通俗解释**：前四种都是"**数据还是你自己动手搬**（read 拷到用户态）"——所以都是同步，区别只在"等数据"的方式；只有第五种 AIO/io_uring 是"**快递直接帮你把包裹搬进家门再按门铃**"——等和搬都交给内核，才是真正异步。Linux 下 `io_uring`（内核 5.1+）是目前真正的异步 IO 主流方案。

---

## 5. Reactor V1：Socket、InetAddress、Acceptor、TcpConnection、SocketIO 各负责什么？为什么需要 SocketIO 单独封装读写？

**原题：**
> Reactor V1类设计：Socket、InetAddress、Acceptor、TcpConnection、SocketIO各负责什么功能？为什么需要SocketIO单独封装读写？

**答案：**

| 类                | 职责                                                         |
| ----------------- | ------------------------------------------------------------ |
| **Socket**        | 封装**套接字 fd**：创建（`socket()`）、关闭（RAII 析构 close）、设置非阻塞（`setNonBlock()`）、获取 fd、设置 SO_REUSEADDR 等选项。它只管理"一个 fd 的生死与属性"。 |
| **InetAddress**   | 封装**IP + 端口**（`sockaddr_in` 的包装）：`InetAddress(port)`、`InetAddress(ip, port)` 构造；`toIp()/toPort()/toIpPort()`；地址转换（`inet_pton` 等）。解决"把 127.0.0.1:8080 填进 sockaddr_in"这类脏活。 |
| **Acceptor**      | 封装**服务器监听 + 接受连接**：内部持有一个监听 Socket 和非阻塞 fd；`listen()`、`accept()`；通过回调 `setNewConnectionCallback` 把新连接的 connfd 交给上层（EventLoop/TcpServer）。只负责"接客"，不负责通信。 |
| **TcpConnection** | 封装**一个已建立的连接**：持有一个已连接 Socket、SocketIO、连接状态、事件回调（可读/可写/关闭）；负责"把底层 fd 事件翻译成上层回调"、管理连接生命周期。 |
| **SocketIO**      | 封装**读写原语**：`readn/writen/readLine/read/write`，处理 EINTR 重试、EAGAIN、读半包/写半包等细节。 |

**为什么需要 SocketIO 单独封装读写：**

1. **TCP 是字节流、read/write 是"尽力而为"**：一次 `read()` 可能只读到半个消息（拆包），一次 `write()` 可能只写出去一部分（写不全）；还可能在信号打断时返回 EINTR、非阻塞时返回 EAGAIN。这些**边角逻辑**（循环读写、EINTR 重试、EAGAIN 处理、按字节数凑满）非常繁琐且容易写错。
2. **职责分离（SRP）**：把"怎么把 N 个字节可靠地读进来/写出去"的细节从 `TcpConnection` 中抽出来，`TcpConnection` 只管"协议语义、事件分发、生命周期"，`SocketIO` 只管"字节搬运"。这也是 Reactor 课程把它单独成类的直接原因。
3. **可复用、可测试**：同一套 `readn/writen` 逻辑可以被很多连接复用，且便于单元测试。

**通俗解释**：Socket 是"**门牌 + 门锁**"（fd 的生死属性），InetAddress 是"**地址本**"（IP:端口），Acceptor 是"**前台总机**"（接客），TcpConnection 是"**专用分机**"（单个客户通话），SocketIO 是"**话务员规范手册**"——专门解决"一次只说半句话怎么办、对方忙线（EAGAIN）怎么办、被打断（EINTR）怎么办"这些琐碎又致命的细节。单独抽出来，是因为这些琐碎细节**太多太危险**，不该和"连接管理"混在一起。

---

## 6. Reactor V2：handleNewConnection 中为什么要 addEpollReadFd？为什么要存 connfd→TcpConnectionPtr 映射？handleMessage 如何区分断开和消息到达？

**原题：**
> Reactor V2关键代码：在EventLoop::handleNewConnection()中，创建TcpConnection后为什么要调用addEpollReadFd？为什么要存储connfd‑>TcpConnectionPtr的映射？在handleMessage中如何区分连接断开和消息到达？

**答案：**

**为什么创建 TcpConnection 后要调用 `addEpollReadFd`：**

新 accept 出来的 connfd **默认没有被 epoll 监听**。`addEpollReadFd(connfd)` 就是把该 fd **注册进 epoll 的读事件监听集合**（`epoll_ctl(ADD, EPOLLIN)`）。否则客户端发来的数据到了内核，EventLoop 根本收不到通知，这个连接就"聋了"。所以**每个新连接必须注册读事件，之后才能被事件循环驱动**。

**为什么要存储 `connfd → TcpConnectionPtr` 的映射：**

用 `std::map<int, TcpConnectionPtr>` 把**文件描述符和它对应的连接对象**关联起来，原因有三：
1. **epoll 只返回 fd**：`epoll_wait` 返回的事件里只有 fd 编号（或 data 指针），EventLoop 需要根据 fd **快速找到**对应的 TcpConnection 对象，才能调用它的处理函数；
2. **管理连接生命周期**：映射里存 `shared_ptr`，保证连接对象在事件处理期间不会被提前析构（引用计数存活）；
3. **连接断开时清理**：收到关闭事件时，要能从映射里**删除**该 fd，避免 fd 复用后误判。

```cpp
// 伪代码
std::map<int, std::shared_ptr<TcpConnection>> conns_;  // fd → 连接

void handleNewConnection(int connfd) {
    auto conn = std::make_shared<TcpConnection>(connfd, ...);
    conns_[connfd] = conn;            // 保存映射
    epoll_add_read(connfd);           // 注册读事件
}

void handleRead(int fd) {
    auto conn = conns_[fd];           // 用 fd 找回连接对象
    conn->handleMessage();            // 交给连接处理
}
```

**handleMessage 中如何区分连接断开和消息到达：**

核心是看 **`read()` 的返回值**（这是 TCP 断开的"约定"——对端正常关闭后 read 返回 0）：

```cpp
void TcpConnection::handleMessage() {
    ssize_t n = socketIO_.read(...);      // 尝试读数据
    if (n > 0) {
        // ① 读到数据 → 消息到达 → 调用 onMessage 回调交给业务
        onMessageCallback_(shared_from_this(), inputBuffer_);
    } else if (n == 0) {
        // ② 读到 0 → 对端已关闭（FIN）→ 调用 onClose，从映射删除、关闭 fd
        onCloseCallback_(shared_from_this());
        // 从 EventLoop 的 map 中 erase(fd)，epoll_ctl DEL
    } else {
        // ③ n < 0：
        //    EINTR → 重试；EAGAIN/EWOULDBLOCK → 本次无数据，非错误，忽略
        //    其他错误 → 按断开处理
        if (errno == EAGAIN || errno == EWOULDBLOCK) { /* 稍后再读 */ }
        else if (errno != EINTR) { /* 真正错误 → 关闭连接 */ }
    }
}
```

**通俗解释**：epoll 是"来电总机"，`addEpollReadFd` 就是给新客户**开通来电提醒**；`connfd→TcpConnectionPtr` 的映射是"**来电极 = 通话记录**"，响铃时一查就知道是谁；`handleMessage` 里判断"n>0 是说话、n==0 是挂断"——TCP 的规矩就是 **read 返回 0 = 对方挂了**，这是最可靠的分手信号。

---

## 7. 三个半事件指什么？Reactor V2 中如何通过回调处理？TcpConnection 的三个回调如何从 EventLoop 传递并执行？

**原题：**
> 三个半事件：TCP网络编程中"三个半事件"指什么？Reactor V2中如何通过回调函数处理这三个事件？TcpConnection中的三个回调是如何从EventLoop传递并执行的？

**答案：**

**三个半事件（muduo 风格的经典总结）：**

1. **连接建立**（新客户端 connect 成功）；
2. **消息到达**（对端发来可读数据）；
3. **连接关闭**（对端 FIN / 本端关闭）；
4. **半个事件：消息发送完成**——之所以算"半个"，是因为**大多数时候数据能直接写进内核发送缓冲，无需特别处理**；只有当输出缓冲写满（`write` 返回 EAGAIN）时才需要关注"可写事件、把剩余数据发完"。它不是每次通信都必然发生的独立事件，所以叫"半个"。

**Reactor V2 如何通过回调处理这三个事件：**

Reactor V2 把"事件发生"与"事件处理"解耦成**回调注册 + 回调分发**：

```
epoll_wait 返回就绪 fd
      │
      ▼
EventLoop::loop()
      ├─ 监听fd可读        → Acceptor::accept() → 新连接 → 建 TcpConnection
      │                      → 触发【连接建立】回调 onConnection
      ├─ connfd 可读       → TcpConnection::handleMessage()
      │                      → 读数据 → 触发【消息到达】回调 onMessage
      └─ read 返回 0/错误   → TcpConnection 关闭流程
                             → 触发【连接关闭】回调 onClose
```

**TcpConnection 的三个回调如何从 EventLoop 传递并执行：**

三个回调类型一般是：
```cpp
using ConnectionCallback = std::function<void(const TcpConnectionPtr&)>;   // 连接建立
using MessageCallback    = std::function<void(const TcpConnectionPtr&,
                                             std::string& /*或Buffer*/)>; // 消息到达
using CloseCallback      = std::function<void(const TcpConnectionPtr&)>;   // 连接关闭
```

传递链：**用户（EchoServer/TcpServer）注册回调 → TcpServer 保存 → 创建 TcpConnection 时设置到连接对象 → 事件发生时 EventLoop 调用连接对象的对应方法 → 方法内部调用用户回调**：

```cpp
// TcpServer 中：
void TcpServer::setConnectionCallback(ConnectionCallback cb) { connectionCallback_ = std::move(cb); }

// 接受新连接时：
auto conn = std::make_shared<TcpConnection>(connfd, ...);
conn->setConnectionCallback(connectionCallback_);   // ① 回调从 TcpServer 传给 TcpConnection
conn->setMessageCallback(messageCallback_);
conn->setCloseCallback(closeCallback_);

// EventLoop 事件循环中：
conn->handleMessage();    // ② 事件发生时调用连接方法
// ③ TcpConnection::handleMessage 内部：
onMessageCallback_(shared_from_this(), buffer);     // 最终执行用户注册的回调
```

**通俗解释**：三个半事件是"**TCP 世界一共就这几件事**"——有人来、有人说话、有人走、外加半个"话说完没说完"。Reactor 的做法是把"谁来处理这些事"做成**回执（回调函数）**：用户提前写好"收到消息怎么办"，塞给连接对象（注册），事件一发生，事件循环只负责"喊一声"（调用 handleXxx），具体怎么做由用户的回调决定——这就是"**事件循环 + 回调**"的整套范式。

---

## 8. 为什么向回调传 this 必须用 shared_from_this()，而绝不能 std::shared_ptr(this)？

**原题：**
> 在TcpConnection中注册三个半事件回调时，为什么向回调传递this指针必须通过shared_from_this()，而绝对不能直接写std::shared_ptr(this)？（考察重复析构/双重control block问题）

**答案：**

**核心问题：双重 control block / 重复析构（double free）。**

`std::shared_ptr` 的引用计数存放在一个**独立的"控制块"（control block）**里。**一个对象只能对应一个控制块**。

- `TcpConnection` 本身很可能已经通过 `std::make_shared<TcpConnection>` 创建，它已经有一个控制块（引用计数 = 1）；
- 如果在回调里直接写 `std::shared_ptr<TcpConnection>(this)`，等于**为同一个对象又创建了一个全新的、互相独立的控制块**（引用计数也是 1）；
- 结果：**同一个 `this` 被两个毫不相干的 shared_ptr 管理**。当第一个控制块的引用计数归零 → `delete this`；之后第二个控制块归零 → **再次 `delete this`** → **double free 崩溃**（未定义行为）。而且内存泄漏/崩溃的行为不可预测，极难排查。

```cpp
// 错误示范（产生两个控制块）：
std::shared_ptr<TcpConnection> bad(this);   // 新造一个控制块，计数=1
// ... 这个 bad 析构时 delete this；而外层 make_shared 的控制块也 delete this → double free

// 正确示范：shared_from_this() 返回的是"与既有控制块共享同一个计数"的 shared_ptr
void TcpConnection::handleMessage() {
    onMessageCallback_(shared_from_this(), inputBuffer_);  // 引用计数 +1，但控制块只有一个
}
```

**正确做法：**

1. `TcpConnection` **继承 `std::enable_shared_from_this<TcpConnection>`**；
2. 在需要把自身 shared_ptr 传给回调的地方调用 **`shared_from_this()`**；
3. `shared_from_this()` 内部通过**既有控制块的 `weak_ptr` 提升**得到 shared_ptr，保证**与已有控制块共享同一个引用计数**，且引用计数必然 ≥ 1（因为对象一定先被某个 shared_ptr 持有才能调用它）。

> 注意陷阱：`shared_from_this()` **只能在对象已经被某个 shared_ptr 管理之后调用**（不能在构造函数里调用，此时还没有控制块），否则抛 `std::bad_weak_ptr`。

**通俗解释**：`std::shared_ptr(this)` 就像给同一个人**办了两张互不相识的身份证**，两张证各自记录"这个人还活着"，结果两边都以为"就剩我在管他"，各 delete 一次——**同一间房子拆了两遍，必然崩塌**。`shared_from_this()` 则是"**查同一本户口本**"（同一个控制块），每次都去公共台账上 +1/-1，永远只 delete 一次。

---

## 9. epoll 的 LT 与 ET 本质区别；ET 下为什么必须非阻塞 + 循环读写到 EAGAIN？只读一次会怎样？定长协议如何处理粘包/拆包？

**原题：**
> epoll的LT与ET触发模式有何本质区别？为什么在ET模式下，套接字必须设置为非阻塞（O_NONBLOCK）且read/write必须循环读写直到遇到EAGAIN / EWOULDBLOCK？如果只读一次会造成什么后果？定长协议来处理粘包/拆包？

**答案：**

**LT 与 ET 的本质区别：**

- **LT（水平触发 Level Triggered，默认）**：只要 fd **"还有可读数据"（或可写空间）**，`epoll_wait` 就会**反复**通知你——哪怕你上次没读完，下次还通知。编程简单、不易漏数据。
- **ET（边沿触发 Edge Triggered）**：只有在**状态发生变化的那一刻**（从"没有数据"变为"有数据"）**通知一次**，之后即使数据没读完，**不再通知**，直到下次状态再次变化。效率更高（减少重复唤醒），但容易漏数据。

**为什么 ET 下必须非阻塞 + 循环读写到 EAGAIN：**

因为 ET **只通知一次**，你必须趁这次通知**把所有数据一次搬完**：
1. **必须循环 read**：一次 `read` 可能读不完（缓冲区还有剩余），既然不会再被通知，就必须**循环读直到读尽**；
2. **读到 EAGAIN/EWOULDBLOCK 才能停**：非阻塞模式下，当内核缓冲区被读空、返回 `EAGAIN`/`EWOULDBLOCK` 时，才说明"这一轮数据读完了"，此时停止循环；
3. **必须非阻塞（O_NONBLOCK）**：如果套接字是阻塞的，循环里最后一次 read 会因为"没数据了"而**永久阻塞**（而不是返回 EAGAIN），整个线程卡死。所以 ET 与"非阻塞 + 读到 EAGAIN"是**配套强制要求**。

**如果只读一次会怎样：**

会**丢数据 + 连接卡死**：内核缓冲区里剩余的数据永远留在缓冲里，而 ET 又不会再通知你"还有数据"——剩余数据**无人处理**，而且对方还会一直等你的回复。对于协议而言，半截消息永远凑不齐 → 协议解析卡死/超时。

**定长协议处理粘包/拆包：**

**定长协议**：每个消息 = **固定长度的包头**（含总长度字段）+ **数据体**。接收侧用"缓冲 + 状态机"：

```
① 接收到的字节先全部塞进 InputBuffer
② 循环：
   - 缓冲里不足 sizeof(Header)（定长包头）→ 等下一次 read（拆包）
   - 够包头 → 解析出 len（数据体长度）
   - 缓冲里不足 len → 等下一次 read（拆包）
   - 够 len → 从缓冲取出完整"包头+数据体"交给业务处理（粘包时缓冲里可能还有下一个包，继续循环）
③ 剩下不足一个完整包的数据留在缓冲，等下一次 read 凑齐
```

这样无论网络如何拆/粘包，应用层拿到的一定是**一个个完整的消息**。

**通俗解释**：LT 是"**菜还没端完就一直在你耳边提醒**"（反复通知），ET 是"**锅一开只叫你一次，你必须一口气把菜全盛走**"。ET 下盛菜（read）必须"盛到锅里真没了（EAGAIN）才停"，而且锅必须是"无盖可随时伸手"（非阻塞），否则最后一勺会卡在锅里（阻塞）。定长协议就像**每个快递箱都贴了"内容长度"标签**——收到箱子先看标签（包头），标签说 100 字节，就等凑够 100 字节再拆箱，不管快递是分几车送来的（拆包）还是几箱同车来（粘包），都能正确拼出完整货物。

---

## 10. TCP 字节流无边界：应用层应如何设计 InputBuffer / OutputBuffer 与包头定长协议处理粘包/拆包？

**原题：**
> TCP是字节流协议，没有边界。在SocketIO::readLine或高并发收发时，如果一次read读到了半个数据包或两个粘在一起的数据包，应用层应如何设计输入/输出缓冲区（InputBuffer/OutputBuffer）与包头定长协议来处理粘包/拆包？

**答案：**

**总体思想**：TCP 只保证字节顺序，不保证"一次 read = 一个消息"。所以应用层必须**自己加边界**，用**缓冲区 + 包头定长协议**把"字节流"切回"消息流"。

**① 输入缓冲区 InputBuffer（接收侧）——处理"读半包 / 粘包"：**

```cpp
class InputBuffer {
public:
    // 底层 read 到的原始字节先全倒进缓冲，再按协议解析
    void append(const char* data, size_t len) {
        buf_.append(data, len);
    }
    // 从缓冲中尝试取出一个完整消息；不足则返回"未完整"
    std::optional<std::string> tryGetMessage() {
        const size_t hdrLen = sizeof(Header);        // 定长包头（如4字节长度）
        while (buf_.size() >= hdrLen) {
            Header hdr;  memcpy(&hdr, buf_.data(), hdrLen);
            uint32_t bodyLen = ntohl(hdr.length);     // 长度字段（注意网络序！）
            size_t total = hdrLen + bodyLen;
            if (buf_.size() < total) break;          // 拆包：数据还不够，等下一次read
            std::string msg(buf_.data(), total);     // 粘包：可能一次拼出多个，while 继续
            buf_.erase(0, total);
            return msg;
        }
        return std::nullopt;                          // 不完整，等更多数据
    }
private:
    std::string buf_;   // 实际工程用两块连续缓冲（readable/writable区）避免反复移动
};
```

**② 输出缓冲区 OutputBuffer（发送侧）——处理"write 写不完 / EAGAIN"：**

```cpp
class OutputBuffer {
public:
    void append(const char* data, size_t len) { buf_.append(data, len); }

    // 尝试把缓冲尽量发出去；EAGAIN 时剩余数据留在缓冲，等待"可写事件"继续发
    void sendAll(int fd) {
        while (!buf_.empty()) {
            ssize_t n = ::write(fd, buf_.data(), buf_.size());
            if (n > 0) {
                buf_.erase(0, n);
            } else if (n < 0 && errno == EAGAIN) {
                break;                 // 内核缓冲满，剩余留缓冲，注册可写事件，等能写再发
            } else if (n < 0 && errno != EINTR) {
                break;                 // 真正错误（如对端关闭）
            } // EINTR 重试
        }
        // 若 buf_ 非空 → 打开 EPOLLOUT 监听；发空 → 关闭 EPOLLOUT
    }
private:
    std::string buf_;
};
```

**③ 包头定长协议格式：**

```
┌──────────────┬──────────────────────────────┐
│   Header     │           Body               │
│  (定长,如8B) │       (变长,由Header给出)      │
│ magic|type|len│       实际消息数据            │
└──────────────┴──────────────────────────────┘
len = 网络字节序的 4 字节无符号整数，表示 Body 的字节数
```

**接收处理流程（状态机）：**
```
read → append到InputBuffer → while(tryGetMessage) { 交给业务 }
       │
       ├─ 粘包：缓冲里同时有多个完整包 → while 循环一个个切出来
       ├─ 拆包：只凑齐包头、body不够 → 留在缓冲，等下一次read再拼
       └─ 半包头：连包头都不够 → 留在缓冲等待
```

**工程要点（高并发下）：**
1. 缓冲区避免每次 `erase(0,n)` 的 O(n) 移动——用**双指针（readIndex/writeIndex）**的环形/分片缓冲；
2. 长度字段一律 `ntohl` 转回主机序；
3. 要防"恶意超大长度"（`len` 设上限，防内存耗尽）；
4. 发送侧大包/慢客户端要用 OutputBuffer + EPOLLOUT，避免阻塞写。

**通俗解释**：InputBuffer 像"**分拣仓库**"——快递（字节）到了先放仓库，仓库按"箱子上的长度标签"（包头）凑齐一个完整箱子就交给业务，凑不齐的（拆包）留在仓库等下一车；一车拉来好几个箱子（粘包）就一个个拆。OutputBuffer 像"**发件缓冲站**"——要寄的东西先放站里，车一次装不下（EAGAIN）就留站里，等有空车（可写事件）再继续运，保证不丢件。

---

# 作业7：Reactor V3

## 1. V3 与 V2 的类图差异在哪里？为什么说没有本质区别？

**原题：**
> V3变化：Reactor V3与V2的类图差异在哪里？为什么说没有本质区别？

**答案：**

**类图差异：**

V3 相对 V2 主要新增了**多线程/业务处理**相关的类：

| V2                                                           | V3 新增/变化                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Socket / InetAddress / Acceptor / EventLoop / TcpConnection / SocketIO | 新增 **ThreadPool（线程池）**、**TaskQueue（任务队列）**、**Thread（线程封装）** |
| 业务逻辑直接在 IO 线程（EventLoop 线程）同步执行             | 业务逻辑被封装成任务（`std::function<void()>`）投递到 ThreadPool，由**工作线程**执行 |
| TcpServer 只持有 EventLoop                                   | TcpServer 同时持有 **EventLoop + ThreadPool**，可组合出多线程架构 |

**为什么说"没有本质区别"：**

无论 V2 还是 V3，底层都是**同一个 Reactor 事件驱动模型**：
- 都有一个 **EventLoop** 负责 `epoll_wait` 事件循环；
- 事件到来都通过**回调分发**（onConnection / onMessage / onClose）；
- 网络 IO（收发字节）仍然只发生在 IO 线程。

V3 改变的只是"**业务处理在哪个线程跑**"（从 IO 线程挪到线程池），以及组件组织的粒度（多出来线程池类），**事件驱动核心、IO 模型、回调范式完全没变**。所以 V3 是 V2 的"**结构演进**"而非"**本质革命**"——本质仍是"单/多事件循环 + 回调"的 Reactor。

**通俗解释**：V2 到 V3 就像把"柜台收银"改成了"收银台 + 后场仓库分拣"——前台（EventLoop）还是那套"来了客人就喊人"的流程，只是把"算账/打包"（业务）挪到后场（线程池）做了。前台的服务流程（Reactor）本质上一点没变。

---

## 2. 当业务逻辑较复杂时，Reactor V3 有什么致命缺陷？为什么会导致串行处理？

**原题：**
> 性能瓶颈：当业务逻辑（如加解密、数据库操作）较复杂时，Reactor V3有什么致命缺陷？为什么会导致串行处理？

**答案：**

**致命缺陷：业务逻辑仍与事件循环耦合（在 IO 线程/或未解耦的回调链上同步执行），复杂业务会"卡死"事件循环，导致整机退化为串行处理。**

具体表现与原因（这是理解 V4 引入 eventfd 的关键背景）：

1. **一个慢任务阻塞整个 EventLoop**：如果业务处理（加解密、数据库查询）直接放在 IO 线程的 `onMessage` 回调里同步执行，那么这段耗时的计算会**阻塞 `epoll_wait` 之后的整个事件分发循环**。该 EventLoop 上注册的**所有连接**的读写事件都必须排队等这个慢任务做完——**一个请求卡住，所有请求排队**（队头阻塞 Head-of-Line Blocking），高并发下退化成近似**单线程串行**。

2. **即使引入"朴素线程池"，回发路径仍可能串行**：把业务放进线程池后，若线程池 worker 处理完**直接在 worker 线程调用 `TcpConnection::send()`**，就会跨线程访问/修改 `OutputBuffer`、epoll 事件注册等**非线程安全**的网络层状态——为避免数据竞争，要么给网络层加"一把大锁"（所有 worker 在回发路径上互相等待 → **发送串行化**），要么就存在未定义行为。这正是 V3 的"半同步/半异步"没有做透的地方：**结果回发没有回到它该在的 IO 线程**。

3. **线程池任务队列本身也可能成为瓶颈**：如果业务任务量远超线程数，任务在单一队列里排队，加上锁竞争，吞吐上不去，高并发下同样表现为"串行排队"。

**核心结论**：V3 的致命缺陷 = **"耗时业务 + 未解耦的回发路径"导致事件循环或发送路径被串行化**。解决方向（即 V4）：
- 业务逻辑**彻底搬到线程池**（worker 并行计算，互不阻塞 IO 线程）；
- worker 处理完**不能直接跨线程 send**，而是把"发送动作"作为任务**回投到连接所属的 EventLoop**（这正是 V4 的 `eventfd + pendingFunctors` 要解决的）。

**通俗解释**：V3 就像"**只有一个窗口的银行**"——前面那位客户（一个连接的复杂业务：加解密/查数据库）迟迟不走，后面排队的客户（所有其他连接）全被堵死，整家银行实际就是"一个一个办"（串行）。V4 的解法是：把"算账"这种重活交给**后台柜台**（线程池）去做，前台（EventLoop）只负责"接单、喊号"；算好的结果再**递回前台**通知客户，全程不堵窗口。

---

## 3. 引入线程池后，业务逻辑放在哪个环节？为什么要将 TcpConnection 和消息一起传给线程池？线程池处理完后如何发回结果？

**原题：**
> 改进思路：引入线程池后，业务逻辑处理放在哪个环节？为什么需要将TcpConnection对象和消息一起传递给线程池？线程池处理完后如何将结果发回给客户端？

**答案：**

**① 业务逻辑放在哪个环节：**

放在 **`onMessage` 回调的"投递动作"之后、线程池工作线程中**执行。EventLoop 的 `onMessage` 只做**一件事**：把任务投递到线程池任务队列，然后**立即返回**，绝不阻塞事件循环：

```cpp
// EventLoop（IO线程）中的 onMessage —— 只负责"投递，不干活"
void TcpServer::onMessage(const TcpConnectionPtr& conn, std::string& msg) {
    // 把 "连接对象 + 消息" 一起打包成任务投给线程池
    threadPool_.addTask([conn, msg]() mutable {
        // ↓↓↓ 业务逻辑（加解密/数据库/计算）在这里，由工作线程执行 ↓↓↓
        std::string result = doBusiness(msg);
        // 处理完不直接 send，而是把发送动作交还 IO 线程（见第③点）
        conn->getLoop()->runInLoop([conn, result] { conn->send(result); });
    });
}
```

**② 为什么要将 TcpConnection 对象和消息一起传给线程池：**

1. **知道"结果发给谁"**：业务处理完后必须把结果发回**对应的那个客户端**，所以必须带着这个连接对象；
2. **保证连接存活（生命周期）**：worker 执行期间连接可能被对端关闭，若传裸指针会**野指针**。因此要传 `TcpConnectionPtr`（**shared_ptr**）——**延长连接生命周期到业务执行完毕**，防止"任务还没跑完，连接先析构"；
3. **消息本身**是业务输入，自然要一起传。

**③ 线程池处理完后如何将结果发回客户端：**

**关键：不能直接在 worker 线程调用 `conn->send()`（跨线程操作 socket 不安全）**。正确做法是**把"发送动作"回投到连接所属的 EventLoop（IO 线程）**执行：

```cpp
conn->getLoop()->runInLoop([conn, result] {
    conn->send(result);   // 回到 IO 线程才真正写 socket
});
```

`runInLoop` 的机制（就是 V4 的 eventfd）：
- 若当前线程**就是** EventLoop 线程 → 直接执行；
- 否则 → 把回调 push 进 `_pendingFunctors`（加锁），并**写一次 eventfd** 唤醒 EventLoop；
- EventLoop 被唤醒后执行 `doPendingFunctors`，在 **IO 线程**里调用 `conn->send(result)` → 结果发回客户端。

**为什么要回投（为什么 worker 不能直接 send）**：
`TcpConnection::send()` 会修改输出缓冲、注册/注销 `EPOLLOUT`、操作 epoll 等**非线程安全**的网络层状态；若多个 worker 同时向同一连接 send，或者与 IO 线程的读写并发，会造成数据竞争、缓冲错乱。把 send 统一放回 IO 线程，**所有 socket 操作都只在所属事件循环线程执行**，天然无锁、安全。

**通俗解释**：线程池 worker 是"**后场大厨**"，炒好菜（算出结果）后**不能自己冲出后厨把菜端给客人**（跨线程 send 会撞翻别人/打乱秩序）。正确做法是：**按铃（eventfd）通知前台服务员（EventLoop）**，前台到后场取菜（doPendingFunctors 执行回调），再由前台稳稳端给对应客人（IO 线程 send）。这就是 V4 的核心机制。

---

# 作业8：Reactor V4

## 1. eventfd 原理：read / write 对内核计数器做什么？计数器为 0 时 read 会怎样？

**原题：**
> eventfd原理：eventfd返回的文件描述符如何用于线程间通信？read和write分别对内核计数器做什么操作？当计数器为0时read会怎样？

**答案：**

**eventfd 是什么：**

`eventfd()` 返回一个**文件描述符**，内核为它维护一个 **64 位无符号计数器**（初始值由参数指定）。它专门用于**线程间/进程间"事件通知"**（信号量/唤醒用途），配合 epoll 使用时，是一个 fd 就能实现"跨线程唤醒事件循环"。

```c
int eventfd(unsigned int initval, int flags);   // flags: EFD_NONBLOCK / EFD_CLOEXEC / EFD_SEMAPHORE
```

**read / write 对计数器做什么：**

- **write**：把一个 **8 字节的 `uint64_t`** 值写入 fd，内核把**计数器的值加上该值**（累加）。
  - 例如 `write(fd, &one, 8)` 且 `one=1` → 计数器 +1；
  - 若加完后计数超过 `UINT64_MAX-1`，阻塞模式下 write 阻塞，非阻塞返回 EAGAIN。
- **read**：把计数器的当前值读出来：
  - **普通模式**：若计数器 > 0，读出**当前值并清零**（一次性取走全部），返回读到的 8 字节值；
  - **EFD_SEMAPHORE 模式**：每次 read 只**减 1**（信号量语义），返回 1。

**当计数器为 0 时 read 会怎样：**
- **阻塞模式**：`read` 会**阻塞**，直到有人 write 使计数器 > 0；
- **非阻塞模式（EFD_NONBLOCK）**：`read` 立即返回 **-1，errno = EAGAIN**（/ EWOULDBLOCK）。

**如何用于线程间通信（一次通知流程）：**

```
线程B（EventLoop线程）: 把 eventfd 加入 epoll 监听可读事件，阻塞在 epoll_wait
线程A（任意线程）    : 写 eventfd（write 8字节，计数器+1）
   → 内核把 eventfd 设为可读，唤醒线程B 的 epoll_wait
线程B              : 处理可读事件 → read 清零计数器 → 执行要通知的任务
```

**通俗解释**：eventfd 就是一块"**计数器黑板 + 门铃**"。`write` 是在黑板上加一横（计数+1）；`read` 是把黑板上的数抄走并清零；黑板上没数时，你伸手去抄（read）——要么在门口等到有人来写（阻塞），要么抄个空手而归（EAGAIN）。epoll 把这块黑板也纳入监听，一旦有人写了字，就"叮铃"一声把睡觉的线程叫醒。

---

## 2. 代码封装：写出 Eventfd 类的核心成员（fd、read、write 函数），说明如何实现一次通知

**原题：**
> 代码封装：写出Eventfd类的核心成员（文件描述符、read、write函数），并说明如何在两个线程间实现一次通知。

**答案：**

**Eventfd 类核心封装（muduo 风格）：**

```cpp
#include <sys/eventfd.h>
#include <unistd.h>
#include <functional>

class Eventfd {
public:
    using EventCallback = std::function<void()>;

    explicit Eventfd(EventCallback cb)
        : fd_(::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC)),   // 非阻塞 + close-on-exec
          callback_(std::move(cb)) {
        if (fd_ < 0) { /* 错误处理 */ }
    }

    ~Eventfd() { ::close(fd_); }

    // 可读事件处理：读出计数 → 执行用户回调（一般用于"执行待办任务"）
    void handleRead() {
        uint64_t value = 0;
        ssize_t n = ::read(fd_, &value, sizeof(value));   // 清零计数器
        if (n != sizeof(value)) { /* 出错处理 */ }
        if (callback_) callback_();                       // 执行被唤醒后该做的事
    }

    // 唤醒：写一个 8 字节值，计数器 +1
    void wakeup() {
        uint64_t one = 1;
        ssize_t n = ::write(fd_, &one, sizeof(one));
        (void)n;
    }

    int fd() const { return fd_; }   // 供 epoll_ctl 注册监听

private:
    int fd_;
    EventCallback callback_;
};
```

**两个线程间实现一次通知：**

```
线程A（要通知别人的人）          线程B（EventLoop / 等待者）
─────────────────               ─────────────────────────
                                 1. Eventfd ef(cb);            // 创建
                                 2. epoll_ctl(ADD, ef.fd(), EPOLLIN)
                                 3. epoll_wait(...)            // 阻塞等待
4. ef.wakeup();  ── write(+1) ──▶ 内核置 eventfd 可读
                                 5. epoll_wait 返回
                                 6. ef.handleRead()
                                    - read 清零计数器
                                    - 执行 callback_()（如 doPendingFunctors）
```

也就是：**`write` 负责"叫人"，`read` 负责"接应"，回调负责"办事"**。

**通俗解释**：Eventfd 类就是"**门铃 + 黑板**"的封装：`wakeup()` 是"按门铃"（write，计数+1），`handleRead()` 是"开门并看黑板记了什么事"（read 清零 + 执行 callback）。一个线程想叫醒另一个线程，只需要"按一下门铃"，被叫醒的线程就知道"该干活了"。

---

## 3. Reactor V4：线程池处理完后如何通过 eventfd 通知 EventLoop 取回结果？为什么不能直接让线程池调用 TcpConnection::send？

**原题：**
> Reactor V4通信机制：线程池处理完业务后，如何通过eventfd通知EventLoop取回结果？为什么不能直接让线程池调用TcpConnection::send？

**答案：**

**线程池处理完 → eventfd 通知 EventLoop 的完整机制：**

EventLoop 内部维护两个东西：**`_pendingFunctors`（待办任务队列，加锁保护）** 和 **`_wakeupFd`（一个 eventfd，已注册进 epoll）**。核心入口是 `runInLoop` / `queueInLoop`：

```cpp
void EventLoop::runInLoop(Functor cb) {
    if (isInLoopThread()) {
        cb();                       // ① 本来就是 IO 线程 → 直接执行
    } else {
        queueInLoop(std::move(cb)); // ② 别的线程 → 入队 + 唤醒
    }
}

void EventLoop::queueInLoop(Functor cb) {
    {
        std::lock_guard<std::mutex> lock(mutex_);
        pendingFunctors_.push_back(std::move(cb));
    }
    wakeup();                        // ③ 写 eventfd（计数器+1）唤醒阻塞中的 epoll_wait
}
```

线程池 worker 处理完业务后的动作：

```cpp
// worker 线程中（业务处理完毕）
conn->getLoop()->runInLoop([conn, result] {
    conn->send(result);              // 最终在 IO 线程执行 send
});
```

**完整链路：**
```
worker 线程                    EventLoop 线程
───────                        ─────────────
runInLoop(发送回调)
  → queueInLoop: 加锁push回调到_pendingFunctors
  → wakeup(): write eventfd(+1)
                                epoll_wait 被唤醒（eventfd 可读）
                                → handleRead(): read 清零
                                → doPendingFunctors():
                                   { lock; tmp.swap(_pendingFunctors); }  // 锁外执行
                                   for each cb: cb()  → conn->send(result) 发回客户端
```

**为什么不能直接让线程池调用 TcpConnection::send：**

1. **线程安全 / 数据竞争**：`TcpConnection::send()` 会修改**输出缓冲（OutputBuffer）**、注册/注销 **EPOLLOUT**、调用 epoll 接口等——这些状态**不是线程安全的**。若 worker 直接 send，会与 EventLoop 线程同时操作同一连接的缓冲/epoll，产生数据竞争、缓冲错乱甚至崩溃。
2. **epoll 注册必须在其所属事件循环线程做**：修改 epoll 事件集如果从多个线程并发做，内核对象状态会乱；muduo 的约定是"**每个 fd 的 socket 操作只能发生在它所属的 EventLoop 线程**"，保证天然无锁。
3. **顺序保证**：统一回到 IO 线程按队列顺序执行发送，才能保证同一个连接的**发送顺序**（字节流协议对顺序敏感）。

所以 V4 的答案就是：**worker 不直接碰 socket，只把"发送动作"作为一个待办回调，通过 `runInLoop`（加锁入队 + eventfd 唤醒）交还 IO 线程执行**。

**通俗解释**：worker 是"**后厨炒菜**"，`send` 是"**端菜上桌**"。后厨绝不自己端着菜冲出厨房（跨线程碰 socket 会撞翻一片），而是**按一下服务铃（eventfd）**，前台（EventLoop）听到铃响后，把"上菜"这件事在自己的节奏里一件件做完（doPendingFunctors 里调 send）。这样**所有碰桌子的动作（socket 操作）都只在前台一个人做**，不会打架。

---

## 4. 画出 Reactor V4 的完整流程图（main 线程、EventLoop 线程、线程池工作线程、eventfd 通知路径）

**原题：**
> 流程图：画出Reactor V4的完整流程图，包含main线程、EventLoop线程、线程池工作线程、eventfd通知路径。

**答案：**

```
┌────────────────────────────────────────────────────────────────────────┐
│ main 线程（主线程）                                                      │
│  - 创建 EventLoop（作为 IO 线程运行的宿主）                              │
│  - 创建 ThreadPool（N 个工作线程，每线程循环：取任务 → 执行业务）         │
│  - 创建 Acceptor + 注册到 EventLoop（监听 listenfd）                    │
│  - eventLoop.loop()（把控制权交给 EventLoop，或 EventLoop 跑在独立线程） │
└───────────────┬────────────────────────────────────────────────────────┘
                │ 启动
                ▼
┌────────────────────────────────────────────────────────────────────────┐
│ EventLoop 线程（IO 线程）                                               │
│  loop():                                                               │
│   epoll_wait（监听 listenfd / connfd* / wakeupfd(eventfd)）            │
│       │                                                                │
│       ├─ listenfd 可读 ──▶ Acceptor::accept() → 新 connfd             │
│       │                    → 创建 TcpConnection(shared_ptr)            │
│       │                    → 注册 connfd 到 epoll + 存 fd→conn 映射     │
│       │                                                                │
│       ├─ connfd 可读 ──▶ TcpConnection::handleMessage()               │
│       │                   → read 数据                                  │
│       │                   → 把 (TcpConnectionPtr + 消息) 投递到线程池   │
│       │                     [仅投递，不阻塞，立即返回]                  │
│       │                                                                │
│       └─ wakeupfd(eventfd) 可读 ──▶ handleRead()（read 清零）          │
│                                      → doPendingFunctors()            │
│                                        （锁外执行发送回调 → send 发回） │
│                                                                        │
│   EventLoop 同时维护：_pendingFunctors(加锁) + _wakeupFd(eventfd)      │
└───────┬───────────────────────────────────────▲────────────────────────┘
        │ ① 投递业务任务（addTask）              │ ③ eventfd 唤醒（wakeup: write+1）
        ▼                                       │
┌───────────────────────────────────────────────┴────────────────────────┐
│ ThreadPool 工作线程（N 个）                                            │
│  while(!退出):                                                         │
│     task = TaskQueue.pop()      （空则阻塞在条件变量）                  │
│     task()  → 执行业务逻辑（加解密/数据库/计算）                        │
│     处理完：conn->getLoop()->runInLoop([=]{ conn->send(result); })     │
│             └─ 若不在 IO 线程：加锁入队 _pendingFunctors               │
│                              + eventfd write（②唤醒路径）              │
└────────────────────────────────────────────────────────────────────────┘

eventfd 通知路径（②→③）：worker 线程 ──write(eventfd, +1)──▶ 唤醒 EventLoop
                                                             的 epoll_wait
```

**关键链路小结：**
```
客户端 ──▶ connfd 可读 ──▶ onMessage 投递 ──▶ 线程池 worker 执行业务
            ▲                                      │
            │             conn->send(result)        │ 处理完
            │                 （IO线程执行）         ▼
            └──── doPendingFunctors ◀── eventfd 唤醒 ── runInLoop 回投
```

**通俗解释**：这张图讲的是"**前台（EventLoop）只管接单喊号，后厨（线程池）只管炒菜，出锅后按铃叫前台端菜**"的完整流水线。main 是"老板"负责搭好这家店；eventfd 就是前后台之间那根**传话铃**——凡是跨线程的"通知"，一律走它。

---

## 5. doPendingFunctors 为什么用 `vector tmp; { lock_guard; tmp.swap(_pendingFunctors); }` 这种写法？直接加锁遍历会怎样？

**原题：**
> 在EventLoop::doPendingFunctors执行待办任务时，源码中通常使用vector tmp; { lock_guard lg(_mutex); tmp.swap(_pendingFunctors); }这种swap临时变量的写法。为什么要这么写？如果直接加锁遍历_pendingFunctors执行回调会导致什么严重的锁竞争或死锁问题？

**答案：**

**muduo 的标准写法（交换出队列 + 锁外执行）：**

```cpp
void EventLoop::doPendingFunctors() {
    std::vector<Functor> functors;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        functors.swap(pendingFunctors_);   // 把待办任务"搬"到本地，队列清空
    }                                      // 锁在这里就释放了
    for (auto& f : functors) {
        f();                               // 在锁外执行回调
    }
}
```

**为什么这样写（三个理由）：**

1. **缩短临界区 / 减少锁竞争**：只把"取走任务"这一小步放在锁内，执行回调（可能很耗时）放在锁外。否则每个要投递任务的线程（worker）都要等你慢慢执行完一堆回调才能拿到锁入队 → 投递线程被长时间阻塞，锁竞争严重。

2. **避免死锁（最关键）**：执行回调期间**不持有 `_mutex`**。为什么重要？因为回调内部很可能再次调用 `runInLoop`/`queueInLoop` 去**投递新任务**，而 `queueInLoop` 需要 `lock(_mutex)`。如果此时你还握着 `_mutex`，那么**"自己投递任务 → 自己等自己的锁"→ 直接死锁**（`std::mutex` 非递归）。

3. **保证执行顺序与一致性**：swap 把当前这一批任务整体取出，即使回调中又有新任务投进来，也只进入下一轮，不会把本批队列结构弄乱。

**如果直接加锁遍历 `_pendingFunctors` 执行回调，会有什么严重问题：**

```
死锁场景：
  doPendingFunctors() {
      std::lock_guard lock(mutex_);       // 持有锁
      for (auto& f : pendingFunctors_) {
          f();                            // 回调内部若调 queueInLoop() 投递新任务
          //   → queueInLoop() 里 lock(mutex_)  → 同一个非递归 mutex
          //   → 当前线程自己等自己已经持有的锁 → 死锁（永久阻塞）
      }
  }

锁竞争场景（即使回调不投递）：
  持锁执行耗时回调 → 所有其他想投递任务的线程（线程池 worker）
  全部阻塞在 lock(mutex_) 上排队 → 投递严重变慢，IO 吞吐崩塌
```

**通俗解释**：这就是"**只把任务从仓库搬到走廊上，再到走廊外面慢慢干**"——搬出来的一瞬间才需要锁（因为要动仓库账本），干活（执行回调）在走廊（锁外）干，不影响别人进出仓库。若非要"锁着仓库门在里面干活"，别人想放新货（投递任务）就得一直等；更糟的是干活的人自己还想往里放货（回调内再投递）——**自己锁着门，又想让别人把门打开放货进来，直接卡死**。

---

## 6. 对端已关闭（RST），本端继续 write/send 会收到什么信号？默认行为？网络库应如何忽略？

**原题：**
> 当对端已经关闭连接（收到RST），本端如果继续调用write/send向其发送数据，操作系统会向本进程发送什么信号（SIGPIPE）？默认行为是什么（进程直接崩溃退出）？网络库初始化时应该如何忽略该信号？

**答案：**

**什么信号：SIGPIPE（管道破裂信号）。**

当对端已经关闭（本端收到过 RST，即连接已重置），本端继续 `write()`/`send()` 到该 socket 时，内核会向本进程发送 **`SIGPIPE`** 信号（"你在向一个已经破裂/关闭的管道写数据"）。 [cite:b487e862-3]

**默认行为：直接终止进程（崩溃退出）。**

`SIGPIPE` 的**默认动作是终止（terminate）进程**——这是导致服务器"莫名其妙挂掉"的经典原因之一：对端主动断开后，服务器线程稍后向它发送数据 → 进程被 SIGPIPE 杀死，整个服务崩溃，而不是仅仅这次发送失败。

**网络库初始化时如何忽略该信号：**

在程序入口（网络库 init）用 `signal()` 或 `sigaction()` 把 SIGPIPE 设为 `SIG_IGN`（忽略）：

```cpp
#include <csignal>
// 方案1：最简单
::signal(SIGPIPE, SIG_IGN);

// 方案2：更严谨（sigaction，跨平台更可靠）
struct sigaction sa;
sa.sa_handler = SIG_IGN;          // 忽略
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
::sigaction(SIGPIPE, &sa, nullptr);
```

**忽略之后的行为：**
- 进程**不再被杀**；
- 再次 `send` 时 `write` 返回 **-1，errno = EPIPE**；
- 应用层据此把该连接标记为"已断开"，走正常关闭流程（回调 onClose、清理 fd）。

**补充（两条并发方案）：**
1. **`send` 时加 `MSG_NOSIGNAL` 标志**（Linux 特有），单次发送不触发 SIGPIPE：
   ```cpp
   ::send(fd, data, len, MSG_NOSIGNAL);
   ```
2. 只对**当前 socket** 生效的 `SO_NOSIGPIPE`（macOS/BSD）——Linux 下用 MSG_NOSIGNAL。

**通俗解释**：SIGPIPE 就像"**你对着一根已经断掉的电话线喊话**"——操作系统暴怒，默认直接把整个进程枪毙（不管你这个服务器有多忙，就因为你跟一个已经断线的客户端说话）。所以网络库初始化时第一件事就是"**提前跟系统打招呼：别为这事杀我**"（SIG_IGN），这样系统顶多让这次 send 失败（EPIPE），由你从容地收拾连接。

---

# 作业9：Reactor V5综合

## 1. V5 封装：EchoServer 如何组合 ThreadPool 和 TcpServer？start() 中注册回调为什么用 std::bind？占位符 _1 代表什么？

**原题：**
> V5封装：EchoServer类如何将ThreadPool和TcpServer组合？start()中注册回调时为什么使用std::bind？占位符_1代表什么？

**答案：**

**EchoServer 如何组合 ThreadPool 和 TcpServer：**

EchoServer 内部**同时持有** ThreadPool 和 TcpServer（**组合关系 has-a**，且负责二者生命周期），在 `start()` 里按顺序启动：

```cpp
class EchoServer {
public:
    EchoServer(const InetAddress& addr, size_t threadNum)
        : _tcpServer(addr), _threadPool(threadNum)   // 组合成员
    {}

    void start() {
        // ① 注册业务回调（用 bind 把成员函数绑定到 this）
        _tcpServer.setConnectionCallback(std::bind(&EchoServer::onConnection, this, _1));
        _tcpServer.setMessageCallback(
            std::bind(&EchoServer::onMessage, this, _1, _2));
        _tcpServer.setCloseCallback(std::bind(&EchoServer::onClose, this, _1));

        // ② 先启动线程池（业务处理者）
        _threadPool.start();

        // ③ 再启动 TcpServer（事件循环/IO）
        _tcpServer.start();
    }

private:
    // 回调：把业务交给线程池，结果通过 runInLoop 回发
    void onMessage(const TcpConnectionPtr& conn, std::string& msg) {
        _threadPool.addTask([conn, msg]() mutable {
            std::string result = msg;            // 示例：原样回显
            conn->getLoop()->runInLoop([conn, result] {
                conn->send(result);
            });
        });
    }

    void onConnection(const TcpConnectionPtr& conn) { /* 连接建立/断开 */ }
    void onClose(const TcpConnectionPtr& conn)     { /* 连接关闭 */ }

    TcpServer   _tcpServer;    // 负责 IO 事件循环
    ThreadPool  _threadPool;   // 负责业务处理
};
```

**为什么注册回调时使用 std::bind：**

1. **成员函数不能当普通回调**：`onMessage` 是 EchoServer 的**成员函数**，它隐含 `this` 参数，不能直接赋给 `std::function<void(const TcpConnectionPtr&, std::string&)>`。`std::bind` 把 `this` 和成员函数"捆绑"成一个可调用对象，签名就符合回调要求了。
2. **C++11 时代的标准做法**（muduo 源码即如此）：用 bind 实现"**对象 + 成员函数 → 普通函数对象**"的转换。

**占位符 `_1` 代表什么：**

`std::placeholders::_1` 表示"**这个位置由回调被调用时的第一个实参填充**"。回调是由框架在事件发生时调用的，框架会传入 `TcpConnectionPtr`（连接对象）：

```cpp
std::bind(&EchoServer::onConnection, this, _1)
//    onConnection 第一个参数 = 调用时框架传入的 TcpConnectionPtr
std::bind(&EchoServer::onMessage, this, _1, _2)
//    _1 = TcpConnectionPtr（哪个连接），_2 = std::string&（消息内容）
```

即：**`_1` 在运行期由框架填充为"发生事件的连接对象"，`_2` 填充为"读到的消息"**。

> 更现代（C++14+）的等价写法：用 **lambda + 捕获 this**，可读性更好：
> ```cpp
> _tcpServer.setMessageCallback([this](const TcpConnectionPtr& c, std::string& m){ onMessage(c, m); });
> ```
> lambda 捕获 this 就是"隐式 bind"，功能相同、更直观。

**通俗解释**：`std::bind` 是"**提前把执行人和参数表签好**"的授权书——`this` 是"谁来做"、`_1`/`_2` 是"到时候由系统填的空位"。框架一有事件（来了消息），就按授权书把"连接对象"填进 `_1`、"消息"填进 `_2`，然后执行。这本质和"回调 = 委托"是一样的：**我告诉你到时候做什么，具体谁来、带什么消息由你（框架）决定**。

---

## 2. 为什么基础 Reactor 适合 IO 密集，Reactor + 线程池适合 CPU 密集？既有 IO 又有复杂计算如何设计？

**原题：**
> IO密集型vs CPU密集型：解释为什么基础Reactor适合IO密集型，而Reactor +线程池适合CPU密集型？如果业务既有大量IO又有复杂计算，该如何设计线程模型？

**答案：**

**为什么基础 Reactor 适合 IO 密集型：**

- IO 密集型的特征是：线程**大量时间在等**（等网络数据、等磁盘），真正占 CPU 的片段很少。
- 基础 Reactor（单/少数 EventLoop）用 **epoll 事件驱动**：IO 等待期间线程**阻塞睡眠、不占 CPU**；事件来了才被唤醒处理一小段。所以**一个 IO 线程就能"同时"支撑成千上万条连接**——等待被复用、CPU 只在有事件时工作。这正好匹配 IO 密集（等待多、计算少），线程数量 = CPU 核数也够用。

**为什么 Reactor + 线程池适合 CPU 密集型：**

- CPU 密集型的特征是：任务**占用大量 CPU 计算**（加解密、图像处理、复杂计算），多核并行才能提速。
- 基础 Reactor 里如果业务在 IO 线程同步执行，一个耗时计算会**阻塞事件循环**（队头阻塞、串行）。把计算投递到**线程池（线程数 ≈ CPU 核数）**后，多个核并行算，IO 线程不被阻塞。所以 **Reactor 管收发（IO）+ 线程池管计算（并行）**正是 CPU 密集型的高并发解。

**既有大量 IO 又有复杂计算，如何设计线程模型：**

采用经典的**半同步/半异步（Half-Sync / Half-Async）多线程模型**：

```
┌────────────────────────────────────────────────────────────┐
│ main / 主 Reactor 线程                                       │
│   只负责 accept 新连接，再分发给下面的 IO 线程（负载均衡）      │
└───────────────┬────────────────────────────────────────────┘
                ▼ 分发
┌────────────────────────────────────────────────────────────┐
│ IO 线程池（N 个 EventLoop，每个跑一个 epoll）                 │
│   负责：连接收发、协议解析、数据读写                          │
│   特征：IO 密集，等待不占 CPU；N 一般 = CPU 核数              │
└───────┬───────────────────────────▲─────────────────────────┘
        │ ① 投递计算任务              │ ③ 结果回投（eventfd）
        ▼                            │
┌────────────────────────────────────────────────────────────┐
│ 业务/计算线程池（M 个 worker）                               │
│   负责：加解密、数据库、复杂计算（CPU 密集）                   │
│   特征：M ≈ CPU 核数，并行计算                               │
└────────────────────────────────────────────────────────────┘
规则：IO 只做 IO；计算只在计算池做；结果一律通过 runInLoop(eventfd)
      回投到"该连接所属的 IO 线程"再发送（保证 socket 操作单线程）
```

要点：
1. **两类任务分离**：把"IO/协议解析"和"CPU 计算"分别放进**不同的线程池**，避免互相拖累；
2. **线程数按类型定**：IO 池 ≈ CPU 核数（等待多，太多反而无益），计算池 ≈ CPU 核数（可含超线程），并且要根据任务性质调；
3. **回发必须回投**：worker 算完不直接 send，`runInLoop` 回投到连接所属 EventLoop（V4/V5 的 eventfd 机制）；
4. **背压保护**：任务队列设上限，避免"计算速度跟不上，任务堆积挤爆内存"；
5. 简单场景也可以：**单 Reactor + 单业务线程池**（用任务队列分开 IO 与计算）就够了，不必一上来就上主从多 Reactor。

**通俗解释**：IO 密集像"**前台接待**"——多数时间在等人，一个前台能同时招呼很多客人；CPU 密集像"**后厨炒菜**"——真的在烧火（占 CPU），要多开几个灶（多线程）才快。两者都有，就"**前台只管接待，后厨只管炒菜**"，中间用传菜铃（eventfd）联系，绝不让后厨直接冲出来端菜（跨线程 send）。

---

## 3. timerfd_create 和 timerfd_settime 的作用；itimerspec 的 it_interval 和 it_value；如何实现每 5 秒一次的周期性定时器

**原题：**
> timerfd封装：timerfd_create和timerfd_settime的作用是什么？itimerspec中的it_interval和it_value分别表示什么？如何实现一个周期性定时器（如每5秒触发一次）？

**答案：**

**timerfd_create / timerfd_settime 的作用：**

- `timerfd_create(clockid, flags)`：创建一个**定时器文件描述符**。它把一个内核定时器"包装成 fd"，从而可以被 **epoll 监听**——定时器到点 → fd 变为**可读** → 事件循环被唤醒 → 执行超时回调。这就是"**用 IO 多路复用的方式做定时器**"，天然融入 Reactor。
  - `clockid`：`CLOCK_REALTIME`（墙上时间，可被改时间影响）/ `CLOCK_MONOTONIC`（单调时钟，不受改时间影响，**推荐用于超时/周期**）；
  - `flags`：`TFD_NONBLOCK` / `TFD_CLOEXEC`。
- `timerfd_settime(fd, flags, new_value, old_value)`：**设置/启动定时器**——设定"首次触发时间 + 周期"，并可取回旧设置。
  - `flags` 为 0 表示 `new_value` 是**相对时间**；`TFD_TIMER_ABSTIME` 表示**绝对时间**。

**itimerspec 的两个字段：**

```c
struct itimerspec {
    struct timespec it_interval;   // 周期：触发一次后，每隔 it_interval 再次触发（0 = 只触发一次）
    struct timespec it_value;      // 首次触发时间（相对 or 绝对，取决于 flags）
};
```

- **`it_value`（首次触发）**：定时器**什么时候第一次到点**。设为非 0，定时器才启动；为 0 则关闭定时器。
- **`it_interval`（周期）**：**到点之后每隔多久再触发一次**。若为 0 → **一次性定时器**（只触发一次）；若非 0 → **周期性定时器**。

**实现每 5 秒一次的周期性定时器：**

```cpp
#include <sys/timerfd.h>
#include <unistd.h>
#include <cstdint>

int createPeriodicTimer5s() {
    // 1. 创建 timerfd（单调时钟 + 非阻塞，方便集成进 epoll）
    int tfd = ::timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC);
    if (tfd < 0) return -1;

    // 2. 配置：首次 5 秒后触发，之后每 5 秒触发一次
    struct itimerspec its{};
    its.it_value.tv_sec  = 5;   // 首次触发：5 秒后
    its.it_value.tv_nsec = 0;
    its.it_interval.tv_sec  = 5; // 周期：每 5 秒一次
    its.it_interval.tv_nsec = 0;

    // 3. 启动定时器（flags=0 表示相对时间）
    if (::timerfd_settime(tfd, 0, &its, nullptr) < 0) {
        ::close(tfd);
        return -1;
    }
    return tfd;
}
```

**在 Reactor 中的用法：**

```cpp
// EventLoop 中：把 timerfd 注册进 epoll 读事件
int tfd = createPeriodicTimer5s();
epoll_ctl(epfd, EPOLL_CTL_ADD, tfd, {EPOLLIN});

// 事件循环中：timerfd 可读 → 到点了
void handleTimerEvent() {
    uint64_t expirations = 0;
    ::read(tfd, &expirations, sizeof(expirations));   // 读出已触发的次数
    // 执行周期任务：如"检查所有连接是否空闲超时，超时则断开"
    checkIdleTimeout();
}
```

> 补充：`read` 读出的 8 字节是"自上次读取以来**定时器触发的次数**"（例如睡过头没处理，可能一次读出多次）；必须在回调里把它**读掉清零**，否则 fd 一直可读、epoll 持续通知（LT 模式）。

**通俗解释**：timerfd 就是"**把一个闹钟做成了文件描述符**"——`it_value` 是"闹钟第一次什么时候响"（第一次 5 秒后），`it_interval` 是"响完一次隔多久再响"（每 5 秒）。闹钟一响，这个 fd 就变成"有信"（可读），epoll 立刻知道，于是事件循环在**自己的轮子里**顺手把周期任务办了——不用额外起定时器线程，统一走 epoll 这一套，简洁又线程安全。

---

## 4. 综合设计：高并发 HTTP 服务器（长连接、静态文件下载、动态计算如斐波那契），如何设计 Reactor 架构？

**原题：**
> 综合设计：假设你要实现一个高并发HTTP服务器，支持长连接、静态文件下载和动态计算（如斐波那契）。请结合所学知识，设计你的Reactor架构（版本选择、线程模型、如何集成timerfd做超时断开、如何用eventfd做异步任务通知）。

**答案：**

**① 版本选择：Reactor V5（综合版）。**

V5 = EventLoop（epoll 事件循环）+ Acceptor + TcpConnection（含 InputBuffer/OutputBuffer + 三个半事件回调）+ ThreadPool（业务计算）+ **eventfd**（跨线程任务通知）+ **timerfd**（定时器）。这套正好覆盖 HTTP 服务器的全部需求。

**② 线程模型：主从 Reactor + 业务线程池（半同步/半异步）**

```
┌──────────────────────────────────────────────────────────────────┐
│ main 线程：主 Reactor                                             │
│   只做：创建事件循环、Acceptor、线程池；loop() 里 accept 新连接     │
│   然后按负载（round-robin）把 connfd 分发给某个子 EventLoop        │
└───────────────┬──────────────────────────────────────────────────┘
                ▼ 分发连接
┌──────────────────────────────────────────────────────────────────┐
│ 子 EventLoop 池（N 个 IO 线程，每个一个 epoll）                    │
│   每个连接固定归属某一个 EventLoop（连接级绑定）                    │
│   职责：HTTP 解析、静态文件读取发送、长连接管理、空闲超时检查        │
│   所有 socket 操作只在本 EventLoop 线程做（天然无锁）               │
└───────┬──────────────────────────────▲────────────────────────────┘
        │ 动态计算任务投递              │ 结果回投（eventfd）
        ▼                               │
┌──────────────────────────────────────────────────────────────────┐
│ 业务线程池（M 个 worker）                                         │
│   职责：斐波那契等 CPU 密集计算（HTTP 解析/文件 IO 不做在这里）      │
│   结果：conn->getLoop()->runInLoop([=]{ conn->send(res); })       │
└──────────────────────────────────────────────────────────────────┘
```

**③ 各类请求怎么走：**

- **长连接（Keep-Alive）**：连接建立后不关闭，HTTP 解析完一个请求继续等在 epoll 上；用 **InputBuffer + 包头定长协议**（`Content-Length`/`Transfer-Encoding` 是 HTTP 自带的长度字段）切分请求，解决粘包/拆包；响应后根据请求头决定是否关闭连接。
- **静态文件下载（IO 密集）**：在**连接所属的 EventLoop 线程**里做——把文件分块读入 OutputBuffer 发送；大文件优先用 **`sendfile()`/`mmap`（零拷贝）**，配合 `EPOLLOUT`（可写事件）+ OutputBuffer 分块发，避免阻塞和一次性占用大内存。
- **动态计算（斐波那契，CPU 密集）**：IO 线程解析出请求后，**投递到业务线程池**并行计算；worker 算完**不直接 send**，而是 `runInLoop`（eventfd）把"发送动作"回投到该连接所属的 EventLoop，再由 IO 线程 `send` 发回。这样"计算不堵 IO，IO 不乱算"。

**④ 如何用 timerfd 做超时断开（长连接空闲保护）：**

每个 EventLoop 维护**一个周期 timerfd**（如每 5 秒触发一次），事件循环统一检查：
- 每个 TcpConnection 记录 `lastActive_`（最后一次收到数据的时间，在 onMessage 里更新时间戳）；
- timerfd 到点 → `read` 清计数 → 遍历该 EventLoop 上的连接，若 `now - lastActive_ > 超时阈值`（如 60 秒）→ 判定空闲超时 → 主动 `close` 该连接（发送 FIN + 清理映射）。
- 优点：**用"一个 timerfd + 时间戳扫描"替代"每连接一个定时器"**，O(连接数) 扫描、实现简单，且天然在 IO 线程做（无跨线程问题）。

```
timerfd(每5s可读) ──▶ 检查所有连接的 lastActive_
                     └─ 空闲超时 → 断开连接（发 FIN、清理 fd、删映射）
```

**⑤ 如何用 eventfd 做异步任务通知：**

- 每个 EventLoop 持有一个 eventfd（`_wakeupFd`），注册进自己的 epoll；
- **跨线程投递任务一律走 `runInLoop`**：
  ```
  queueInLoop(cb): { lock; _pendingFunctors.push(cb); }  +  write(eventfd, +1)
  EventLoop: eventfd 可读 → read 清零 → doPendingFunctors()（swap 取出 + 锁外执行）
  ```
- 用途：① 业务线程池把"发送结果"回投给连接所属 EventLoop；② 其他线程（如定时器线程）想让它干任何事，都统一走这一条通道；③ 主线程把新连接分发给子 EventLoop 也走它（子 EventLoop 正在 epoll_wait 睡觉，必须用 eventfd 叫醒）。

**⑥ 整体数据流（一次"GET /fib?n=30"请求）：**

```
客户端 ──▶ 子EventLoop(epoll) ──▶ TcpConnection::handleMessage
         ──▶ InputBuffer 拼出完整HTTP请求（Content-Length 定长）
         ──▶ 解析: /fib?n=30 → 动态请求
         ──▶ 投递到业务线程池: fib(30)
              │
              ▼ 计算完
         runInLoop(发送回调) ── eventfd 唤醒 ──▶ doPendingFunctors
                                                 └─ conn->send(HTTP响应) → 客户端
```

**⑦ 工程补充要点：**
- 每个子 EventLoop 绑定一个线程（one loop per thread），连接数多可水平扩展；
- HTTP 解析器防"超长请求行/超大 body"（限长，防 DoS）；
- 静态文件加缓存（LRU 内存缓存 + ETag/Last-Modified 条件请求）；
- 连接上限/并发数限制、Socket 选项（SO_REUSEADDR、TCP_NODELAY）等。

**通俗解释**：这台 HTTP 服务器就是一家**分工明确的餐厅**：前台（EventLoop）接待并记录每个客人（连接）的最后点菜时间（lastActive），超过多久不点菜就请走（timerfd 超时断开）；后厨（业务线程池）专门算复杂的菜（斐波那契）；静态文件这种"现成的菜"（IO）前台自己端；后厨出锅后按铃（eventfd）叫前台端给对应客人（回投 send）。整套系统靠"一个 epoll + 一根铃（eventfd）+ 一个钟（timerfd）+ 一队后厨（线程池）"把高并发、长连接、下载、计算全部纳入统一模型。

---

## 附：对本套作业的整体小结（帮你串知识点）

- **作业1（设计与原则）**：类关系决定"耦合高低"，SOLID 是"开闭=目标、里氏=基础、依赖倒置=手段"，多态替代分支是落实它们的万能刀。
- **作业2（创建型 + 行为型）**：简单工厂→工厂方法→抽象工厂 是一路"向开闭原则靠近"的演化；观察者用 weak_ptr 解决生命周期。
- **作业3（多线程基础）**：锁（mutex/lock_guard/unique_lock/scoped_lock）、条件变量（while + 谓词 wait）、原子/CAS，是并发的地基。
- **作业4（线程池）**：优雅退出 = 置标志 + notify_all + join + pop 判断标志；任务返回值用 packaged_task/future。
- **作业5（网络基础）**：分层、TCP 握手挥手状态机、字节序、socket 五件套。
- **作业6（IO 多路复用）**：select/poll/epoll 是"轮询 vs 回调"的分水岭；LT/ET、缓冲区 + 定长协议解决粘包拆包。
- **作业7-9（Reactor V3→V5）**：一条"单线程事件驱动 → 业务进线程池 → eventfd 跨线程回投 → timerfd 定时器"的演进主线——**核心始终是"事件循环 + 回调"**，V3-V5 都是在解决"业务怎么不堵 IO、结果怎么安全回发"两个问题。

