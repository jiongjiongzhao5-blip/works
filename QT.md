# 作业1

## 题目1

> **原题**：现阶段不实现计算功能，只实现显示表达式和编辑表达式的功能。可以使用QString 的 append 方法追加一个字符。
> 界面参考图：[https://raw.githubusercontent.com/liaozhengsong/cpp_picture/main/20240530180746.png](https://raw.githubusercontent.com/liaozhengsong/cpp_picture/main/20240530180746.png)

### 答案

参考图是一个经典的计算器界面：上方一个显示表达式的地方，下方 4×4 的按钮阵列（数字、四则运算符、小数点、清空、等号等）[cite:2ad20ac1-1]。本阶段**不做运算**，只做两件事：

1. 用一个 `QString m_expression` 保存当前表达式；
2. 点击数字/运算符按钮时，用 `QString::append()`（等价于 `operator+=`）把该字符追加到表达式尾部，并刷新显示；`C` 清空表达式。

下面给一个**单文件、可直接编译运行**的示例（Qt6）：

```cpp
// main.cpp
#include <QApplication>
#include <QWidget>
#include <QGridLayout>
#include <QLineEdit>
#include <QPushButton>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QWidget w;
    w.setWindowTitle(QStringLiteral("计算器"));

    auto *layout   = new QGridLayout(&w);
    auto *display  = new QLineEdit;
    display->setReadOnly(true);                 // 只显示，不允许手动编辑
    display->setAlignment(Qt::AlignRight);      // 右对齐，像真实计算器
    display->setMinimumHeight(60);
    display->setFont(QFont(QStringLiteral("Consolas"), 16));
    layout->addWidget(display, 0, 0, 1, 4);     // 显示框占整行

    // 4x4 按钮布局
    const QStringList keys = {
        "7", "8", "9", "/",
        "4", "5", "6", "*",
        "1", "2", "3", "-",
        "C", "0", ".", "="
    };

    QString expr;   // 表达式（QString 隐式共享，按值用即可）
    for (int i = 0; i < keys.size(); ++i) {
        auto *btn = new QPushButton(keys[i]);
        layout->addWidget(btn, i / 4 + 1, i % 4);
        connect(btn, &QPushButton::clicked, &w, [&expr, display, btn]() {
            const QString t = btn->text();
            if (t == QStringLiteral("C")) {
                expr.clear();                       // 清空
            } else if (t == QStringLiteral("=")) {
                // 本阶段暂不计算，仅占位（下一阶段在此解析表达式）
            } else {
                expr.append(t);                     // QString::append 追加
            }
            display->setText(expr);                 // 刷新显示
        });
    }

    w.resize(320, 360);
    w.show();
    return app.exec();
}
```

**要点说明**

- `QString::append()` 是追加字符串/字符的最常用成员函数，等价于 `operator+=`；这里每次点击按钮就把按钮文本 `append` 进表达式。
- 一个按钮的 `clicked` 连接到一个 **lambda**，通过 `btn->text()` 知道按的是谁——这是 Qt5/6 推荐的写法，替代了已经废弃的 `QSignalMapper`（旧教程里常用它给多个按钮传参数，现已不推荐）。
- 显示控件用 `QLineEdit`（只读）或 `QLabel` 都行；`QLineEdit` 自带单行文本显示、可以方便地右对齐。
- 下一步实现计算时，只需要在 `=` 分支里把 `expr` 解析成逆波兰式或用 `QJSEngine`/自己写表达式求值即可，UI 层不用动。

**通俗理解**：这就好比在黑板上写算式——你点一个键，就在字符串末尾多写一个字符，点 `C` 就把黑板擦干净。这一版先不“算出结果”，所以 `=` 只是占位。

---

# 作业2

## 题目1

> **原题**：创建一个界面W，上面有QPushButton对象A,B,C,D,E,F：
> - 将A作为根，BCD是A的孩子，E、F是B的孩子
> - 在E上触发鼠标按下事件，传递给W处理
> - 在所有对象上触发鼠标移动事件，显示当前鼠标位置
> - 进入E和F对象之后，C会移动到另一个位置
> - 只要C移动，D也会移动

### 答案

**知识点**：事件过滤器（`installEventFilter` / `eventFilter`）、事件类型（`QEvent::MouseButtonPress`、`MouseMove`、`Enter`、`Move`）、鼠标跟踪（`setMouseTracking`）。父-子关系通过 `new QPushButton(text, parent)` 建立，子控件显示在父控件区域内。

**核心思路**：W 把自己安装为所有按钮的事件过滤器。这样**所有发给这些按钮的事件都会先经过 `W::eventFilter`**，W 就可以统一处理按下、移动、进入、移动等需求，这正是“把 E 上的按下事件交给 W 处理”的标准做法。

```cpp
// W.h
#pragma once
#include <QWidget>

class QPushButton;
class QLabel;

class W : public QWidget {
    Q_OBJECT
public:
    explicit W(QWidget *parent = nullptr);

protected:
    bool eventFilter(QObject *watched, QEvent *event) override;

private:
    QPushButton *A = nullptr, *B = nullptr, *C = nullptr;
    QPushButton *D = nullptr, *E = nullptr, *F = nullptr;
    QLabel *m_posLabel = nullptr;
};
```

```cpp
// W.cpp
#include "W.h"
#include <QPushButton>
#include <QLabel>
#include <QMouseEvent>
#include <QEvent>
#include <QDebug>
#include <QRandomGenerator>

W::W(QWidget *parent) : QWidget(parent) {
    setWindowTitle(QStringLiteral("事件练习"));
    setFixedSize(700, 450);

    // 层次关系：A 是根（A 自己是 W 的孩子）；B、C、D 是 A 的孩子；E、F 是 B 的孩子
    A = new QPushButton(QStringLiteral("A"), this);
    B = new QPushButton(QStringLiteral("B"), A);
    C = new QPushButton(QStringLiteral("C"), A);
    D = new QPushButton(QStringLiteral("D"), A);
    E = new QPushButton(QStringLiteral("E"), B);
    F = new QPushButton(QStringLiteral("F"), B);

    A->setGeometry(20, 40, 640, 380);
    B->setGeometry(20, 20, 260, 260);
    C->setGeometry(320, 20, 120, 80);
    D->setGeometry(320, 130, 120, 80);
    E->setGeometry(20, 20, 110, 80);
    F->setGeometry(150, 20, 90, 80);

    // 显示鼠标位置的标签
    m_posLabel = new QLabel(QStringLiteral("pos: (-1, -1)"), this);
    m_posLabel->setGeometry(5, 5, 300, 24);

    // 让所有控件（含 W 自己）都跟踪鼠标移动
    for (auto *b : {A, B, C, D, E, F})
        b->setMouseTracking(true);
    setMouseTracking(true);

    // W 作为事件过滤器，安装在所有子按钮上
    for (auto *b : {A, B, C, D, E, F})
        b->installEventFilter(this);
    installEventFilter(this);          // W 也过滤自己的事件
}

bool W::eventFilter(QObject *watched, QEvent *event) {
    // 1) 在 E 上按下鼠标 -> 交给 W 处理
    if (watched == E && event->type() == QEvent::MouseButtonPress) {
        qDebug() << "W 处理了 E 上的鼠标按下事件";
        return true;                   // 返回 true：拦截，不再继续传递
    }

    // 2) 所有对象上移动鼠标 -> 显示当前鼠标位置
    if (event->type() == QEvent::MouseMove) {
        auto *me = static_cast<QMouseEvent *>(event);
        m_posLabel->setText(
            QStringLiteral("对象:%1  局部:(%2,%3)  全局:(%4,%5)")
                .arg(watched->objectName().isEmpty()
                         ? QStringLiteral("W")
                         : qobject_cast<QPushButton *>(watched)->text(),
                     QString::number(me->pos().x()),
                     QString::number(me->pos().y()),
                     QString::number(me->globalPosition().toPoint().x()),
                     QString::number(me->globalPosition().toPoint().y())));
        return false;                  // 不拦截，控件仍可正常响应
    }

    // 3) 进入 E 或 F -> C 移动到随机新位置
    if ((watched == E || watched == F) && event->type() == QEvent::Enter) {
        C->move(QRandomGenerator::global()->bounded(0, A->width()  - C->width()),
                QRandomGenerator::global()->bounded(0, A->height() - C->height()));
        return false;
    }

    // 4) 只要 C 移动（Move 事件），D 也跟着移动（保持相对偏移）
    if (watched == C && event->type() == QEvent::Move) {
        D->move(C->x(), C->y() + 110); // D 在 C 正下方
        return false;
    }

    return QWidget::eventFilter(watched, event);
}
```

**解释**

- **“在 E 上触发鼠标按下，传递给 W 处理”**：由于 W 被安装成 E 的事件过滤器，事件发给 E 之前先经过 `W::eventFilter`，此时 `watched == E`，于是 W 处理并返回 `true` 拦截——这就是“交给 W 处理”。如果你想让事件真正**冒泡**到父对象，可以让 E 的 `mousePressEvent` 调用 `event->ignore()`，Qt 就会把事件继续发给父对象 B、A（可传播的鼠标/键盘事件在被忽略时会向父级传递）[cite:0acb7814-3]。两种方式都能实现“W 处理”，事件过滤器更常用。
- **“所有对象上触发鼠标移动，显示鼠标位置”**：默认情况下控件只在**按下按钮并拖动**时才收到 `MouseMove`；`setMouseTracking(true)` 让控件在“悬停不按”时也持续收到移动事件。
- **“进入 E/F，C 移动到另一个位置”**：`QEvent::Enter` 是鼠标进入控件的边界事件（进入/离开事件不需要鼠标跟踪也一定发送），在过滤器里判断 `watched==E||watched==F` 且类型为 `Enter` 即可。
- **“只要 C 移动，D 也移动”**：`QWidget::move()` 会向该控件发送 `QEvent::Move` 事件；给 C 装过滤器，捕获 `Move` 后让 D 跟随，就实现了“C 一动 D 必动”的联动。

**通俗理解**：事件过滤器就像“前台保安”——所有送进门的快递（事件）都先过保安（W）的手。保安可以在门口决定“这个我自己收了（return true，拦截）”，也可以放行（return false）。而 C 一搬家就会广播“我搬家了”（Move 事件），保安听到后顺手把 D 也搬过去。

---

## 题目2

> **原题**：创建一个程序，存在一个窗口W，W的大小是800x600，存在两个QPushButton子类对象A和B，位置是0，0 和 400，150。使用信号槽机制和事件机制，当在鼠标在A上按住并移动时，获取鼠标位置x和y，并将B的位置改成10x+400, 10y+150

### 答案

**设计**：把“事件机制”和“信号槽机制”分开用：
- **事件机制**：让 A 继承 `QPushButton`，重写 `mousePressEvent` / `mouseMoveEvent`，在“按住并移动”时采集鼠标位置，`emit` 一个自定义信号 `dragged(int x, int y)`（把采集到的数据发出去）；
- **信号槽机制**：在 W 里 `connect` 这个信号，槽函数（lambda）按公式 `B->move(10*x+400, 10*y+150)` 移动 B。

```cpp
#include <QApplication>
#include <QPushButton>
#include <QMouseEvent>
#include <QDebug>

// 事件机制：子类化 QPushButton，捕获“按住并移动”
class MyButton : public QPushButton {
    Q_OBJECT
public:
    using QPushButton::QPushButton;

signals:
    void dragged(int x, int y);              // 自定义信号，携带局部坐标 x、y

protected:
    void mousePressEvent(QMouseEvent *e) override {
        m_pressed = true;
        QPushButton::mousePressEvent(e);
    }
    void mouseReleaseEvent(QMouseEvent *e) override {
        m_pressed = false;
        QPushButton::mouseReleaseEvent(e);
    }
    void mouseMoveEvent(QMouseEvent *e) override {
        if (m_pressed)                        // 按住才处理（拖动）
            emit dragged(e->position().x(), e->position().y());
        QPushButton::mouseMoveEvent(e);
    }

private:
    bool m_pressed = false;
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QWidget w;
    w.setWindowTitle(QStringLiteral("拖动联动"));
    w.resize(800, 600);                       // W 大小 800x600

    auto *A = new MyButton(QStringLiteral("A"), &w);
    auto *B = new QPushButton(QStringLiteral("B"), &w);
    A->setGeometry(0, 0, 200, 150);           // A 位置 (0,0)
    B->setGeometry(400, 150, 200, 150);       // B 初始位置 (400,150)

    // 信号槽机制：A 的 dragged 信号 -> 修改 B 的位置
    QObject::connect(A, &MyButton::dragged, &w, [B](int x, int y) {
        B->move(10 * x + 400, 10 * y + 150);
        qDebug() << "B moved to" << B->pos();
    });

    w.show();
    return app.exec();
}
```

**说明**
- 鼠标在 A 上**按住再移动**时，`mouseMoveEvent` 会持续触发（按下状态下的移动不需要 `setMouseTracking` 也能收到），此时采集 `e->position()`（A 内的局部坐标），通过信号发给接收者。
- 公式 `10x+400, 10y+150` 相当于“B = 初始位置 + 10×(鼠标在 A 内的偏移)”，x=0 时 B 回到 (400,150)。
- 这正是事件（采集输入）与信号槽（业务处理）配合的经典范式：**handler 里收数据、发信号，槽里做逻辑**。

**通俗理解**：A 像一个“传感器”：你一按住它并滑动，它就不断喊“我的局部坐标是 (x,y)”（发信号）。W 听到后按公式算一算，把 B 挪过去——**事件负责“听见/看见”，信号槽负责“通知/执行”**，两者解耦。

---

## 题目3

> **原题**：简述事件的产生、分发和处理流程，中间会产生哪些类型的对象，调用了什么方法。

### 答案

Qt 的事件系统分**产生 → 分发 → 处理**三个阶段，中间产生的核心对象是 **`QEvent` 及其子类**。

**1. 事件的产生（Event generation）**

事件按来源分三类 [cite:0acb7814-3]：
- **自发事件（spontaneous）**：由窗口系统（操作系统）产生，比如鼠标点击、键盘按键、窗口重绘、尺寸变化。事件循环通过 `QAbstractEventDispatcher` 从系统队列取出并转换成对应的 `QEvent` 对象。
- **投递事件（posted）**：由程序通过 `QCoreApplication::postEvent()` 产生，进入 Qt 的事件队列，由事件循环异步处理；Qt 会按类型做**合并/压缩**（例如连续多次 `update()` 会合并成一次重绘）。投递的事件用 `new` 创建，处理完后 Qt 自动 `delete`。
- **发送事件（sent）**：由 `QCoreApplication::sendEvent()` **同步**直接发给目标对象（栈上创建），处理完才返回。

**2. 事件的分发（Event dispatch）**

- 事件循环最终都汇聚到 **`QApplication::notify(QObject *receiver, QEvent *event)`**（Qt 中真正的“分发入口”，`sendEvent` 和事件循环都调用它）。
- 发送给目标对象之前，先执行该对象上安装的**事件过滤器**：安装时调用目标对象 `installEventFilter(filterObj)`，分发时按安装顺序依次调用各过滤器的 **`eventFilter(QObject *watched, QEvent *event)`**；过滤器返回 `true` 表示拦截（不再往下），返回 `false` 继续 [cite:0acb7814-3]。

**3. 事件的处理（Event handling）**

- 过滤器放行后，目标对象的 **`QObject::event(QEvent *)`** 被调用。`QWidget::event()` 的默认实现按 `event->type()` 用 `switch` 把事件分发给**具体的 handler**（比如 `mousePressEvent`、`keyPressEvent`、`paintEvent`、`resizeEvent`、`closeEvent`）。
- 具体 handler 内可调用 `event->accept()`（事件已处理）或 `event->ignore()`（未处理）。默认 `QWidget::event()` 会把 handler 的 accept/ignore 状态转换成 `event()` 的**布尔返回值**（接受→true，忽略→false），用于告诉 `notify()` 是否处理成功 [cite:0acb7814-3]。
- **冒泡传播**：鼠标、键盘、滚轮等**可传播事件**，如果最终没有被接受（被 `ignore()`），`notify()` 会把同一事件再次发给**父对象**，父对象同样走“过滤器 → event() → handler”流程，逐级向上直到被接受或到达顶层窗口 [cite:0acb7814-3]。

**中间产生的对象与调用的方法一览**

| 阶段 | 产生的对象                                                   | 调用的方法                                                   |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 产生 | `QEvent` 子类：`QMouseEvent`、`QKeyEvent`、`QPaintEvent`、`QResizeEvent`、`QCloseEvent`、`QWheelEvent`、`QContextMenuEvent` 等 | `QAbstractEventDispatcher`、`postEvent()`、`sendEvent()`     |
| 分发 | 过滤器对象（继承 `QObject` 者）                              | `installEventFilter()`、`QApplication::notify()`、`eventFilter()` |
| 处理 | 目标 `QObject`/`QWidget`                                     | `QObject::event()`、具体 handler（`mousePressEvent` 等）、`accept()`/`ignore()` |

**通俗理解**：事件像一封“快递”：操作系统或程序产生包裹（`QEvent`）；`notify` 是快递站，负责投递；投递前“门卫”（事件过滤器）可以拆开看一眼——退回（true）或放行（false）；签收人（`event()`）再按包裹类型分给具体同事处理（handler）。如果这个同事说“我不管”（ignore），快递站就转投给领导（父对象）。

---

## 题目4

> **原题**：简述信号和槽机制的优势和劣势

### 答案

**优势**

1. **类型安全（编译期检查）**：Qt5/6 的函数指针式 `connect`，信号与槽的参数类型、个数在编译期校验，写错直接编译报错，不会等到运行期才暴露。
2. **松耦合（解耦）**：发送者（信号）不需要知道接收者是谁、有几个、在哪个线程；接收者也不需要知道谁发的。对象之间只通过“信号名”联系。
3. **一对一、一对多、多对一、多对多**：一个信号可连多个槽、多个信号可连同一个槽，自由度很高。
4. **自动断开、防悬空**：`connect` 建立的连接是 `QObject` 感知的——任一端对象销毁，连接自动解除，不会调用已销毁对象的槽（相比裸函数指针安全得多）。
5. **线程友好**：配合 `Qt::QueuedConnection`，可以安全地把信号跨线程投递给接收者所在线程的事件循环，无需自己加锁。
6. **与事件循环融合**：槽作为事件被调度执行，天然适合 GUI 编程；也支持连接 lambda、函数指针、函数对象。
7. **面向对象**：槽就是普通成员函数（Qt5/6 甚至不要求写在 `slots:` 区），可用 `public slots:` 明确暴露，也可私有化。

**劣势**

1. **运行开销**：信号发射是间接调用（比直接函数调用稍慢；DirectConnection 其实只是查连接表后直接调用，开销很小，但在性能极端敏感处仍应避免高频发射）。
2. **难以调试**：调用链是间接的，断点时不易直观看到“谁调了谁”；`sender()` 信息有限。
3. **发射信号没有返回值**：无法直接从 `emit` 拿到槽的结果；需要返回值时必须改为“信号带输出参数”或函数直接调用。
4. **旧式字符串连接不安全**：`connect(a, SIGNAL(clicked()), b, SLOT(...))` 用字符串在运行期匹配，写错不报编译错、只在运行时警告甚至静默失败；且要求槽必须声明在 `slots:` 区。
5. **构建复杂化**：依赖元对象编译器 `moc`，类定义必须写在头文件并带 `Q_OBJECT`，模板类无法使用信号槽。
6. **滥用会造成性能/可读性问题**：若把每个状态变化都发信号、槽里再做大量工作，逻辑会被拆得很散，读代码需要跳转。

**通俗理解**：信号槽像“广播电台”——电台（信号）只管播报，不关心谁在听，想听的人自己调频（connect）。好处是电台和听众互不认识、很灵活；坏处是“播报”比“当面打电话”（直接调用）多一层间接，出了问题不容易一眼看到是谁在接。

---

## 题目5

> **原题**：简述信号和槽机制的使用流程。如何设计和使用 信号、槽和关联

### 答案

**使用流程（五步）**

1. **类继承 `QObject` 并写 `Q_OBJECT` 宏**（`QWidget`/`QMainWindow` 等已继承 `QObject`），这是信号槽生效的前提（需要 `moc` 处理）。
2. **声明信号**：在 `signals:` 区声明，形如 `void valueChanged(int v);`。信号**只声明、不实现**（由 moc 生成），也不能有返回值。
3. **声明槽**：Qt5/6 中槽就是普通成员函数，写在 `public slots:`/`private slots:` 区只是更规范；lambda 也能当槽。
4. **关联（connect）**：`connect(sender, &Sender::signal, receiver, &Receiver::slot)` 或 `connect(sender, &Sender::signal, context, [=]{...})`。
5. **发射（emit）**：在需要的位置 `emit signalName(args);`，事件循环或 `DirectConnection` 会调用已连接的槽。

**设计原则**

- **信号命名**：用“事件已发生”的语义，如 `clicked`、`textChanged`、`finished`；避免用“doSomething”这种命令式命名。
- **参数匹配**：槽的参数**个数 ≤ 信号参数**，且类型可转换（多余的信号参数被忽略）；不要反过来。
- **一个信号可连多个槽，一个槽可被多个信号连**；用 `disconnect` 可断开。
- **谨慎在构造函数里 `emit`**：此时连接可能还没建立，且对象未完全构造，信号通常丢失。
- **`sender()` 只能在槽函数（非 lambda）里用**，用于反查发送者；lambda 里想拿发送者就捕获发送者指针，或用 `qobject_cast` 配合。
- **用上下文对象做接收者**：`connect(s, &S::sig, this, &This::slot)` 让接收者销毁时连接自动解除，避免野指针。

**示例**

```cpp
class Counter : public QObject {
    Q_OBJECT
public:
    void setValue(int v) {
        if (v == m_value) return;
        m_value = v;
        emit valueChanged(m_value);        // 值变化 -> 发信号
    }
signals:
    void valueChanged(int newValue);      // 只声明，不实现
private:
    int m_value = 0;
};

class Display : public QObject {
    Q_OBJECT
public slots:                              // 槽：普通成员函数
    void show(int v) { qDebug() << "value =" << v; }
};

// 使用
Counter c;
Display d;
QObject::connect(&c, &Counter::valueChanged, &d, &Display::show);
c.setValue(42);                           // 打印 value = 42
```

**通俗理解**：信号是“宣布一件事发生了”，槽是“对这件事的处理”，connect 就是“把这件事和处理挂钩的登记单”。谁都能登记，也能随时撤销。

---

## 题目6

> **原题**：使用元对象系统的注意事项有哪些？

### 答案

（注：作业3第4题与本题重复，此处完整作答，作业3可直接引用。）

1. **必须继承 `QObject`**（直接或间接），且**必须声明 `Q_OBJECT` 宏**，否则没有元对象（没有信号槽、属性、`tr()`、`qobject_cast` 等）。
2. **类定义必须放在头文件里**，由 `moc` 预处理；若硬要写在 `.cpp`，需在文件末尾 `#include "xxx.moc"`（不推荐，易出错）。
3. **`QObject` 不可拷贝、不可赋值**（拷贝构造/赋值被禁用），只能**通过指针**使用；因此不能把 `QObject` 子类对象按值放进容器，要用指针（`QList<MyObj*>`）。
4. **父对象（parent）所有权**：`new MyWidget(parent)` 后，父对象析构会自动 `delete` 子对象，**不要重复 delete**；顶层对象可设 `parent=nullptr` 自行管理。
5. **多重继承时 `QObject` 必须是第一个基类**（moc 要求）。
6. **信号槽依赖元对象**：字符串式 `SIGNAL()/SLOT()` 连接在运行期匹配，编译期不检查；建议用 Qt5/6 的**函数指针式**连接（编译期类型检查）。
7. **模板类不能带 `Q_OBJECT`**（moc 无法处理模板）；如需要，可在模板外部再封装具体类型。
8. **不要用 `Q_OBJECT` 类跨 DLL 边界直接共享元数据**（除非用 Q_DECLARE_METATYPE / 合理导出）。
9. **属性/元类型**：自定义类型要在信号槽间传值需 `Q_DECLARE_METATYPE`（队列连接）、或用 `Q_PROPERTY` 注册属性、`Q_INVOKABLE` 标记可被调用的函数。
10. **线程亲和性**：对象的槽默认在**对象所属线程**的事件循环里执行（`QueuedConnection`）；跨线程要保证 `moveToThread` 使用正确，避免并发访问。
11. **不要在构造函数/析构函数里发射信号**：此时连接可能未建立、对象未完全初始化，信号易丢失；析构中调 `emit` 也可能访问已释放资源。
12. **`Q_OBJECT` 改动后需重新跑 `moc`**（构建工具会自动做；手动改 `.moc` 文件可能导致“重新生成后编译不一致”）。
13. **不要直接 delete 正在被信号槽使用的对象**：优先用 `deleteLater()` 让对象在处理完当前事件后安全销毁。
14. **`qobject_cast` 只对带 `Q_OBJECT`（或 Q_GADGET）的类型有效**，普通类会返回 `nullptr`。

**通俗理解**：元对象系统像是给对象发了一张“身份证+通讯录”（元信息表）。要这张卡就必须满足前提：是 QObject 的后代、在头文件里盖了 `Q_OBJECT` 章、而且不能把它“复制”（只能拿指针）。不满足这些，信号槽、反射、类型转换全都失效。

---

## 题目7

> **原题**：QString和std::string和char数组有什么区别？QString增删改查用什么成员函数？

### 答案

**三者的区别**

| 项目            | `QString`                                                   | `std::string`                                              | `char` 数组/`const char*`     |
| --------------- | ----------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------- |
| 编码            | **UTF-16** 存储 Unicode 字符（`QChar` 16位），面向文本与 UI | 存储**字节序列**（`char`，无编码概念），常用于通用 IO/算法 | C 风格字节数组，`'\0'` 结尾   |
| 是否以 NUL 结尾 | 内部有结尾哨兵，但按“字符数”操作，不按字节                  | 否（有 `size()`）                                          | **是**（必须手动保证 `'\0'`） |
| 生命周期/内存   | 隐式共享（写时复制），值传递安全                            | 自管理内存                                                 | 手动管理，易溢出/悬空         |
| 适用场景        | Qt 界面、Qt API、Unicode 处理                               | 标准库算法、文件/网络原始字节                              | C API、底层、性能敏感         |
| 字符 vs 字节    | `size()` 返回字符数（QChar 数）                             | `size()` 返回字节数                                        | `strlen` 返回字节数           |

**转换（Qt6）**

```cpp
QString  q  = QString::fromStdString(s);      // std::string -> QString（按 UTF-8）
std::string s2 = q.toStdString();             // QString -> std::string（UTF-8）
QString  q2 = QString::fromUtf8(cstr);        // const char* -> QString（UTF-8，推荐）
QByteArray b = q.toUtf8();                    // QString -> UTF-8 字节
const char *p = q.toUtf8().constData();       // 注意：返回的 QByteArray 是临时对象，需立刻使用
```

Qt6 中 `QString` 从 `const char*` 构造默认按 **UTF-8** 解释；跨编码场景请显式使用 `fromUtf8`/`fromLocal8Bit`。

**QString 增删改查成员函数**

- **增**：`append()`（追加）、`prepend()`（前插）、`insert(pos, str)`（指定位置插入）、`operator+=`、`push_back()`。
- **删**：`remove(start, len)`/`remove(str)`（删子串）、`chop(n)`（删尾部 n 个字符）、`truncate(n)`（截断到 n 个字符）、`clear()`（清空）、Qt6 还有 `removeIf()`。
- **改**：`replace(old, new)`（替换，支持正则）、`setNum()`（数字转字符串）、`fill(ch)`、`operator=`。
- **查**：`indexOf()`/`lastIndexOf()`（找子串位置）、`contains()`、`startsWith()`/`endsWith()`、`count()`（计数）、`at(i)`/`operator[]`（取第 i 个字符）、`mid()/left()/right()`（取子串）、`split()`（按分隔符切分）、`section()`（取分段）。

**通俗理解**：`QString` 是“专门给界面和中文用的字符串”，存的是 Unicode 字符，不怕乱码；`std::string` 是“存原始字节的通用字符串”；`char[]` 是最底层的“老古董”，要自己管 `'\0'` 和内存。GUI 里一律用 `QString`，传到底层或网络再转成 `std::string`/`QByteArray`。

---

# 作业3

## 题目1

> **原题**：书写一个函数MyIntersection，可以支持两个相同的QList或者QSet 的取交集的操作，返回新集合。书写一个函数MyOdd，可以将个QMap<int,int>的所有奇数的键或者是值搜集起来，返回新QList。

### 答案

`QList` 和 `QSet` 都支持 `begin()/end()`、`contains()`，但成员不同：`QSet` 有 `intersect()`（**注意：Qt6 中它是“原地修改”并返回自身引用**，不是返回新集合）[cite:0acb7814-2]，`QList` 没有。所以写成两个模板重载最清晰。

```cpp
#include <QList>
#include <QSet>
#include <QMap>

// 两个 QSet 求交集，返回新 QSet
template<typename T>
QSet<T> MyIntersection(const QSet<T> &a, const QSet<T> &b) {
    QSet<T> result = a;     // 先拷贝一份（避免修改 a）
    result.intersect(b);    // Qt6 中 intersect() 原地修改，返回自身引用
    return result;
}

// 两个 QList 求交集，返回新 QList（保持 a 的顺序，去重）
template<typename T>
QList<T> MyIntersection(const QList<T> &a, const QList<T> &b) {
    QList<T> result;
    for (const T &v : a)
        if (b.contains(v) && !result.contains(v))
            result.append(v);
    return result;
}
```

**说明**：Qt6 中 `QSet::intersect()` 的签名是 `QSet<T>& intersect(const QSet<T>&)`（原地修改、返回自身引用），而返回**新集合**的是 `operator&`[cite:0acb7814-2]。上面先拷贝再 `intersect` 就能安全地“不改原集合、返回新集合”；或者直接写 `return a & b;` 也等价。

```cpp
// 收集 QMap<int,int> 中所有【奇数键】
QList<int> MyOddKeys(const QMap<int, int> &m) {
    QList<int> result;
    for (auto it = m.constBegin(); it != m.constEnd(); ++it)
        if (it.key() % 2 != 0)
            result.append(it.key());
    return result;
}

// 收集 QMap<int,int> 中所有【奇数值】
QList<int> MyOddValues(const QMap<int, int> &m) {
    QList<int> result;
    for (auto it = m.constBegin(); it != m.constEnd(); ++it)
        if (it.value() % 2 != 0)
            result.append(it.value());
    return result;
}
```

如果想“一个函数完成键或值”，加一个布尔参数：

```cpp
QList<int> MyOdd(const QMap<int, int> &m, bool keyMode = true) {
    QList<int> result;
    for (auto it = m.constBegin(); it != m.constEnd(); ++it) {
        int v = keyMode ? it.key() : it.value();
        if (v % 2 != 0) result.append(v);
    }
    return result;
}
```

**通俗理解**：交集就是“两边都有的元素”。`QSet` 天然是集合，用现成的 `intersect` 就行（但要先拷贝，别把原集合改坏）；`QList` 没有现成交集，就遍历第一个，凡是在第二个里出现的都收进结果。`MyOdd` 就是遍历 `QMap`，把奇数挑出来——`key()` 是键、`value()` 是值。

---

## 题目2

> **原题**：给出一个路径/usr/include/dir1/dir2:将路径切割并存入一个栈中。实现cd子目录和cd上一级功能，返回路径的字符串。

### 答案

用**栈**保存路径的每一级：栈底是“/”（根），栈顶是当前最深的目录。`cd 子目录` 就 `push` 压栈；`cd ..` 就 `pop` 弹栈（在根目录则不动）。

Qt6 里 `QStack` 仍然存在（继承自 `QList`，提供 `push/pop/top`）[cite:0acb7814-1]，但从现代 C++ 的角度，更推荐**标准库的 `std::stack`**，因为它是纯 STL 组件、更通用。下面两种都给出。

```cpp
#include <stack>
#include <QString>
#include <QStringList>

class PathNavigator {
public:
    // 用路径初始化：切割后压栈（从根开始）
    explicit PathNavigator(const QString &path) {
        for (const QString &part : path.split('/', Qt::SkipEmptyParts))
            m_stack.push(part);
    }

    // cd 进入子目录
    void cd(const QString &sub) {
        if (sub.isEmpty() || sub == "/") return;   // 忽略空和根
        if (sub == "..") { cdParent(); return; }   // 兼容 cd .. 写法
        m_stack.push(sub);
    }

    // cd 上一级
    void cdParent() {
        if (!m_stack.empty())
            m_stack.pop();                          // 栈顶是最深目录，弹出即上一级
    }

    // 返回当前路径字符串
    QString toString() const {
        QStringList parts;
        auto copy = m_stack;                        // 拷贝一份，不破坏原栈
        while (!copy.empty()) {                     // 栈底在队列头部，需反序
            parts.push_front(copy.top());
            copy.pop();
        }
        return "/" + parts.join('/');
    }

private:
    std::stack<QString> m_stack;                    // 注意：std::stack 无迭代器，用 top/pop 遍历
};
```

**要点**：
- `path.split('/', Qt::SkipEmptyParts)` 把 `/usr/include/dir1/dir2` 切成 `["usr","include","dir1","dir2"]`（`Qt::SkipEmptyParts` 是 Qt5.14 起对旧 `QString::SkipEmptyParts` 的改名）。
- 栈的 LIFO 特性正好符合“上一级”语义：当前在 `dir2`，`cd ..` 弹出 `dir2`，路径变成 `/usr/include/dir1`。
- `toString()` 因为要按“根在前”的顺序输出，而栈顶在最深目录，所以用一个临时副本从栈底到栈顶拼出来。

如果老师要求用 Qt 自带的栈，只需把 `std::stack<QString>` 换成 `QStack<QString>`（`<QStack>`），`push/pop/top/isEmpty` 用法一致即可 [cite:0acb7814-1]。

**通俗理解**：路径就是一摞“俄罗斯套娃”。你在最里面的套娃（dir2），想往里走就套一个更大的（push），想退出去就拿掉最外层那个（pop）。栈（LIFO）天生就是干这个的。

---

## 题目3

> **原题**：实现一个井字棋点击空白按钮可以显示X或者O。使用sender和qobject_cast改写。
> 参考图：[https://raw.githubusercontent.com/liaozhengsong/cpp_picture/main/20240601121918.png](https://raw.githubusercontent.com/liaozhengsong/cpp_picture/main/20240601121918.png)

### 答案

参考图显示井字棋棋盘之外，还有 **Clear（清空）** 和 **Undo（撤销）** 两个按钮 [cite:2ad20ac1-2]，所以一并实现。核心技巧是：**9 个格子按钮共用一个槽函数**，在槽里用 `sender()` 拿到发出信号的按钮，再用 `qobject_cast<QPushButton*>` 安全转换，从而知道点的是哪一格。

```cpp
#include <QApplication>
#include <QWidget>
#include <QGridLayout>
#include <QHBoxLayout>
#include <QVBoxLayout>
#include <QPushButton>
#include <QStack>
#include <QFont>

class TicTacToe : public QWidget {
    Q_OBJECT
public:
    explicit TicTacToe(QWidget *parent = nullptr) : QWidget(parent) {
        setWindowTitle(QStringLiteral("井字棋"));

        auto *mainLayout = new QVBoxLayout(this);
        auto *grid       = new QGridLayout;

        // 创建 3x3 个按钮，全部连接到同一个槽 onCellClicked
        for (int r = 0; r < 3; ++r) {
            for (int c = 0; c < 3; ++c) {
                m_buttons[r][c] = new QPushButton(this);
                m_buttons[r][c]->setFixedSize(90, 90);
                m_buttons[r][c]->setFont(QFont(QStringLiteral("微软雅黑"), 28, QFont::Bold));
                grid->addWidget(m_buttons[r][c], r, c);
                connect(m_buttons[r][c], &QPushButton::clicked,
                        this, &TicTacToe::onCellClicked);
            }
        }

        auto *clear = new QPushButton(QStringLiteral("Clear"), this);
        auto *undo  = new QPushButton(QStringLiteral("Undo"),  this);
        connect(clear, &QPushButton::clicked, this, &TicTacToe::onClear);
        connect(undo,  &QPushButton::clicked, this, &TicTacToe::onUndo);

        auto *btnRow = new QHBoxLayout;
        btnRow->addStretch();
        btnRow->addWidget(clear);
        btnRow->addWidget(undo);
        btnRow->addStretch();

        mainLayout->addLayout(grid);
        mainLayout->addLayout(btnRow);
    }

private slots:
    // 关键：用 sender() + qobject_cast 判断被点击的是哪个按钮
    void onCellClicked() {
        auto *btn = qobject_cast<QPushButton *>(sender());
        if (!btn) return;                       // 不是 QPushButton 发出的，忽略
        if (!btn->text().isEmpty()) return;     // 已有 X/O 的格子不可再点

        btn->setText(m_isX ? QStringLiteral("X") : QStringLiteral("O"));
        m_history.push(btn);                    // 记录这一步，供 Undo 使用
        m_isX = !m_isX;                         // 换手
        checkWin();                             // 可选：判胜负
    }

    void onUndo() {
        if (m_history.isEmpty()) return;
        QPushButton *btn = m_history.pop();
        btn->clear();
        m_isX = !m_isX;                         // 换手回去
    }

    void onClear() {
        for (int r = 0; r < 3; ++r)
            for (int c = 0; c < 3; ++c) {
                m_buttons[r][c]->clear();
            }
        m_history.clear();
        m_isX = true;
    }

private:
    void checkWin() { /* 遍历行/列/对角线判断胜负，可弹 QMessageBox */ }

    QPushButton *m_buttons[3][3] = {};
    QStack<QPushButton*> m_history;   // 记录每一步，支持撤销
    bool m_isX = true;
};
```

**为什么用 `sender()` + `qobject_cast`？**
- 9 个按钮共用一个槽，槽里无法静态知道是谁发的信号，所以用 `sender()` 动态取出发信号的对象（`QObject*`）。
- `qobject_cast<QPushButton*>(sender())` 是**类型安全**的下转型：如果 `sender()` 不是 `QPushButton`（或其后代）就返回 `nullptr`，不会像 C 风格强转那样产生未定义行为。这正是元对象系统（`Q_OBJECT`）提供的能力。

**通俗理解**：一个接线员同时接 9 个格子按钮的电话（共用一个槽）。有人打电话过来，接线员先问“你是谁？”（`sender()`），再确认“你确实是个格子按钮吧？”（`qobject_cast`），确认后看格子是否为空，是就写上 X/O 并换手。`Undo` 用栈记住“谁最后下的”，撤销时把栈顶那格清空；`Clear` 全部清空。

---

## 题目4

> **原题**：使用元对象系统的注意事项有哪些？

### 答案

与作业2第6题完全相同，答案见上文《作业2 · 题目6》。核心注意点：必须继承 `QObject` 且声明 `Q_OBJECT`；类放头文件给 `moc` 处理；`QObject` 不可拷贝、只能指针使用；父对象自动管理生命周期；`QObject` 放多重继承第一位；模板类不能用 `Q_OBJECT`；优先用函数指针式 `connect`；自定义类型跨线程传值要注册元类型；不要在构造/析构中 `emit`；销毁用 `deleteLater()` 等。

---

## 题目5

> **原题**：信号和槽机制的目的是什么？书写代码实现一个信号和自定义槽函数。

### 答案

**目的**：实现**对象之间的松耦合通信**。发送者只需发出“某事件发生了”的信号，不关心谁接收；接收者（槽）也无需知道信号来自谁。它让程序逻辑可以按“事件驱动”的方式组织，是 Qt 事件驱动编程的基石，也天然支持一对多、跨线程（QueuedConnection）等场景。

**代码实现**

```cpp
#include <QObject>
#include <QDebug>

class Counter : public QObject {
    Q_OBJECT
public:
    explicit Counter(QObject *parent = nullptr) : QObject(parent) {}

    void setValue(int v) {
        if (v == m_value) return;
        m_value = v;
        emit valueChanged(m_value);     // 值变化时发射信号
    }
signals:
    void valueChanged(int newValue);    // 自定义信号：只声明，不实现
private:
    int m_value = 0;
};

class Display : public QObject {
    Q_OBJECT
public:
    explicit Display(QObject *parent = nullptr) : QObject(parent) {}
public slots:                           // 自定义槽
    void show(int v) {
        qDebug() << "收到信号，值 = " << v;
    }
};

// 使用
Counter c;
Display d;
QObject::connect(&c, &Counter::valueChanged, &d, &Display::show);
c.setValue(10);     // 触发：Display::show(10)
c.setValue(20);     // 触发：Display::show(20)
```

**要点**：信号只能在所属类内部（或其子类）`emit`（外部可用 `QMetaObject::invokeMethod` 间接触发）；槽是普通成员函数，放在 `slots:` 区是为了旧式兼容与可读性，Qt5/6 的 `connect` 并不强制。

---

## 题目6

> **原题**：connect函数有几种重载形式？哪一种更好为什么？

### 答案

Qt5/6 中 `QObject::connect` 的主要重载（针对不同槽形态）：

1. **函数指针式（成员函数）**：`connect(sender, &Sender::signal, receiver, &Receiver::slot)`
2. **带上下文对象的 lambda**：`connect(sender, &Sender::signal, context, [=](...) { ... })` —— context（通常是 `this`）提供自动断开。
3. **无上下文的函数对象/lambda**：`connect(sender, &Sender::signal, functor)` —— 接收者没有 QObject 上下文，**不会自动断开**，需手动管理生命周期。
4. **连接到自由函数/静态函数**：`connect(sender, &Sender::signal, &globalFunc)`
5. **旧式字符串宏**：`connect(sender, SIGNAL(clicked()), receiver, SLOT(mySlot()))` —— 运行期字符串匹配。
6. **混合式**：`connect(sender, SIGNAL(clicked()), receiver, lambda)`。
7. **元方法式**：`connect(sender, &Sender::staticMetaObject, ..., QMetaMethod...)` 等底层重载（较少用）。

**哪一种更好？**
**函数指针式（第1种）最好**，原因：
- **编译期类型检查**：信号/槽的参数类型、个数不匹配会直接编译报错，而旧式字符串在运行期才可能警告或静默失败。
- **拼写/符号安全**：函数指针直接引用成员，不存在“字符串写错”的问题；改名时编译器会提示。
- **性能略优**：省去运行期字符串查表。
- **槽不需要写进 `slots:` 区**：普通成员函数即可作为槽，更灵活。
- 配合 lambda（第2种）还能捕获局部状态，但要**务必提供上下文对象**，保证接收者销毁时连接自动断开（否则可能回调已销毁对象）。

**通俗理解**：旧式 `SIGNAL()/SLOT()` 像“把两个人的名字写在纸条上，运行时核对身份”——写错了要等出事才发现；新式函数指针像“直接加微信好友”——当场就能确认对象正确，错了编译就报错。所以能用函数指针就用函数指针。

---

## 题目7

> **原题**：现在有三个对象A，B，C。需要实现下面效果:发射信号会导致B调用槽函数C调用槽函数。B发射信号会导致C调用槽函数。C发射信号会导致A发射信号

### 答案

需要三个都继承 `QObject` 的类，分别有信号；关键是三种连接方式：
1. **A 的信号 → B 的槽 和 C 的槽**（一个信号连两个槽）；
2. **B 的信号 → C 的槽**；
3. **C 的信号 → A 的另一个信号**（**信号连接信号**，C 发信号会“转发”让 A 也发信号）。

注意：信号是受保护的，外部不能直接 `a.sig()`，所以每个类提供一个公开的触发函数来 `emit`。

```cpp
#include <QObject>
#include <QDebug>

class A : public QObject {
    Q_OBJECT
public:
    using QObject::QObject;
    void triggerA()  { emit sigA();  }     // 供外部触发
    void triggerA2() { emit sigA2(); }
signals:
    void sigA();                            // 被 B、C 的槽接收
    void sigA2();                           // 由 C 的信号触发（信号连信号）
};

class B : public QObject {
    Q_OBJECT
public:
    using QObject::QObject;
    void triggerB() { emit sigB(); }
signals:
    void sigB();
public slots:
    void slotB() { qDebug() << "B::slotB 被调用"; }
};

class C : public QObject {
    Q_OBJECT
public:
    using QObject::QObject;
    void triggerC() { emit sigC(); }
signals:
    void sigC();
public slots:
    void slotC() { qDebug() << "C::slotC 被调用"; }
};

// 组装与连接
A a; B b; C c;

// 1) A 发射信号 -> B 调用槽函数、C 调用槽函数
QObject::connect(&a, &A::sigA, &b, &B::slotB);
QObject::connect(&a, &A::sigA, &c, &C::slotC);

// 2) B 发射信号 -> C 调用槽函数
QObject::connect(&b, &B::sigB, &c, &C::slotC);

// 3) C 发射信号 -> A 发射另一个信号（信号连信号）
QObject::connect(&c, &C::sigC, &a, &A::sigA2);

// 演示
a.triggerA();   // 输出: B::slotB 被调用 / C::slotC 被调用
b.triggerB();   // 输出: C::slotC 被调用
c.triggerC();   // 输出: A::sigA2 被发射（可再连接一个槽验证）
```

**要点**：信号连接信号（第3条）是 `connect` 的合法用法——此时 A 只是“转发”信号，不真正执行任何逻辑；如果需要 A 在转发时干点别的，可以在 A 里写一个槽，在槽里 `emit sigA2()` 并做其他处理。

**通俗理解**：A 是“总指挥”，一吹哨（sigA）B 和 C 同时动手；B 自己也能吹哨（sigB）让 C 动手；C 一吹哨（sigC）就让 A 也吹哨（sigA2）——信号也能当“传话人”一路接力。

---

# 作业4

## 题目1

> **原题**：现在有三个对象ABC，A的父亲是B，B的父亲是C。点击A的内部，要求执行：A的filter、event和event_handler；B的filter、event。

### 答案

层次：`A`（子）→ `B`（父）→ `C`（顶层窗口）。要“A 的 filter、event、handler”和“B 的 filter、event”都执行，设计如下：

- A 和 B 各自把自己安装为自己的事件过滤器（`installEventFilter(this)`），于是各自在 `event()` 之前先跑自己的 `eventFilter()`（这就是各自的 filter）。
- A 的 `mousePressEvent` 里调用 `event->ignore()`，让**可传播的鼠标事件冒泡给父对象 B**；于是事件随后在 B 上经历 B 的 filter → B 的 event。

执行顺序打印为：

```
A::eventFilter       ← A 的 filter
A::event             ← A 的 event()
A::mousePressEvent   ← A 的 event_handler（调用 ignore()）
B::eventFilter       ← B 的 filter（事件冒泡到 B 后，先过 B 的过滤器）
B::event             ← B 的 event()
```

```cpp
#include <QApplication>
#include <QWidget>
#include <QMouseEvent>
#include <QEvent>
#include <QDebug>

class A : public QWidget {
    Q_OBJECT
public:
    explicit A(QWidget *parent = nullptr) : QWidget(parent) {
        installEventFilter(this);          // A 安装自己为事件过滤器
    }
protected:
    bool eventFilter(QObject *watched, QEvent *e) override {
        if (watched == this && e->type() == QEvent::MouseButtonPress)
            qDebug() << "A::eventFilter —— A 的 filter";
        return QWidget::eventFilter(watched, e);   // 返回 false 语义：放行
    }
    bool event(QEvent *e) override {
        if (e->type() == QEvent::MouseButtonPress)
            qDebug() << "A::event —— A 的 event()";
        return QWidget::event(e);          // 默认分发给具体 handler
    }
    void mousePressEvent(QMouseEvent *e) override {
        qDebug() << "A::mousePressEvent —— A 的 event_handler，调用 ignore() 冒泡给 B";
        e->ignore();                       // 未处理 -> 事件向父对象 B 冒泡
    }
};

class B : public QWidget {
    Q_OBJECT
public:
    explicit B(QWidget *parent = nullptr) : QWidget(parent) {
        installEventFilter(this);          // B 安装自己为事件过滤器
    }
protected:
    bool eventFilter(QObject *watched, QEvent *e) override {
        if (watched == this && e->type() == QEvent::MouseButtonPress)
            qDebug() << "B::eventFilter —— B 的 filter";
        return QWidget::eventFilter(watched, e);
    }
    bool event(QEvent *e) override {
        if (e->type() == QEvent::MouseButtonPress)
            qDebug() << "B::event —— B 的 event()";
        return QWidget::event(e);
    }
};

class C : public QWidget {                 // C：最外层窗口
    Q_OBJECT
public:
    explicit C(QWidget *parent = nullptr) : QWidget(parent) {}
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    C c;
    c.resize(400, 400);
    B b(&c);            // B 的父亲是 C
    b.setGeometry(50, 50, 300, 300);
    A a(&b);            // A 的父亲是 B
    a.setGeometry(50, 50, 200, 200);

    c.show();
    return app.exec();
}
```

**为什么 A 的 handler 要 `ignore()`？**
只有事件“未被接受”时，Qt 才会把可传播的鼠标事件向上（父对象）冒泡[cite:0acb7814-3]。A 的 `mousePressEvent` 里 `e->ignore()` 就表达了“我没处理完”，于是 `notify` 把事件转投给 B；B 再走一遍“filter → event → handler”。这样正好满足题目要求“A 的 filter、event、event_handler；B 的 filter、event”。

**通俗理解**：A、B 都是“先过自己的门卫（filter）再进办公室（event）再找具体同事（handler）”的流程。A 的同事说“这事不归我管”（ignore），邮件就往上送到领导 B 那里，B 又先过门卫、再进办公室。C 只是最上面那层楼（顶层窗口）。

---

## 题目2

> **原题**：实现一个“打蚊子”游戏。
> 在屏幕中央有一个600 *400的QWidget，一个用来统计分数的QLabel。
> 一开始会在QWidget内部随机位置生成一个蚊子，当鼠标点击到蚊子以后，旧蚊子消失然后在另一个位置生成新的蚊子，分数增加。
> 提示:这里需要继承QWidget类然后重写paintEvent，在paintEvent当中可以创建QPainter对象绘制各种图形和图片。

### 答案

继承 `QWidget` 自定义一个 `GameBoard`，重写 `paintEvent` 用 `QPainter` 画“蚊子”；重写 `mousePressEvent` 判断点击是否命中蚊子，命中则加分、换位置、`update()` 重绘。

```cpp
#include <QApplication>
#include <QWidget>
#include <QLabel>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QPainter>
#include <QMouseEvent>
#include <QRandomGenerator>
#include <QFont>

// 游戏画板：600x400，负责画蚊子、检测点击
class GameBoard : public QWidget {
    Q_OBJECT
public:
    explicit GameBoard(QWidget *parent = nullptr) : QWidget(parent) {
        setFixedSize(600, 400);
        spawnMosquito();
    }

signals:
    void scoreChanged(int score);          // 分数变化信号，交给外界更新 QLabel

protected:
    void paintEvent(QPaintEvent *) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        p.fillRect(rect(), QColor(230, 245, 230));      // 浅绿背景

        // 画一只“蚊子”：黑色身体 + 翅膀 + 标签
        p.setBrush(Qt::black);
        p.drawEllipse(m_mosquitoPos, 12, 12);           // 身体
        p.setBrush(QColor(160, 190, 230));
        p.drawEllipse(m_mosquitoPos + QPoint(0, -14), 14, 8); // 翅膀
        p.setPen(Qt::white);
        p.setFont(QFont(QStringLiteral("微软雅黑"), 9));
        p.drawText(m_mosquitoPos + QPoint(-8, 4), QStringLiteral("蚊"));
    }

    void mousePressEvent(QMouseEvent *e) override {
        // 命中检测：以蚊子位置为中心的正方形区域
        QRect hit(m_mosquitoPos.x() - 24, m_mosquitoPos.y() - 24, 48, 48);
        if (hit.contains(e->pos())) {
            m_score++;
            emit scoreChanged(m_score);                 // 通知界面更新分数
            spawnMosquito();                            // 旧蚊子消失，新位置生成
            update();                                   // 请求重绘
        }
    }

private:
    void spawnMosquito() {
        int x = QRandomGenerator::global()->bounded(40, width()  - 40);
        int y = QRandomGenerator::global()->bounded(40, height() - 40);
        m_mosquitoPos = QPoint(x, y);
        update();
    }

    QPoint m_mosquitoPos;
    int m_score = 0;
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QWidget win;
    win.setWindowTitle(QStringLiteral("打蚊子"));
    win.resize(800, 600);

    auto *v     = new QVBoxLayout(&win);
    auto *score = new QLabel(QStringLiteral("得分：0"));
    score->setAlignment(Qt::AlignCenter);
    score->setFont(QFont(QStringLiteral("微软雅黑"), 18, QFont::Bold));
    v->addWidget(score);

    auto *h = new QHBoxLayout;
    h->addStretch();                     // 左右拉伸，把 600x400 的板子居中
    auto *board = new GameBoard;
    h->addWidget(board);
    h->addStretch();
    v->addLayout(h);

    QObject::connect(board, &GameBoard::scoreChanged, &win,
                     [score](int s) {
                         score->setText(QStringLiteral("得分：%1").arg(s));
                     });

    win.show();
    return app.exec();
}
```

**要点**
- **绘制**：`paintEvent` 里创建 `QPainter`（栈对象），所有绘制命令在函数内完成；数据变化后调用 `update()`（异步、可合并）触发重绘，不要直接调 `paintEvent`。
- **随机位置**：用 `QRandomGenerator::global()->bounded(min, max)`（Qt5.10+/Qt6 的现代随机 API，替代已过时的 `qrand`）。
- **点击命中**：比较鼠标坐标是否落在蚊子包围盒内；命中后 `emit scoreChanged`，让界面 QLabel 与游戏逻辑解耦。
- **居中**：用 `addStretch()` 让 600×400 的画板在 800×600 窗口中水平居中（也可用 `setAlignment(Qt::AlignHCenter)`）。

**通俗理解**：`paintEvent` 像“画师每次重画整张画布”的机会——你告诉它“蚊子现在在 (x,y)”，它就画在那个位置。点击命中了就加一分、给蚊子换个新坐标、再喊一声“重画”。QLabel 只负责显示分数，由信号通知。

---

## 题目3

> **原题**：什么是窗口？怎么样创建一个窗口？

### 答案

**什么是窗口**

- **窗口（Window）**：Qt 中**没有父部件（parent == nullptr）的顶层 `QWidget`（或其子类）**就是一个独立窗口。它由窗口系统（Windows/Linux/X11/Wayland/macOS）负责装饰：标题栏、边框、关闭/最小化/最大化按钮等。
- 与之相对的是**子部件（child widget）**：作为某个父部件的一部分嵌在父部件内部，跟随父部件一起移动/显示，本身没有系统边框。
- 一个窗口可以包含任意多的子部件（子部件还可以有子部件），构成控件树。

**怎么样创建一个窗口**

1. **最直接**：创建 `QWidget` 对象并 `show()`：

```cpp
#include <QApplication>
#include <QWidget>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    QWidget w;
    w.resize(400, 300);
    w.setWindowTitle(QStringLiteral("我的窗口"));
    w.show();                 // show() 让窗口可见（顶层无父部件）
    return app.exec();        // 进入事件循环，窗口才能响应
}
```

2. **继承 `QWidget`/`QMainWindow`/`QDialog`** 自定义窗口类（最常用），在构造函数里搭界面，`main` 里实例化并 `show()`：

```cpp
class MainWin : public QMainWindow {
    Q_OBJECT
public:
    explicit MainWin(QWidget *parent = nullptr) : QMainWindow(parent) {
        setWindowTitle(QStringLiteral("主窗口"));
        // ... 设置中央部件、菜单栏、工具栏、状态栏
    }
};

// main:  MainWin win; win.show();
```

3. **按用途选基类**：
   - `QWidget`：最基础，普通窗口/自定义控件；
   - `QMainWindow`：带菜单栏、工具栏、状态栏、中央部件的主窗口；
   - `QDialog`：对话框（配合 `exec()` 做模态）；
   - `QWidget` 加 `Qt::Window` 等窗口标志也可强制成为独立窗口。

**注意**：只有 `parent == nullptr` 的顶层部件才是独立窗口；`new QWidget(this)` 创建的是**子部件**，不是窗口。`show()` 是“显示”而非“创建”，`exec()` 用于模态对话框（阻塞直到关闭）。

**通俗理解**：窗口就像“房子”——顶层窗口是带院墙大门（系统边框、标题栏）的独立房子，子控件是房子里的家具。`new` 是“盖房子”，`show()` 是“开门迎客”，`exec()` 是“关门开会”（模态，别的窗口都点不动）。

---

# 作业5（试卷）

## 一、单项选择题

> **1. Qt 事件到达 QObject 对象时，处理顺序正确的是（）**
> A. event() → 事件过滤器 → 具体事件 handler
> B. 事件过滤器 → event() → 具体事件 handler
> C. 具体事件 handler → event() → 事件过滤器
> D. event() → 具体事件 handler → 事件过滤器

**答案：B**

事件过滤器最先执行（`eventFilter` 返回 false 才继续），然后 `event()` 按类型分发给具体 handler [cite:0acb7814-3]。

> **2. 关于 eventFilter 返回值说法正确的是（）**
> A. 返回 true，事件继续向下传递
> B. 返回 true，事件被拦截，不再继续传递
> C. 返回 false 直接销毁对象
> D. 返回 false 代表事件已经处理完毕

**答案：B**

`eventFilter` 返回 `true` 表示事件被拦截，不再调用后续过滤器和目标的 `event()`；返回 `false` 表示放行继续 [cite:0acb7814-3]。

> **3. 事件 handler 中调用 ignore() 的作用是（）**
> A. 事件未处理，继续向上冒泡传递
> B. 事件处理完毕，停止向父对象冒泡传递
> C. 直接丢弃事件，不传递给任何对象
> D. 触发过滤器重新处理

**答案：A**

`ignore()` 表示“未处理”，可传播事件会继续向父对象冒泡；`accept()` 才是“已处理，不再传递” [cite:0acb7814-3]。

> **4. 鼠标事件默认发送给（）**
> A. 焦点所在对象
> B. 鼠标位置下 Z 轴最上层最近的子对象
> C. 根父对象
> D. 任意子对象

**答案：B**

鼠标事件发送给光标下 Z 序最上层、可接收事件的子对象。

> **5. 键盘事件默认发送给（）**
> A. 鼠标悬停的控件
> B. 焦点（focus）所在对象
> C. 主窗口
> D. 第一个创建的子控件

**答案：B**

键盘事件发给拥有焦点的对象 [cite:0acb7814-3]。

> **6. QDialog 调用哪个函数实现模态对话框，阻塞整个进程直到对话框关闭？（）**
> A. exec()　B. show()　C. open()　D. display()

**答案：A**

`exec()` 打开应用级模态对话框并进入嵌套事件循环，关闭后才返回；`show()` 是非模态不阻塞；`open()` 是窗口级模态。

> **7. 使用 QFileDialog 获取单个已存在文件路径的静态函数是（）**
> A. getOpenFileName()　B. getOpenFileNames()　C. getSaveFileName()　D. getExistingDirectory()

**答案：A**

`getOpenFileName()` 返回单个已存在文件；`getOpenFileNames()` 返回多个；`getSaveFileName()` 是保存；`getExistingDirectory()` 是目录。

> **8. QMessageBox 中代表报错、红色叉号图标的函数是（）**
> A. information()　B. question()　C. critical()　D. warning()

**答案：C**

`critical()` 是错误（红色叉）；`warning()` 是警告；`information()` 信息；`question()` 提问。

> **9. 关于 QAction 描述错误的是（）**
> A. QAction 是行为动作，可以同时放在菜单栏和工具栏
> B. QAction 的 triggered() 信号类似按钮的 clicked()
> C. QAction 是控件实体，只能出现在一个界面位置
> D. 可以设置文本、图标、快捷键

**答案：C**

`QAction` 是**可复用的抽象动作**（不是控件实体），同一个 `QAction` 可以同时出现在菜单栏和工具栏等多个位置。A、B、D 都正确，C 是错误说法。

> **10. QRadioButton 的关键信号是（）**
> A. clicked() 无参　B. toggled(bool)　C. textChanged(QString)　D. triggered()

**答案：B**

单选按钮选中/取消选中会发 `toggled(bool)`（勾选状态是否改变的核心信号）。

> **11. 想要限制输入框只能输入 100~20000 的整数，应使用哪个验证器？（）**
> A. QDoubleValidator　B. QIntValidator　C. QRegularExpressionValidator　D. QFileValidator

**答案：B**

`QIntValidator(100, 20000)` 用于整数范围验证（注意：如“代码阅读题1”所说，它不能 100% 保证最终值，但这是最匹配的验证器）。

> **12. QComboBox 选项索引发生改变时触发的信号是（）**
> A. currentIndexChanged(int)　B. textChanged(QString)　C. returnPressed()　D. toggled(bool)

**答案：A**

下拉框当前索引变化发 `currentIndexChanged(int)`。

> **13. 纯代码使用布局的正确步骤是（）**
> A. 创建布局 → 设置容器 layout → 创建控件 → addWidget
> B. 创建容器、控件、布局 → addWidget 添加控件到布局 → 容器 setLayout
> C. setLayout → addWidget → 创建控件
> D. 控件 setLayout → addWidget 到布局

**答案：B**

正确顺序：创建容器/控件/布局对象 → `layout->addWidget(控件)` → `容器->setLayout(布局)`。

> **14. Qt Model/View 架构中，下列说法正确的是（）**
> A. View 修改数据后 Model 自动更新
> B. Model 发生变化时 View 会自动更新
> C. Model 和 View 共用同一个对象
> D. 编辑 View 时直接修改 View，不经过 Delegate

**答案：B**

Model 数据变化通过信号通知，View 自动刷新；A、C、D 均错误。

> **15. QTimer 超时时触发的信号是（）**
> A. finished()　B. timeout()　C. triggered()　D. readyRead()

**答案：B**

`QTimer` 每次超时发 `timeout()`。

> **16. 关于 QByteArray 与 QString 说法错误的是（）**
> A. QString 存储 UTF-16 字符串，读写时会编码转换
> B. QByteArray 保存原始字节，不做编码转换
> C. 读取二进制数据应优先使用 QByteArray
> D. QByteArray 可以直接显示到 UI 界面，不需要转为 QString

**答案：D**

`QByteArray` 是原始字节，不能直接显示到 UI，需转 `QString`；D 是错误说法。

> **17. QFile 的基类中提供 open/read/write 核心接口的是（）**
> A. QIODevice　B. QFileDevice　C. QObject　D. QAbstractSocket

**答案：A**

`open()/read()/write()/readAll()` 等核心 IO 接口定义在 `QIODevice`；`QFileDevice` 提供文件特有接口（seek、flush 等）。

> **18. QTcpSocket 收到对端发来的数据时，触发的信号是（）**
> A. connected()　B. readyRead()　C. stateChanged()　D. disconnected()

**答案：B**

有新数据可读时发 `readyRead()`。

> **19. QTcpSocket 的 connectToHost() 是（）**
> A. 阻塞函数，连接成功后才返回
> B. 非阻塞，调用后立即返回，连接成功通过 connected 信号通知
> C. 同步等待直到收到服务端数据
> D. 直接返回读取到的字节

**答案：B**

`connectToHost()` 异步非阻塞，立即返回；连接成功由 `connected()` 信号通知。

> **20. HTTP 协议无状态，若要把状态保存在客户端，使用的是（）**
> A. Session　B. Token　C. 服务端内存变量　D. 服务端全局变量

**答案：B**

状态存在客户端用 Token（如 JWT、Cookie）；Session 存在服务端。

> **21. 关于 QObject 的 event() 返回值，说法正确的是（）**
> A. event() 返回 false，事件一定向上冒泡传递
> B. event() 返回 false，是否向上冒泡取决于 handler 中的 accept/ignore
> C. event() 返回 true，事件一定会交给 eventFilter 处理
> D. event() 返回 true，代表事件未处理

**答案：B**

默认 `QWidget::event()` 把 handler 的 accept/ignore 状态转成 bool 返回值，是否冒泡最终取决于 handler 是否 accept/ignore（及事件类型是否可传播、有无父对象）[cite:0acb7814-3]。A 的“一定”太绝对；C、D 明显错误。

> **22. QFontDialog 获取字体后，通过哪个布尔变量判断用户是否点击了“确定”？（）**
> A. isValid　B. ok　C. accepted　D. ignored

**答案：B**

`QFontDialog::getFont(&ok, this)` —— 布尔变量名是 `ok`。

> **23. QMainWindow 中用来放置主要业务内容的部件是（）**
> A. QToolBar　B. 中心部件（central widget）　C. QMenuBar　D. 状态栏(statusBar)

**答案：B**

中心部件 `setCentralWidget()` 放置主要业务内容。

> **24. 下面不属于 Qt 容器控件的是（）**
> A. QTabWidget　B. QStackedWidget　C. QTimer　D. QFrame

**答案：C**

`QTimer` 是定时器对象（QObject），不是容器控件。

> **25. 将 Model 中的 Item 禁止拥有子节点后，Model/View 数据结构退化为（）**
> A. 线性表　B. 表格　C. 树　D. 字符串

**答案：A**

节点禁止有子节点后，树退化为线性表（一维列表）。表格是多列无子节点；树是有子节点。

> **26. 下面哪个 Model 代表一维字符串列表模型？（）**
> A. QStandardItemModel　B. QStringListModel　C. QSqlTableModel　D. QAbstractItemModel

**答案：B**

`QStringListModel` 专门展示一维字符串列表（配 `QListView`）。

> **27. 关于 MVC 和 Qt Model/View 的描述，正确的是（）**
> A. Qt Model/View 完全等价于 MVC
> B. Qt 把 View 和 Controller 合并，编辑时临时生成 Delegate
> C. Qt Model/View 中 Controller 是独立类
> D. Delegate 对象长期存在，程序启动时就创建

**答案：B**

Qt 将 View 与 Controller 的角色合并，编辑控件由 Delegate（委托）在需要编辑时临时创建。

---

## 二、判断题

> **1. event() 返回 true，表示事件不再向父对象传递。（）**

**答案：√**

默认 `QWidget::event()` 把 handler 的 accept/ignore 状态转换为布尔返回值：handler 接受事件 → `event()` 返回 true → 事件不再冒泡 [cite:0acb7814-3]。若你自定义 `event()` 不走默认分发逻辑，返回值的语义可能不同，但按标准流程该说法正确。

> **2. 事件冒泡是指事件从父对象向下传递给子对象。（）**

**答案：×**

事件冒泡是**从子对象向上传递到父对象**，方向反了。

> **3. 事件过滤器必须继承从QObject并重写eventFilter函数。（）**

**答案：√**

过滤器对象必须继承 `QObject` 并重写 `bool eventFilter(QObject*, QEvent*)`。

> **4. show() 打开的是模态对话框，会阻塞整个程序执行。（）**

**答案：×**

`show()` 打开的是**非模态**，不阻塞；`exec()` 才是模态并阻塞。

> **5. QMainWindow 中，同一个 QAction 可以同时出现在菜单栏和工具栏。（）**

**答案：√**

`QAction` 可复用，可同时放菜单栏和工具栏。

> **6. QLabel 可以显示图片，调用 setPixmap() 即可。（）**

**答案：√**

`QLabel::setPixmap()` 显示图片。

> **7. 同一容器内多个 QRadioButton 自动互斥；不同容器可以实现多组独立单选。（）**

**答案：√**

同一父容器内 `autoExclusive` 默认开启自动互斥；不同容器互不影响。

> **8. QValidator 可以 100% 保证输入一定落在合法范围内，不需要额外校验。（）**

**答案：×**

`QValidator` 允许“中间态（Intermediate）”输入，不能 100% 保证最终值，仍需在提交时额外校验（见代码阅读题1）。

> **9. QGridLayout 的 addWidget 可以设置 rowSpan、colSpan 实现跨行跨列。（）**

**答案：√**

`addWidget(widget, row, col, rowSpan, colSpan)` 支持跨行跨列。

> **10. QStandardItemModel 修改数据后，关联的 QTableView 需要手动调用 refresh 刷新界面。（）**

**答案：×**

Model 变化通过信号自动通知 View 刷新，无需手动 refresh。

> **11. QTimer 调用 start() 启动定时器，setInterval 设置毫秒间隔。（）**

**答案：√**

`start()` 启动，`setInterval(毫秒)` 设间隔。

> **12. 二进制文件存储的全部是可读 ASCII 字符。（）**

**答案：×**

二进制文件存原始字节，可包含任意值，不限于可读 ASCII。

> **13. QFile 以 Append 模式打开文件，代表追加写入。（）**

**答案：√**

`QIODevice::Append` 表示追加写入。

> **14. TCP 是传输层协议，HTTP 属于应用层协议。（）**

**答案：√**

TCP 在传输层，HTTP 在应用层。

> **15. HTTP 无状态意味着业务完全不能保存登录状态，没有任何补救方案。（）**

**答案：×**

可用 Cookie/Session/Token 等方案补救（见简答6）。

> **16. 对话框使用成员指针持久化对象，只申请一次内存，响应速度更快。（）**

**答案：√**

成员指针只 `new` 一次、多次复用，避免每次构造，响应更快（见简答9）。

> **17. QLabel 继承 QFrame，可以调用 setFrameStyle 设置边框样式。（）**

**答案：√**

`QLabel : public QFrame`，可调用 `setFrameStyle`。

> **18. QStackedWidget 是分页容器，但没有标签栏。（）**

**答案：√**

`QStackedWidget` 只管理分页，不显示标签栏（标签栏是 `QTabWidget`）。

> **19. 二进制文件的存储效率低于文本文件。（）**

**答案：×**

二进制文件存储效率**更高**（无编码转换、更紧凑），文本文件效率低（见简答10）。

---

## 三、填空题

> **1. 事件 handler 中，____代表事件处理完成，不再向上传递；____代表事件未处理，继续向父对象传递。**

**答案：`accept()`；`ignore()`**

> **2. 键盘事件发送给拥有____的对象；鼠标事件发送给鼠标位置下 Z 轴最上层的子对象。**

**答案：焦点（focus）**

> **3. 安装事件过滤器的函数；目标对象调用____。**

**答案：`installEventFilter()`**（`target->installEventFilter(filterObj)`）

> **4. QFileDialog 获取保存文件路径的静态函数是____。**

**答案：`getSaveFileName()`**

> **5. QMainWindow 中菜单栏的类名是____，工具栏的类名是____。**

**答案：`QMenuBar`；`QToolBar`**

> **6. QCheckBox 点击切换勾选状态时，常用的信号是____。**

**答案：`stateChanged(int)`**（也可用 `toggled(bool)`）

> **7. QLineEdit 按下回车键会发射____信号；输入内容变化时发射 textChanged 信号。**

**答案：`returnPressed()`**

> **8. Model/View 中，QTableView 常用的模型类是____；QListView 用来展示一维列表。**

**答案：`QStandardItemModel`**

> **9. QTimer 的头文件是____；设置定时间隔的函数是____。**

**答案：`<QTimer>`；`setInterval()`**

> **10. QFile 以只读方式打开文件的枚举宏是____。**

**答案：`QIODevice::ReadOnly`**

> **11. QTcpSocket 连接成功时触发____信号；断开连接时触发 disconnected 信号。**

**答案：`connected()`**

> **12. HTTP 是____层协议，采用客户端-服务器（C/S）模型。**

**答案：应用**

> **13. 在事件处理函数（handler）中发射____，实现事件捕获输入，信号槽处理业务的编程范式。**

**答案：信号（signal）**

> **14. QColorDialog 获取颜色后，应调用____函数判断颜色选择是否有效。**

**答案：`isValid()`**

> **15. Qt 四大布局 QHBoxLayout、QVBoxLayout、QGridLayout、____。**

**答案：`QFormLayout`**

> **16. QTcpSocket 状态枚举中，连接成功后的状态为____。**

**答案：`ConnectedState`**

---

## 四、简答题

### 简答1

> **简述 Qt 事件的完整处理流程（事件过滤器 → event() → 具体事件 handler），并说明事件冒泡传递的规则。**

**答案**：完整流程见《作业2·题目3》。要点：
1. **产生**：系统/程序产生 `QEvent`（鼠标、键盘、重绘等），经事件循环和 `QApplication::notify()` 分发。
2. **过滤器**：目标对象上安装的每个过滤器依次调用 `eventFilter()`；返回 `true` 拦截，`false` 放行 [cite:0acb7814-3]。
3. **event()**：`QObject::event()` 按 `QEvent::type()` 把事件分发给具体 handler（`mousePressEvent` 等）。
4. **冒泡规则**：鼠标、键盘等**可传播事件**，若 handler 调用了 `ignore()`（未接受），`notify()` 会把同一事件继续发给**父对象**，父对象再走“过滤器→event→handler”，逐级向上，直到被 `accept()` 或到达顶层窗口；若调用了 `accept()` 则不再向上 [cite:0acb7814-3]。默认 `QWidget::event()` 会把 accept/ignore 状态转换成 event() 的布尔返回值告知 notify [cite:0acb7814-3]。

### 简答2

> **模态对话框 exec() 和非模态对话框 show() 的区别是什么？**

**答案**：
- `exec()`：打开**应用级模态**对话框，进入嵌套事件循环，**阻塞调用处**直到对话框关闭才返回；期间用户不能操作其他窗口。适合“必须做选择才能继续”的场景（如保存确认）。
- `show()`：打开**非模态**对话框，**立即返回**，不阻塞；用户可与主窗口和其他窗口同时交互。适合“边看边设置”的窗口（如查找替换）。
- 另外 `open()` 是**窗口级模态**（只阻塞父窗口）。非模态对话框若用栈对象，函数结束对象就销毁，所以通常 `new` 出来或作为成员持久保存。

### 简答3

> **QPushButton 与 QAction 的区别是什么？**

**答案**：
- `QPushButton` 是**具体可见的控件**（`QWidget` 子类），只能放在一个位置，显示文本/图标，`clicked()` 信号。
- `QAction` 是**抽象的动作对象**（`QObject`，不是控件），描述一个“行为”：文本、图标、快捷键、可启用/禁用、`triggered()` 信号；**同一个 QAction 可同时添加到菜单栏、工具栏、右键菜单等多个位置**，一处触发处处同步。
- 用途：按钮适合固定位置的一次性交互；`QAction` 适合“菜单+工具栏+快捷键”统一管理的命令式操作。

### 简答4

> **QByteArray 和 QString 的区别是什么？分别适合什么场景？**

**答案**：
- `QByteArray`：保存**原始字节**（`char*` 数组），不关心编码，按字节操作；适合二进制数据、网络/文件 IO、协议解析。
- `QString`：保存 **UTF-16 Unicode 字符**，面向文本显示与处理，读写有编码转换；适合界面显示、文本处理、本地化。
- 场景：读二进制文件/网络包用 `QByteArray`；显示到 UI 前转 `QString`（`QString::fromUtf8(buf)`）。`QByteArray` 不能直接 `setText` 到 UI。

### 简答5

> **“QTcpSocket 的所有操作都是非阻塞的”，这句话意味着什么？编程时应依靠什么机制获取操作结果？**

**答案**：
- 意味着调用**立即返回**，不等待网络完成：`connectToHost()` 发出后马上返回，连接尚未建立；`write()` 只把数据写进发送缓冲区就返回；`readAll()` 只读当前缓冲区。
- 因此**不能**假设“连接后立刻读就有数据”，必须依靠 **Qt 事件循环 + 信号槽**获取结果：
  - 连接成功 → `connected()`；失败 → `errorOccurred()`；
  - 有数据可读 → `readyRead()`（槽里再 `readAll`/`read`）；
  - 写入完成 → `bytesWritten(qint64)`；
  - 断开 → `disconnected()`。
- 事件循环（`app.exec()`）是这些信号能被分发的必要条件。

### 简答6

> **什么是 HTTP 的无状态性？业务层想要保存登录状态有哪些实现思路？**

**答案**：
- **无状态性**：HTTP 每个请求相互独立，服务器不保留上一次请求的任何信息，每个请求都是“第一次见面”。
- **补救思路**：
  1. **Cookie + Session**：服务器创建会话并保存 Session 状态，把会话 ID 通过 Cookie 交给客户端，客户端每次请求带 Cookie，服务器据此还原状态。
  2. **Token（如 JWT）**：服务器签发带签名/过期时间的令牌给客户端，客户端保存并在请求头携带；服务器验签即知身份，无需在服务端存会话（适合分布式/无状态服务）。
  3. 其他：隐藏域、URL 重写等（传统 Web 手段）。
- 对比：Session 状态在服务端、可服务端强制失效；Token 状态在客户端、服务端“无状态”但难以立刻吊销。

### 简答7

> **简述纯代码方式创建布局的完整步骤。**

**答案**：
1. 创建**容器**（如 `QWidget* parent`）、**控件**（按钮、标签等）、**布局对象**（`QHBoxLayout`/`QVBoxLayout`/`QGridLayout`/`QFormLayout`）。
2. 用 `layout->addWidget(控件)`（可带 stretch 伸缩因子、`Qt::Alignment` 对齐；网格布局可设 row/col/span）把控件加入布局；也可以 `addLayout` 嵌套子布局。
3. 最后 `容器->setLayout(layout)` 把布局安装到容器上。
4. 程序进入 `app.exec()` 后布局自动计算并随窗口缩放调整。

```cpp
QWidget *win = new QWidget;
auto *v = new QVBoxLayout(win);       // 或 new QVBoxLayout; win->setLayout(v);
auto *label = new QLabel("Hello");
auto *btn   = new QPushButton("OK");
v->addWidget(label);
v->addWidget(btn);
win->show();
```

### 简答8

> **简述 Qt Model/View 架构的核心思想。**

**答案**：
- **数据与显示分离**：Model 负责存储/提供数据，View 负责展示，二者通过模型索引与信号联系。
- **Model 变化自动通知**：Model 数据变化时发出信号（`dataChanged`、`rowsInserted` 等），关联的 View 自动刷新，无需手动 refresh。
- **Delegate（委托）**：负责绘制单元格与编辑控件，编辑时临时创建 editor（View 与 Controller 的角色合并）。
- **优点**：一个 Model 可被多个 View 共享；解耦、可复用、易于替换数据源（内存/数据库/文件）；`QAbstractItemModel` 抽象让自定义数据接入方便。
- 与 MVC 关系：Qt 不严格区分 Controller，View 承担 View+Controller 职责，Delegate 负责交互细节。

### 简答9

> **对话框的两种实现方式（静态栈临时创建 VS 成员指针持久对象）各自有什么特点？**

**答案**：
- **栈临时创建**（`QMessageBox::about(this, ...)` 或局部 `QDialog d(this); d.exec();`）：
  - 优点：用完即走、内存自动回收、无泄漏风险；适合偶尔弹出、简单、模态的对话框。
  - 缺点：每次调用都要重新构造对象，有构造开销；对象生命周期短，**不能**用于非模态（`show()` 后函数结束对象就析构，窗口消失）；数据需要在调用前准备好。
- **成员指针持久对象**（`m_dialog = new QDialog(this)` 只 `new` 一次）：
  - 优点：**只申请一次内存**，多次 `show()`/`exec()` 复用，响应更快；适合频繁弹出的对话框；可以跨作用域保存状态（上次的输入）。
  - 缺点：常驻内存；需注意释放（设置 parent 由父对象回收，或用 `deleteLater`）；若内容静态不刷新，可能显示旧数据。
- **经验**：简单/偶尔 → 栈临时（或 `new` + 父对象 + `exec()`）；频繁/非模态/需保留状态 → 成员指针 + 父对象管理。

### 简答10

> **简述文本文件与二进制文件的区别（从存储内容、可读性、存储效率三个角度）。**

**答案**：
- **存储内容**：文本文件存“可读字符”（按某种编码如 UTF-8/GBK）；二进制文件存“原始字节”（内存中的二进制表示，如 int 的 4 字节）。
- **可读性**：文本文件可以用记事本直接阅读；二进制文件通常不可读、需程序解析。
- **存储效率**：文本文件**低**——数字转字符串浪费空间（123 占 3 字节，int 只占 4 字节；大数差异更明显），且读写需编码转换；二进制文件**高**——紧凑、无编码转换、读写快。
- **适用**：配置文件、日志、人类可读数据用文本；性能敏感、结构化数据、图片/音频/压缩包等用二进制。

### 简答11

> **什么是 Table-Tree 结构？说明它如何退化得到表格、树和线性表。**

**答案**：
- **Table-Tree（表树）**：Qt `QAbstractItemModel` 使用的通用结构——一个**树**，但每个节点是**一行（含多个列）**，即“树 × 表格”的复合结构（一个节点可有父节点、子节点，并有多列数据）。
- **退化规律**：
  - 所有节点**无子节点**且**只有一列** → **线性表**（一维列表，`QListView`/`QStringListModel`）。
  - 所有节点**无子节点**但**多列** → **表格**（二维表，`QTableView`/`QStandardItemModel`）。
  - 节点**可有子节点** → **树**（`QTreeView`，每行多列就是 Table-Tree 的完整形态）。
- 一句话：列决定“表”，行方向上的父子关系决定“树”；去掉子节点只剩表，单列表就是线性表。

---

## 五、代码阅读题

### 题目1

> **原题**：
> ```cpp
> QIntValidator *v = new QIntValidator(100, 5000, this);
> ui->lineEdit->setValidator(v);
> ```
> **问题**：QIntValidator 能否保证 lineEdit 的输入一定在 100~5000 范围内？请说明原因

**答案**：**不能保证**。

原因：`QIntValidator` 的 `validate()` 返回三种状态——`Acceptable`（合法）、`Invalid`（非法，直接拒绝输入）、**`Intermediate`（中间态，允许继续输入）**。比如输入过程中 `"5"`、`"50"`、`"999"` 都可能被判定为 `Intermediate`（因为后面还能补成 500、5000 等合法值），于是用户可以停留在 `"50"` 这样的**范围外**数值上不继续输入。此外，负数、空串等也可能是中间态。所以验证器只能“尽力约束”，**不能 100% 保证最终值落在 [100, 5000]**。要严格保证，应在提交时（`returnPressed`/`editingFinished`）再次校验或做 `fixup` 修正。

### 题目2

> **原题**：
> ```cpp
> QTimer *timer = new QTimer(this);
> timer->setInterval(1000);
> //connect(timer,&QTimer::timeout,this,&MainWindow::slotTime);
> timer->start();
> ```
> **问题**：定时器会不会定时触发 slotTime？为什么？

**答案**：**不会**。

定时器本身会每 1000ms 发出 `timeout()` 信号，但**连接语句被注释掉了**，`timeout()` 没有连接到任何槽（`slotTime` 没有任何接收者），所以 `slotTime` 永远不会被调用。连接是触发的前提。

### 题目3

> **原题**：
> ```cpp
> QTcpSocket *sock = new QTcpSocket(this);
> sock->connectToHost("127.0.0.1",8888);
> QByteArray data = sock->readAll();
> ```
> **问题**：这里的 readAll() 能读到服务端返回的数据？请解释原因。

**答案**：**不能**。

`connectToHost()` 是**非阻塞异步**的，调用后立即返回，此时连接尚未建立（处于 `ConnectingState`），更不可能已经收到服务端数据，`readAll()` 只能读当前接收缓冲区，此刻为空。正确做法：连接成功后（`connected` 信号）再读，并且用 `readyRead()` 信号在事件循环驱动的槽函数里读取数据。

### 题目4

> **原题**：
> ```cpp
> QFile f("test.txt");
> f.open(QIODevice::readOnly);
> QByteArray buf = f.readAll();
> ui->label->setText(buf);
> ```
> **问题**：这段代码存在哪些严重错误？

**答案**：
1. **未检查 `f.open()` 返回值**：文件可能不存在/无权限，`open()` 失败后 `readAll()` 返回空 `QByteArray`，程序也不会报错，问题被掩盖。应先 `if (!f.open(QIODevice::ReadOnly)) { ...报错 return; }`。
2. **`setText(buf)` 编码问题**：`setText` 需要 `QString`，`QByteArray` 隐式转 `QString` 按 **UTF-8** 解释；若文件是 GBK 等其他编码，中文会**乱码**。应显式 `QString::fromUtf8(buf)` 或按实际编码转换。
3. **未判断读取结果**：文件为空或读取失败时，未给出任何提示。
4. **相对路径问题**：`"test.txt"` 依赖当前工作目录，程序在别的目录运行就找不到，建议用绝对路径或 `QFileDialog` 选择。
5. **未显式 `close()`**：虽然 `QFile` 析构（RAII）会自动关闭，但读完应立即释放文件句柄。
6. **大文件一次性 `readAll()`**：会把整个文件读入内存，大文件可能内存不足；应分块读取。

---

## 六、编程题

### 题目1

> **原题**：使用 QFileDialog 选择打开单个文件，读取全部文本内容，并显示到 textBrowser 中。请写出核心代码片段。

**答案**：

```cpp
// 选择文件
QString filePath = QFileDialog::getOpenFileName(
        this, QStringLiteral("选择文件"), QString(),
        QStringLiteral("文本文件 (*.txt *.log);;所有文件 (*)"));
if (filePath.isEmpty())
    return;                       // 用户取消

// 打开并读取
QFile file(filePath);
if (!file.open(QIODevice::ReadOnly)) {
    QMessageBox::critical(this, QStringLiteral("错误"),
                          QStringLiteral("无法打开文件：%1").arg(file.errorString()));
    return;
}
QByteArray data = file.readAll();
file.close();

// 显示（按 UTF-8 解码，避免乱码）
ui->textBrowser->setPlainText(QString::fromUtf8(data));
```

### 题目2

> **原题**：创建一个 QStandardItemModel 并与 QTableView 关联，设置表头为{"学号","姓名"}，创建一行追加数据{"2025001","张三"}。

**答案**：

```cpp
auto *model = new QStandardItemModel(this);

// 设置表头
model->setHorizontalHeaderLabels({QStringLiteral("学号"), QStringLiteral("姓名")});

// 追加一行数据
QList<QStandardItem *> row;
row << new QStandardItem(QStringLiteral("2025001"))
    << new QStandardItem(QStringLiteral("张三"));
model->appendRow(row);

// 与 QTableView 关联（Model 变化自动刷新 View，无需手动 refresh）
ui->tableView->setModel(model);
```

### 题目3

> **原题**：创建一个300ms的定时器，将 timeout 信号连接到槽函数 updateUI()，并启动定时器。

**答案**：

```cpp
QTimer *timer = new QTimer(this);
timer->setInterval(300);                              // 300ms
connect(timer, &QTimer::timeout, this, &MainWindow::updateUI);
timer->start();                                       // 启动
```

---

## 附：纠正题面/旧材料中几处过时或易错点

- **QSet::intersect 的语义**：Qt6 里它仍然是**原地修改、返回自身引用**，返回新集合请用 `operator&` 或“先拷贝再 intersect”[cite:0acb7814-2]。
- **QStack**：Qt6 中仍存在且继承 `QList`[cite:0acb7814-1]，但现代写法更推荐 `std::stack`。
- **旧式 `connect`（SIGNAL/SLOT 字符串）**：类型不安全，应一律改用 Qt5/6 的函数指针式 `connect`。
- **`qrand()` 已过时**：随机数统一用 `QRandomGenerator`。
- **`QString::SkipEmptyParts` 已改名** `Qt::SkipEmptyParts`（Qt 5.14+）。
- **`event()` 返回值与 accept/ignore 的关系**：默认 `QWidget::event()` 会把 handler 的 accept/ignore 转成布尔返回值（接受→true）[cite:0acb7814-3]；自定义 `event()` 若不遵循默认分发，需自己保证语义一致。

