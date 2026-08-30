# 第一套：文本查询 Query 体系（动态多态）

## 一、问答题

### 第 1 题

> 在文本查询扩展中，Query类为什么要设计成包含shared_ptr<Query_base>？这种设计有什么好处？

**答案：**

`Query` 类的核心成员是一个 `std::shared_ptr<Query_base>`，即用一个**智能指针指向基类**来代表一个查询。好处如下：

1. **值语义（Value Semantics）**：用户用 `Query q = Query("fiery") & Query("bird");` 时，操作起来像操作一个普通的"值"（可拷贝、可赋值、可放进容器），而 `shared_ptr` 的拷贝让多个 `Query` 可以安全地共享同一个查询子树，不用手写深拷贝。
2. **自动生命周期管理**：查询表达式是由 `WordQuery/AndQuery/OrQuery/NotQuery` 组成的一棵**对象树**。`shared_ptr` 引用计数归零时自动销毁整棵树，杜绝了手写 `delete` 的内存泄漏和悬垂指针问题。
3. **动态多态的载体**：`shared_ptr<Query_base>` 是"指向基类的指针"，这正是触发动态绑定（虚函数按运行时真实类型调用）的前提。`Query::eval`/`rep` 内部转发给 `q->eval()`/`q->rep()`，由被指向的派生对象决定行为。
4. **隐藏内部实现**：用户只看到 `Query` 这个门面（Facade），不需要知道 `AndQuery` 这些类存在，降低耦合，符合开闭原则。

**通俗解释**：可以把 `Query` 想象成"遥控器"，`shared_ptr<Query_base>` 就是遥控器里指向"实际机器"（WordQuery/AndQuery…）的指针。遥控器可以复制，但无论复制多少份，控制的是同一台机器；机器坏了没人用时，遥控器自动把它回收。这样用户只碰遥控器，永远不用碰机器本身。

---

### 第 2 题

> 解释WordQuery、AndQuery、OrQuery、NotQuery的继承关系，并说明它们各自eval函数的实现思路（如何求交集、并集、差集）。

**答案：**

**继承关系**：这四个类都**公有继承自抽象基类 `Query_base`**，覆写其纯虚函数 `eval()` 与 `rep()`：

```
              Query_base（纯虚: eval / rep，protected 构造，virtual 析构）
                    ▲
   ┌────────┬───────┴────────┬──────────┐
 WordQuery  AndQuery        OrQuery    NotQuery
```

（严格说教材里还隐含一个 `Query_base` 下的公共接口 `Query` 作为门面类，它 `has-a` 一个 `shared_ptr<Query_base>`，不是继承关系。）

**各 eval 实现思路**（都以 `TextQuery&` 为参数，返回 `QueryResult`，其中保存一个 `set<line_no>` 行号集合）：

| 类          | 构造参数                             | eval 思路                                                    | 集合运算       |
| ----------- | ------------------------------------ | ------------------------------------------------------------ | -------------- |
| `WordQuery` | `std::string word`                   | 直接调用 `TextQuery::query(word)` 拿到该单词的行号集合       | 无（基础查询） |
| `AndQuery`  | `const Query& lhs, const Query& rhs` | 先 `lhs.eval(t)`、`rhs.eval(t)` 得到两个行号集合，再用 `set_intersection` 求**交集** | 交集           |
| `OrQuery`   | `const Query& lhs, const Query& rhs` | 同样先求两个集合，再用 `set_union` 求**并集**                | 并集           |
| `NotQuery`  | `const Query& query`                 | 先求操作数的行号集合，再构造"全部行号集合 `{1..n}`"，用 `set_difference` 求**差集（补集）**，即"没出现该单词的行" | 差集/补集      |

**实现细节**：三个二元/一元运算类在 `eval` 里是先通过内部的 `Query` 成员（`lhs`/`rhs`/`q`）调用其 `eval()`——注意这又会触发一次**动态多态**分发，最后返回两个 `set<line_no>`，然后调用 `<algorithm>` 里的 `set_intersection` / `set_union` / `set_difference`（这三个算法要求输入集合有序，正好 `set` 天然有序）。

`rep()` 对应返回 `"fiery"`、`"(a & b)"`、`"(a | b)"`、`"~(a)"` 这样的字符串，用于打印表达式。

**通俗解释**：`AndQuery` 就像"同时满足两个条件的筛选器"：先分别把两个条件各自命中的行号筛出来，再取两边都有的行号（交集）；`OrQuery` 取两边至少一方的行号（并集）；`NotQuery` 反过来，从"所有行"里挖掉命中行（补集）。

---

### 第 3 题

> 动态多态在Query体系中被激活的五个条件是什么？请结合Query对象调用eval的过程说明。

**答案：**

C++ 中**动态绑定（运行时多态）**被激活，需要同时满足五个条件：

1. **存在继承关系**：`WordQuery` 等派生类继承自基类 `Query_base`；
2. **函数是虚函数**：`eval`/`rep` 在基类中声明为 `virtual`（且是纯虚），派生类覆写；
3. **通过基类的指针或引用调用**：即用 `Query_base*` 或 `Query_base&`（而不是基类对象本身）去调用；
4. **被调用的确实是虚函数**：只有虚函数才走动态绑定，非虚函数一律静态绑定；
5. **在运行时才决定调用哪个版本**：编译器根据**指针实际指向的对象类型**（而非静态类型）在运行时查 vtable 决定跳转目标。

**结合 `Query::eval` 看**：

```cpp
class Query {
    std::shared_ptr<Query_base> q;   // ① 基类指针（智能指针）
public:
    QueryResult eval(const TextQuery &t) const { return q->eval(t); }  // ③④⑤
};
```

调用 `Query q = ...; q.eval(t);` 时，`Query::eval` 内部执行 `q->eval(t)`：

- `q` 的类型是 `shared_ptr<Query_base>`（→ 满足条件 3：基类指针）；
- `eval` 是虚函数（→ 满足条件 2、4）；
- 实际对象可能是 `WordQuery`/`AndQuery`/…（→ 满足条件 1）；
- 运行到 `q->eval(t)` 这一行时，按 `q` 指向的真实对象类型动态查表（→ 满足条件 5）。

于是 `OrQuery` 的对象就会执行 `OrQuery::eval`，与 `Query` 的静态类型无关。

**注意反例**（打破条件时退化为静态绑定）：若直接拿 `Query_base` 对象（而非指针/引用）调用，会发生**对象切片（slicing）**，调用的是基类版本；若函数没加 `virtual`，也按静态类型绑定。

**通俗解释**：五个条件像"打电话必须五样都齐"——有父子关系（1）、电话号码本上有这个号码（2、虚函数）、用的是号码本而不是本人（3、指针/引用）、你拨的确实是那个号码（4）、电话公司按你拨的号接对人（5、运行时查表）。少一样，就只会按"表面身份"办事（静态绑定）。

---

### 第 4 题

> STL的六大组件是什么？各自的作用是什么？

**答案：**

STL 的**六大组件**（这也是《STL 源码剖析》的标准分类）：

| #    | 组件                                      | 作用                                                         |
| ---- | ----------------------------------------- | ------------------------------------------------------------ |
| 1    | **容器（Container）**                     | 存储数据的结构，如 `vector/deque/list/set/map/unordered_*` 等 |
| 2    | **算法（Algorithm）**                     | 对容器数据进行通用操作，如 `sort/find/copy/for_each` 等，**与容器解耦** |
| 3    | **迭代器（Iterator）**                    | 容器与算法之间的"粘合剂"，把"访问某个元素"抽象成统一接口（`*`、`++`、`==`），让同一算法可用于不同容器 |
| 4    | **函数对象（Functor / Function Object）** | 重载了 `operator()` 的对象，可像函数一样调用，常用于算法策略参数（比较器、谓词），如 `less<int>` |
| 5    | **适配器（Adapter）**                     | 对现有组件进行再包装：容器适配器（`stack/queue/priority_queue`）、迭代器适配器（`back_inserter`、`reverse_iterator`）、函数适配器（`std::bind`、`mem_fn`） |
| 6    | **空间配置器（Allocator）**               | 负责内存的分配与释放（以及对象构造/析构），把"内存管理"从容器中抽离出来，允许自定义分配策略 |

**通俗解释**：可以类比一个工厂——容器是"仓库"（放货），算法是"流水线工序"（处理货），迭代器是"传送带和机械手"（把货从仓库送到流水线），函数对象是"给流水线换的模具"（决定怎么处理），适配器是"改装套件"（把现成机器改造成特殊用途），配置器是"后勤部门"（负责批地皮/建仓库、回收仓库）。

---

### 第 5 题

> 序列式容器（vector、deque、list）都支持哪五种初始化方式？请分别举例。

**答案：**

以 `vector` 为例（`deque`/`list` 完全一样，把模板名替换即可）：

1. **默认构造（空容器）**
   ```cpp
   vector<int> v1;
   ```
2. **指定大小（默认值初始化元素）**
   ```cpp
   vector<int> v2(10);            // 10 个 0
   ```
3. **指定大小 + 初值**
   ```cpp
   vector<int> v3(10, 7);         // 10 个 7
   ```
4. **迭代器区间构造（范围构造）**
   ```cpp
   int arr[] = {1,2,3,4,5};
   vector<int> v4(std::begin(arr), std::end(arr));   // 拷贝数组
   // 或者从另一个容器拷贝一段：
   vector<int> v4b(v3.begin(), v3.begin()+3);
   ```
5. **初始化列表（C++11 起）**
   ```cpp
   vector<int> v5{1,2,3,4,5};
   ```

**补充**：还有一个"拷贝构造" `vector<int> v6(v3);` 也常算作一种。另外 `list`/`deque` 与 `vector` 在这五种方式上接口一致（因为它们都有 `size` 构造、`iterator` 范围构造等公共接口），这也是"序列式容器"的共性。

**通俗解释**：五种方式对应五种"开仓库"的方式——空仓（1）、只定货架数不定货（2）、定货架数还摆好货（3）、从别的仓库搬一段货（4）、当场开单子装货（5）。

---

### 第 6 题

> vector、deque、list在遍历方式上有什么异同？为什么list不支持下标访问？

**答案：**

**相同点**：三者都支持**用迭代器遍历**（`begin()/end()`、范围 for `for (auto& x : c)`），也支持 `front()/back()` 访问首尾；都支持 `push_back` 等公共序列接口。

**不同点（遍历方式上的关键差异）**：

| 容器     | 底层存储                  | 迭代器类别     | 支持 `operator[]`？ | 遍历性能特点                                   |
| -------- | ------------------------- | -------------- | ------------------- | ---------------------------------------------- |
| `vector` | 连续内存                  | 随机访问迭代器 | ✅ O(1)              | 最块，缓存友好；支持 `it + n`、`it1 - it2`     |
| `deque`  | 分块连续（中控器+缓冲区） | 随机访问迭代器 | ✅ O(1)              | 支持下标，但底层要跨缓冲区跳转，比 vector 略慢 |
| `list`   | 双向链表节点              | 双向迭代器     | ❌ 不支持            | 只能 `++/--` 逐个走，无缓存局部性              |

**为什么 `list` 不支持下标访问**：

- `list` 的节点在内存中是**离散的**，每个节点包含元素值 + 前后指针，物理上不连续；
- `[]` 需要 O(1) 的"第 i 个元素"定位，这要求**随机访问迭代器**（指针式跳转），而 list 只能从头部/尾部一个个 `++` 走过去，定位第 i 个元素是 **O(n)**；
- 因此 `list` 只提供 `operator++/--` 这类**双向迭代器**能力，没有 `[]`。若必须按位置访问，用 `std::advance(it, i)`（也是 O(n)）或改用 `vector/deque`。

**通俗解释**：`vector` 像一格挨一格的连续货架，报个号就能直接走到第 i 格；`deque` 像分了好多排、排之间用传送带连接的大仓库，报号能算出来在哪个分区再过去；`list` 像手拉手站成一排的人，你只能顺着拉手一个个"传话"过去（`++`），没法凭空瞬移到第 i 个人，所以不能 `list[i]`。

---

### 第 7 题

> 在表达式 Query q = Query("fiery") & Query("bird"); 执行时，operator& 接收的参数类型是什么？它返回的是 Query 对象，但函数内部返回的是 std::shared_ptr<Query_base>，这里发生了什么构造或类型转换？

**答案：**

**1）operator& 接收的参数类型**

```cpp
Query operator&(const Query &lhs, const Query &rhs) {
    return std::shared_ptr<Query_base>(new AndQuery(lhs, rhs));
}
```

`Query("fiery")` 和 `Query("bird")` 是两个临时 `Query` 对象，被绑定到形参 `const Query&` 上——所以 `operator&` **接收的是两个 `const Query&`（引用）**。临时对象可以绑定到 const 引用，生命周期延长到整个表达式结束。

**2）内部发生什么（shared_ptr → Query）**

- `new AndQuery(lhs, rhs)` 在堆上创建 `AndQuery`；
- 用裸指针构造 `shared_ptr<Query_base>`，完成**向上转型**（派生 → 基类，隐式）；
- `return` 语句返回类型是 `Query`，而表达式是 `shared_ptr<Query_base>`。这里走的是 **`Query` 的转换构造函数（converting constructor）**：

```cpp
class Query {
    friend Query operator&(const Query&, const Query&);
    Query(std::shared_ptr<Query_base> q) : q(q) {}   // 非 explicit 转换构造函数
public:
    Query(const std::string &s) : q(new WordQuery(s)) {}
    ...
};
```

`return shared_ptr<...>` 属于**拷贝初始化（copy-initialization）**，编译器发现 `Query` 有一个接受 `shared_ptr<Query_base>` 的**非 explicit 构造函数**，于是调用它，把 `shared_ptr` 转换成一个新的 `Query` 对象，这个 `Query` 内部持有同一个 `shared_ptr`（引用计数 +1）。最后 `Query q` 再从返回值拷贝/移动得到最终对象。

**关键点**：这个构造函数**故意不声明 `explicit`**，否则 `return shared_ptr` 无法隐式转换，`operator&` 就没法这样写了。

**通俗解释**：`operator&` 就像"组装车间"：先造出一台 `AndQuery` 机器（`new`），把它交给"提货单"（`shared_ptr`），然后**再套上一个外壳变成 `Query`**（转换构造），因为用户只认 `Query` 外壳。整个过程中真正干活的是里面的 `AndQuery`，外壳只是门面。

---

### 第 8 题

> Query_base 为什么要将构造函数/析构函数设为 protected 或将 eval/rep 设为 pure virtual，并将 Query 设为友元类？

**答案：**

这四件事分别解决不同问题：

**1）构造函数设为 `protected`**
- `Query_base` 是抽象基类，本来就不该被直接实例化。把**构造放 protected** 后，只有派生类（`WordQuery` 等）和友元能构造它，外部代码无法 `Query_base q;` 创建"没有含义"的基类对象；
- 这也强制所有用户只能通过 `Query` 门面类来创建查询。

**2）析构函数设为 `virtual`（且 protected 或 public）**
- 因为对象总是通过 `shared_ptr<Query_base>`（基类指针）销毁，若析构函数非虚，`delete` 基类指针时**只调用基类析构、不调用派生类析构**，导致派生类资源泄漏/未定义行为；
- `virtual ~Query_base() = default;` 保证通过基类指针删除时能正确析构到最派生类。这也是"用基类指针管理多态对象，基类必须有虚析构"这条铁律。

**3）`eval/rep` 设为纯虚函数（pure virtual）**
- 让 `Query_base` 成为**抽象类**，杜绝实例化；
- 强制所有派生类必须实现 `eval`/`rep`（接口契约），没有"通用实现"这种无意义默认。

**4）`Query` 设为友元类（friend）**
- 在教材实现里，`eval/rep` 被声明为 `private` 纯虚（`private virtual`），目的是"接口与实现分离"——派生类照常覆写，但**只有 `Query` 这个门面能调用它们**；
- `Query` 不是 `Query_base` 的派生类，按访问控制规则本不能访问 private 成员，所以把它声明为 `friend class Query;`，让 `Query::eval()` 内部能执行 `q->eval(t)`、`Query` 的 `operator<<` 能调用 `q->rep()`；
- 好处：外部用户只能通过 `Query` 的公共接口使用查询体系，无法绕过门面直接触碰 `eval/rep` 这些内部细节（信息隐藏 + 控制多态入口）。

**通俗解释**：把 `Query_base` 想成一份"保密图纸"。图纸本身不让你直接照着造（protected 构造 + 纯虚），防止你造出个四不像；图纸上标注了"继承人必须会干的活"（纯虚 eval/rep）；而你手里那张"使用许可"（friend）只发给了 `Query` 门面，只有它能打开图纸去指挥真正的机器干活。别人想用，只能通过 `Query` 这个正规入口。

---

## 二、编程题

### 编程题 1

> 实现一个简单的TextQuery类，包含get_file（读文件存储行）和query（返回某个单词的行号集合）成员函数，使用shared_ptr管理动态资源。

**答案：**

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <vector>
#include <string>
#include <map>
#include <set>
#include <memory>

class TextQuery {
public:
    using line_no = std::vector<std::string>::size_type;

    // 读文件：把每一行存进 vector<string>，用 shared_ptr 托管
    void readFile(const std::string &filename) {
        file = std::make_shared<std::vector<std::string>>();
        std::ifstream ifs(filename);
        std::string line;
        while (std::getline(ifs, line))
            file->push_back(line);
    }

    // get_file：返回整个文件（行集合），shared_ptr 直接返回
    std::shared_ptr<std::vector<std::string>> get_file() const {
        return file;
    }

    // query：返回某个单词出现的所有行号集合（1-based 行号，方便人读）
    std::shared_ptr<std::set<line_no>> query(const std::string &word) const {
        // 惰性建立“单词 -> 行号集合”索引
        static std::map<std::string, std::shared_ptr<std::set<line_no>>> index;
        auto it = index.find(word);
        if (it != index.end()) return it->second;

        auto lines = std::make_shared<std::set<line_no>>();
        for (line_no i = 0; i < file->size(); ++i) {
            std::istringstream iss((*file)[i]);
            std::string w;
            while (iss >> w)
                if (w == word) { lines->insert(i + 1); break; }  // 一行只计一次
        }
        index[word] = lines;
        return lines;
    }

private:
    std::shared_ptr<std::vector<std::string>> file;   // 托管所有行
};

int main() {
    TextQuery tq;
    tq.readFile("data.txt");

    auto file = tq.get_file();
    std::cout << "总行数: " << file->size() << "\n";

    auto lines = tq.query("fiery");
    std::cout << "单词 fiery 出现在行号: ";
    for (auto n : *lines) std::cout << n << " ";
    std::cout << "\n";
    return 0;
}
```

**通俗解释**：`file` 用 `shared_ptr` 托管的 `vector<string>` 存每一行；`query` 会建立一张"单词 → 行号集合"的索引（也是 `shared_ptr` 托管），这样同一单词查第二次不用重扫全文。整段代码没有任何手写 `delete`——所有堆对象都由 `shared_ptr` 自动管理，这正是这类"结果对象要到处共享"场景的正确姿势。

---

### 编程题 2

> 编写代码演示vector、deque、list的尾部插入（push_back）和尾部删除（pop_back）操作，并观察其行为。

**答案：**

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>

template <typename C>
void show(const C &c) {
    std::cout << "[";
    for (auto &x : c) std::cout << x << " ";
    std::cout << "]  size=" << c.size() << "\n";
}

int main() {
    // ---- vector ----
    std::vector<int> v;
    v.push_back(1); v.push_back(2); v.push_back(3);
    std::cout << "vector push_back 后: "; show(v);
    v.pop_back();
    std::cout << "vector pop_back 后: "; show(v);

    // ---- deque ----
    std::deque<int> d;
    d.push_back(1); d.push_back(2); d.push_back(3);
    std::cout << "deque push_back 后: "; show(d);
    d.pop_back();
    std::cout << "deque pop_back 后: "; show(d);

    // ---- list ----
    std::list<int> l;
    l.push_back(1); l.push_back(2); l.push_back(3);
    std::cout << "list push_back 后: "; show(l);
    l.pop_back();
    std::cout << "list pop_back 后: "; show(l);

    // 观察点：pop_back 不会释放 capacity（vector 的内存并不缩回去）
    std::cout << "vector capacity 在 pop_back 后仍为 " << v.capacity() << "\n";
    return 0;
}
```

**行为观察（答案要点）**：

- 三个容器的 `push_back` 都是**尾插**、`pop_back` 都是**尾删**，公共接口一致；
- `vector`：`pop_back` 只减小 `size`，**不释放 capacity**（内存还给的是"逻辑上"，物理缓冲仍占着），所以再 `push_back` 不会立即扩容；`push_back` 在 `size == capacity` 时触发扩容（一般翻倍）；
- `deque`：尾插/尾删均摊 O(1)，且不会像 vector 那样整体搬移已有元素（deque 只在中控器层做增量管理）；
- `list`：尾插/尾删 O(1)，而且**不会使任何已有元素的引用/迭代器失效**（链表只是改指针），这是与 vector 最大的行为差异之一。

**通俗解释**：三兄弟在"队尾加人/减人"这件事上长得一模一样（相同的 API），但内部不同——`vector` 像一排连体的座位，加人时座位不够就整体换一排更大的（扩容）；`deque` 像一排排桌子拼的长桌，加人时在末尾添一张桌子即可；`list` 像手拉手的人链，加人只是让新人和末尾人握手（改指针），其他人完全不受影响。

---

# 第二套：vector / deque / list 底层剖析

## 一、问答题

### 第 1 题

> vector的底层源码中，迭代器是什么类型？为什么vector的迭代器可以像指针一样进行算术运算？

**答案：**

**迭代器类型**：标准只规定 vector 的迭代器是**随机访问迭代器（Random Access Iterator）**，具体类型由实现决定。

- 在 **libstdc++（GCC）** 中，`vector<T>::iterator` 实际是 `__gnu_cxx::__normal_iterator<T*, vector<T>>`——一个**内部只包着一个裸指针 `T*` 的薄封装类**（早期版本甚至直接就是裸指针 `T*`）；
- 在 **MSVC** 中，是 `vector_iterator`（内部持有 `T*`）。
- 无论哪种实现，它的**行为和裸指针等价**：`sizeof(iterator) == sizeof(T*)`。

**为什么能像指针做算术运算**：

1. **底层是连续内存**：vector 的元素存放在**一段连续堆内存**中，第 i 个元素地址 = 首地址 + i * sizeof(T)；
2. 因为连续，`it + n` 就等价于"把内部指针向后移动 n 个元素"（`ptr + n`），`it1 - it2` 等价于 `(ptr1 - ptr2) / sizeof(T)`，得到元素个数差；
3. `__normal_iterator` 重载了 `+ - += -= [] < <= > >=` 等全部随机访问运算符，全部转发给内部裸指针，所以用起来"就像指针"；
4. 同时它又是**类而非裸指针**，能携带类型信息、配合 `std::iterator_traits` 正确报告 `iterator_category = random_access_iterator_tag`，这是裸指针做不到的整洁封装。

**注意**：正因为这种"指针语义"，vector 迭代器一旦发生扩容/插入移位就会失效（指向的内存地址变了），用起来要格外小心（详见后面迭代器失效题）。

**通俗解释**：vector 的元素像一条连续排列的停车位，迭代器就是"你现在站在哪个车位"的坐标。因为车位是挨着的，你报"往前走 3 个车位"就是 `it+3`，报"你和另一个坐标差几个车位"就是 `it1-it2`。`list` 为什么不行？因为它的"车位"散落各处，报数字根本算不出位置。

---

### 第 2 题

> 解释deque的物理结构与逻辑结构（中控器、缓冲区），为什么说deque是"逻辑连续，物理分散"？

**答案：**

**逻辑结构**：`deque`（double-ended queue）对外表现为一个**可以在头尾双端插入/删除、且按下标 O(1) 访问**的"连续序列"——`deque[i]` 像数组一样可用。

**物理结构**：底层由**中控器（map，也叫"映射表"）** 和若干**缓冲区（buffer）** 组成：

```
       中控器 map（指针数组）
   ┌──────┬──────┬──────┬──────┬──────┐
   │ ptr  │ ptr  │ ptr  │ ptr  │ ptr  │
   └──┴───┴───┴───┴───┴───┴──────┘
     │      │      │      │      │
  ┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐
  │buf[0]││buf[1]││buf[2]││buf[3]││buf[4]│   ← 每个缓冲区是一段连续小内存
  └─────┘└─────┘└─────┘└─────┘└─────┘
```

- **缓冲区（buffer）**：一小块连续内存，默认每块能装固定个数的元素（libstdc++ 里通常每块 512 字节，即 `512/sizeof(T)` 个元素）；
- **中控器（map）**：一个"指针数组"，数组的每一项指向一个缓冲区；
- 元素可以**跨越缓冲区**存放：前一个缓冲区的末尾、下一个缓冲区的开头在逻辑上是相邻的。

**为什么说"逻辑连续，物理分散"**：
- **逻辑连续**：从使用者的角度看，`deque[i]` 与"按顺序 ++ 迭代器"都表现为一段连续的序列，随机访问 O(1)；
- **物理分散**：各缓冲区在**堆内存中不一定相邻**，是分散分配的；相邻"逻辑位置"的元素可能落在不同的缓冲区里。
- 因此：deque 的随机访问虽然 O(1)，但必须**先算出元素落在哪个缓冲区、再算出缓冲区内偏移**（`it._M_node` 定位缓冲区、`_M_cur` 定位元素），比 vector 的纯地址运算多一步，通常略慢；
- 也因为"物理分散"，deque **没有"整个数据连续"这一保证**，不能用裸指针做 `begin + n` 这种算术（但迭代器可以）。

**通俗解释**：deque 像"用传送带连起来的一排排货架"。货架（缓冲区）分散在仓库各处，但传送带（中控器 + 迭代器跳转）让你报号就能到达逻辑上第 i 个货位。你看着是连续的一排货，实际货架之间隔着过道（物理不连续）。

---

### 第 3 题

> vector的insert操作可能导致迭代器失效，原因是什么？如何避免？

**答案：**

**失效原因**分两种情况：

1. **插入触发扩容（reallocation）**：当 `size() == capacity()` 时，insert 会分配一块更大的内存、把旧元素移动/拷贝过去、释放旧内存。此时**所有**指向旧内存的迭代器、引用、指针全部失效（地址变了）。
2. **插入未触发扩容（还有空余 capacity）**：元素整体**从插入点开始向后搬移一个位置**，所以：
   - **插入点及之后**的所有迭代器/引用/指针都失效（元素位置变了）；
   - 插入点之前的迭代器仍然有效（前提是没有扩容）。

**如何避免 / 正确做法**：

1. **预留容量**：预先用 `v.reserve(n)` 保证 `capacity` 足够，避免插入时扩容（但不改变"插入点之后移位"的问题）；
2. **利用 insert 的返回值**：`insert` 返回指向**新插入元素**的迭代器，用它继续后续操作，不要用旧的；
3. **用下标代替迭代器**（当允许时）：需要连续插入时用 `int` 索引配合 `std::distance` 重新计算；
4. **换个容器**：如果头/中部插入频繁，改用 `deque`（中插仍可能失效）或 `list`（插入**永不使已有迭代器失效**）。

**经典正确示例**（在某位置连续插入多个值）：

```cpp
std::vector<int> v{1,2,3};
auto it = v.insert(v.begin(), 10);   // 用返回值刷新
it = v.insert(it + 1, 20);
it = v.insert(it + 1, 30);           // 每次都基于最新迭代器
```

**通俗解释**：vector 像一排连体座位。你往中间塞人时，要么整排座位不够要搬到更大的大厅（扩容，所有人都"换座"→ 全部失效），要么原地把后面的人往后挪一格（插入点之后的人"挪窝"→ 这些位置失效）。拿在手里的"座位号"（旧迭代器）自然就不准了。对策是：先确认大厅够大（reserve），并且每次塞完人都**重新问工作人员要最新的座位号**（用 insert 返回值）。

---

### 第 4 题

> 用erase删除容器中所有等于某个值的元素时，传统写法（循环中直接erase）有什么问题？请说明正确写法。

**答案：**

**传统错误写法**：

```cpp
std::vector<int> v{1,2,2,3,2};
for (auto it = v.begin(); it != v.end(); ++it) {   // ❌
    if (*it == 2) v.erase(it);                     // erase 使 it 失效，然后 ++it 是未定义行为
}
```

**问题**：
- `vector::erase(it)` 会让 `it`（以及其后所有迭代器）失效；失效后仍执行 `++it` 属于**未定义行为（UB）**；
- 即使侥幸不崩，erase 会把后面的元素前移，`++it` 还可能**跳过**一个元素，逻辑也不对。

**正确写法（两种主流）**：

**写法一：使用 erase 的返回值**（适用于任意序列容器）
```cpp
std::vector<int> v{1,2,2,3,2};
auto it = v.begin();
while (it != v.end()) {
    if (*it == 2) it = v.erase(it);   // 用返回值刷新迭代器
    else          ++it;
}
```

**写法二：erase-remove 惯用法（推荐，更高效、语义清晰）**
```cpp
#include <algorithm>
v.erase(std::remove(v.begin(), v.end(), 2), v.end());
```
`remove` 把不等于 2 的元素压缩到前面并返回"新逻辑末尾"，`erase` 再把尾巴真正删掉。因为 `remove` 本身只搬移不删除、**不会使迭代器失效**，这个写法安全且是 O(n) 一次遍历。

**注意**：
- 对 `list`，写法一也可以用（list 的 erase 只使被删元素迭代器失效）；
- 对 `list` 还有专门的 `l.remove(2)` 成员函数，更简洁；
- "删除所有满足条件的元素"则用 `remove_if`（见后文）。

**通俗解释**：错误写法像"排队点名，点到 2 号就让 2 号出列，然后接着往下数"——可 2 号一出列，后面的人全往前补位了，你手里的"名单页码"就全错位了，继续翻页要么翻错人、要么直接翻到不存在的页（UB）。正确做法是：**每次出列后重新拿工作人员发给你的新页码**（erase 返回值），或者干脆**先把所有要留的人集中到队伍前头，再一刀切掉尾巴**（erase-remove 惯用法）。

---

### 第 5 题

> list相比于vector有哪些特殊操作？分别说明reverse、sort、unique、merge、splice的功能和注意事项。

**答案：**

`list` 作为链表，提供了一组**专属于它、且都是 O(1)（splice 等）或专门优化**的成员函数，因为链表改指针比搬元素便宜：

| 操作                                                         | 功能                                                         | 注意事项                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `l.reverse()`                                                | 就地反转链表                                                 | O(n)，只改指针方向，不搬元素；无返回值                       |
| `l.sort()`（可传比较器）                                     | 对链表**就地排序**                                           | **稳定排序**、归并思想、O(n log n)；注意它是**成员函数**而非全局 `std::sort`（全局 sort 要求随机访问迭代器，list 用不了） |
| `l.unique()`（可传比较器）                                   | 删除**相邻**的重复元素（保留第一个）                         | 只删"相邻相同"，所以**一般先 sort 再 unique** 才能去重所有重复值；只对相邻元素比较 |
| `l.merge(l2)`（可传比较器）                                  | 把有序的 `l2` **归并**进有序的 `l`，结果仍有序               | 前提：**两个链表都必须已排序**；`l2` 的元素被搬进 `l`，**归并后 `l2` 变成空**；稳定归并 |
| `l.splice(pos, l2)` / `l.splice(pos, l2, it)` / `l.splice(pos, l2, first, last)` | 把 `l2`（的某段/某个元素）**结点直接转移**到 `l` 的 `pos` 处 | O(1)（转移区间时同链表可能 O(n)）；**不拷贝不搬移元素**，只改指针；**同一链表内 splice 时**，若 `pos` 落在被移动的区间 `[first,last)` 内，行为未定义 |

**代码示例**：

```cpp
std::list<int> l{3,1,2}, l2{0,4};
l.sort();                       // l: {1,2,3}
l2.sort();                      // l2: {0,4}
l.merge(l2);                    // l: {0,1,2,3,4}，l2 变空
l.unique();                     // 删相邻重复
std::list<int> l3{9,8};
auto pos = std::next(l.begin(), 2);
l.splice(pos, l3);              // 把 l3 整段搬到 l 的第2个元素前，l3 变空
```

**通俗解释**：链表节点像"手拉手的人"，这些特殊操作全是"让人重新拉手"（改指针）而不是"把人搬来搬去"（拷元素）——所以 `splice` 是 O(1)、`merge` 不用开新内存。注意事项的本质：**merge 要求双方本来就按顺序站好**，**unique 只认相邻重复**，**同链表的 splice 不能自己拉自己那段**（会乱套，标准直接规定为未定义）。

---

### 第 6 题

> set的基本特征是什么？其底层数据结构是什么？set支持下标操作吗？为什么？

**答案：**

**基本特征**（`std::set<K>`）：
1. **关联式容器**：元素是"关键字（key）"本身，key 就是 value；
2. **元素唯一**：不允许重复 key，重复插入会被忽略；
3. **自动有序**：插入即按 `Compare`（默认 `std::less`，升序）排好序，遍历得到有序序列；
4. **元素不可修改**：`set` 的迭代器是 const 迭代器，不能通过迭代器改元素；
5. **查找/插入/删除都是 O(log n)**。

**底层数据结构**：**红黑树（Red-Black Tree）**——一棵**自平衡的二叉搜索树**，保证最坏情况 O(log n) 的查找/插入/删除，且中序遍历天然有序。libstdc++ 中它由 `_Rb_tree` 实现。

**支持下标操作吗？——不支持。**

原因：
1. `[]` 的语义通常是"按位置/键**取到可修改的引用**，键不存在就插入"。对 `set` 来说：
   - 元素本身就是 key，而 key 一旦修改就会**破坏红黑树的有序性/唯一性**，所以 `set` 的元素被设计为**只读**，`[]` 没法返回可修改引用；
2. `[]` 暗含"**一个 key 对应一个 value**"的映射语义，而 `set` 没有"value"，只有一个 key，下标没有意义；
3. 标准库从设计上就没给 `set` 定义 `operator[]`，用 `s.find(k)` / `s.count(k)` 来查询。

（`map` 才有 `[]`，因为 map 有独立的 value 且 key 用 const 保护、value 可改可插。）

**通俗解释**：`set` 像一间"只进唯一编号牌子的收藏室"，牌子（key）本身就是藏品，而且按编号从小到大摆好。你不可能用"取第 i 号牌子的引用"去改它——改了编号，架子上的顺序就全乱了（破坏红黑树性质），所以牌子一律只读、也不能用下标改。想找牌子就报编号查（`find`，O(log n)）。

---

### 第 7 题

> vector 对象的内存大小是多少？底层 _M_start、_M_finish、_M_end_of_storage 三个指针分别表示什么含义？size() 与 capacity() 是如何通过这三个指针计算出来的？

**答案：**

**1）sizeof(vector) 是多少？**

- 标准**不规定** sizeof，但主流实现（libstdc++/MSVC）里 vector 对象**只保存 3 个指针**（迭代器内部各包一个指针）：
  - 64 位平台：`sizeof(std::vector<int>) == 24` 字节；
  - 32 位平台：`sizeof(...) == 12` 字节。
- 注意：**不管存了多少个元素，sizeof(vector) 都不变**——元素都在堆上，vector 对象只装"管理指针"。这通常也是"把 vector 按值传递很便宜（复制 3 个指针……不，拷贝会深拷贝，按值传还是贵；但 sizeof 本身小）"的原因——不过实际上拷贝 vector 会深拷贝堆内存，这里说的是对象本身尺寸。

**2）三个指针的含义**（libstdc++ 中位于 `_Vector_base`）：

```
低地址                                       高地址
│ 已用元素 │                       │ 预留空间 │
├───────────────────────────────────────┤
_M_start          _M_finish        _M_end_of_storage
```
- `_M_start`：**指向第一个元素**（堆缓冲区起始，`begin()` 底层就是这个）；
- `_M_finish`：**指向最后一个元素之后**（即"逻辑末尾"，`end()` 底层就是它；`_M_finish - _M_start` 得到已用元素个数）；
- `_M_end_of_storage`：**指向整块已分配内存的末尾**（`capacity` 的尽头）。

**3）size 与 capacity 的计算**：

```cpp
size()     = _M_finish - _M_start;              // 已用元素个数
capacity() = _M_end_of_storage - _M_start;      // 缓冲区能容纳的元素个数
```

这两个都是**指针相减**（除以 `sizeof(T)` 即元素个数），O(1)。

**通俗解释**：vector 对象像一张"仓库管理卡"，卡上只写三个地址：仓库第一格（start）、货摆到哪一格（finish）、仓库墙在哪（end_of_storage）。货再多，卡上永远只有这三个地址，所以卡本身永远是 24 字节。已摆货数 = finish 到 start 几格；仓库容量 = 墙到第一格几格。

---

### 第 8 题

> 当插入元素触发扩容时，底层发生了哪些内存与对象生命周期操作（分配新空间->移动/拷贝构造旧元素->析构旧对象->释放旧内存）？

**答案：**

以 `push_back` 触发扩容为例，完整流程（C++11 起，优先移动）：

1. **计算新容量**：`new_cap = max(2 * old_cap, 1)`（libstdc++ 按 2 倍增长；MSVC 约 1.5 倍；标准不规定具体倍数，只要求"几何增长"保证均摊 O(1)）。
2. **分配新内存**：通过 allocator `allocate(new_cap)` 获得一块**未构造**的连续内存（原始内存，不含对象）。
3. **搬移旧元素**：对每个旧元素，在新内存对应位置用 `allocator_traits::construct` / placement new 就地构造：
   - 优先 **移动构造** `T(std::move(*old))`（若 `noexcept`），把资源指针"偷"走；
   - 若 `T` 的移动构造可能抛异常且没标 `noexcept`，标准会退化为**拷贝构造**（保证强异常安全：要么全成功，要么回滚）；
4. **析构旧对象**：对每个旧元素调用 `destroy`（析构函数），**释放对象内部的资源**；
5. **释放旧内存**：`deallocate(old_ptr, old_cap)` 把旧缓冲区还给 allocator；
6. **更新管理指针**：`_M_start/_M_finish/_M_end_of_storage` 指向新内存；然后在新末尾再构造新插入的元素（`push_back` 的新元素也在新缓冲区里构造）。

**关键点**：
- 扩容是 **O(n)**（搬 n 个元素），但**均摊 O(1)**（因为扩容频率随容量增大而指数下降）；
- 扩容后**所有旧迭代器/引用/指针失效**（地址全变了）；
- C++11 后借助**移动语义**，像 `vector<string>` 这类"指针+资源"类型扩容只需搬指针，代价远低于深拷贝；但对 `vector<int>` 这种平凡类型，移动与拷贝等价；
- 想完全避免扩容搬移：`reserve()` 预留容量。

**通俗解释**：扩容像"仓库不够，搬家"。先租一栋更大的空楼（allocate，但楼里没摆家具），然后**把旧家具尽量用"推车整体平移"的方式搬**（移动构造，省得一件件拆装=深拷贝），搬完**把旧楼里的家具架子拆掉**（析构），**退掉旧仓库**（deallocate），最后把门牌号（指针）换成新地址。因为每次翻倍，搬家次数很少，平均下来每次"加货"几乎不花多少时间（均摊 O(1)）。

---

### 第 9 题

> deque 迭代器包含哪 4 个关键指针（_M_cur, _M_first, _M_last, _M_node）？operator++ 和 operator+= 跨越缓冲区（Buffer）边界时，迭代器是如何跳转中控器（Map）节点的？

**答案：**

**deque 迭代器（libstdc++ `_Deque_iterator`）的 4 个成员**：

```
struct _Deque_iterator {
    _Tp*    _M_cur;    // 当前指向的元素
    _Tp*    _M_first;  // 当前缓冲区（buffer）的首地址
    _Tp*    _M_last;   // 当前缓冲区的末地址（最后一个元素之后）
    _Map_pointer _M_node; // 指向中控器 map 中、负责当前缓冲区的那一项（该项保存缓冲区首地址）
};
```

它们的关系：`_M_node` 指向中控器数组中的一个槽位，`*_M_node == _M_first`（该槽存的就是当前缓冲区首地址）；`_M_cur` 在 `[_M_first, _M_last)` 内移动。

**operator++ 跨缓冲区**（简化版）：

```cpp
self& operator++() {
    ++_M_cur;                 // 1. 先向后挪一个元素
    if (_M_cur == _M_last) {  // 2. 若到达当前缓冲区末尾
        _M_set_node(_M_node + 1); // 3. 跳到中控器下一项
        _M_cur = _M_first;        // 4. 定位到新缓冲区的第一个元素
    }
    return *this;
}
void _M_set_node(_Map_pointer new_node) {
    _M_node = new_node;
    _M_first = *new_node;                       // 新缓冲区首地址
    _M_last  = _M_first + _S_buffer_size();     // 新缓冲区末地址
}
```

**operator+= 跨缓冲区**（随机访问核心）：

```cpp
self& operator+=(difference_type n) {
    const difference_type offset = n + (_M_cur - _M_first); // 把“当前缓冲区内偏移”折算成“全局元素数”
    if (offset >= 0 && offset < _S_buffer_size())
        _M_cur += n;                  // 没跨缓冲区，直接走
    else {
        const difference_type node_offset =
            offset > 0 ? offset / _S_buffer_size()
                       : -((-offset - 1) / _S_buffer_size()) - 1; // 算出要跨几个缓冲区
        _M_set_node(_M_node + node_offset); // 跳中控器
        _M_cur = _M_first + (offset - node_offset * _S_buffer_size()); // 缓冲区内偏移
    }
    return *this;
}
```

**通俗解释**：deque 迭代器像一本"跨教室点名册"，册子上记着 4 件事：现在点到谁（cur）、这个教室从谁开始（first）、这个教室到谁结束（last）、"下一个教室在哪"的地址（node）。正常向后点名（++）就往后翻一个名；**翻到本教室末尾**（cur==last）就翻到记录本里登记的下一个教室（`_M_node+1`），并从新教室第一名开始。`+=` 则是"一次跳很多名"：先把"我在本教室第几个 + 要跳几个"算成总步数，算出来要跨几间教室（整除），直接跳到对应教室再走几步。

---

### 第 10 题

> emplace_back 底层是如何利用可变参数模板与完美转发（std::forward），在容器预留内存上通过 placement new 直接就地构造对象的？为什么能避免临时对象的构造与移动/拷贝？

**答案：**

**先看 push_back 的缺点**：
`v.push_back(Point(1,2))` 会先构造一个**临时 Point**，再把它**移动/拷贝**进容器（然后析构临时对象）——存在多余的一次构造+一次搬移。

**emplace_back 的签名**（libstdc++ 风格）：

```cpp
template <class... Args>
reference emplace_back(Args&&... args) {
    if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage) {
        // 有预留空间
        _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                 std::forward<Args>(args)...);   // 就地构造
        ++this->_M_impl._M_finish;
    } else {
        // 需要扩容（扩容后再就地构造）
        _M_realloc_insert(end(), std::forward<Args>(args)...);
    }
    return back();
}
```

**三个关键机制**：

1. **可变参数模板 `Args&&...`**：可以接收任意数量、任意类型的实参，把 `Point` 的构造函数参数直接"原封不动"接进来；
2. **完美转发 `std::forward<Args>(args)...`**：保持每个实参的"左值/右值"属性不变地继续往下传（实参是右值就保持右值，左值就保持左值），使 `Point` 的构造函数能用正确的重载（右值 → 移动/直接绑定）；
3. **placement new 就地构造**：`allocator_traits::construct` 底层就是 `::new((void*)ptr) T(std::forward<Args>(args)...)`——在**容器已经分配好的内存地址上直接调用构造函数**生成对象，而不是先造临时对象再拷贝。

**为什么能避免临时对象**：
- 因为参数是**直接传给目标构造函数**的，在目标内存上一步到位构造出最终对象；
- **不需要**先构造临时对象、再移动/拷贝、再析构临时——省掉了"临时对象构造 + 移动/拷贝构造 + 临时对象析构"整条链；
- 特别对**不可移动/不可拷贝**的类型（如带 `std::mutex` 的类），`emplace_back` 是唯一能进容器的办法；对 `vector<vector<int>>` 这类"构造参数就是另一个容器"的场景，`emplace_back(n, 0)` 也能省去中间临时。

**注意**：若元素是 `int` 这种平凡类型，`push_back` 和 `emplace_back` 没有性能差异；`emplace_back` 的真正收益在**构造成本高 / 不可拷贝移动**的对象上。另外 C++17 后也有 `std::vector::insert`/`emplace` 系列同理。

**通俗解释**：`push_back` 像"先在门口搭好一件样品，再把它搬进屋里摆好，最后把门口样品拆掉"；`emplace_back` 像"直接把材料拿到屋里，在屋里现搭好"——材料（参数）直接进场，成品就地成型，省掉一趟搬家和一次拆装。

---

## 二、编程题

### 编程题 1

> 给定一个vector<int>，删除其中所有连续重复的元素（例如{1,2,2,3,3,3,4}→{1,2,3,4}），请使用迭代器正确实现。

**答案：**

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v{1, 2, 2, 3, 3, 3, 4};

    if (v.empty()) return 0;

    // 思路：保留不重复的，覆盖写回前面
    auto write = v.begin();          // 下一个写入位置
    auto read  = v.begin();          // 扫描位置
    while (read != v.end()) {
        // 保留 read 指向的元素（它是某个连续块的开头）
        if (write != v.begin() && *write == *read) {
            // 与上一个保留值相同 → 跳过（不写）
        } else {
            *write = *read;          // 覆盖写入（原地）
            ++write;
        }
        ++read;
    }
    v.erase(write, v.end());         // 用返回值/区间删除真正删掉尾巴

    for (int x : v) std::cout << x << " ";   // 1 2 3 4
    return 0;
}
```

**标准库一行版**：

```cpp
#include <algorithm>
v.erase(std::unique(v.begin(), v.end()), v.end());
```
`std::unique` 删除**相邻**重复（保留每段第一个），返回新的逻辑末尾，再 `erase` 删除尾巴——这正是教材惯用法，安全且无迭代器失效问题（`unique` 不使迭代器失效）。

**通俗解释**：两个版本都是"双指针清扫"：一个读指针往前扫，一个写指针负责把"每段第一个值"留下并向前排；遇到与上一个留下的值相同就跳过；扫完把尾巴一刀切掉。`std::unique` 就是把这个逻辑封装好的标准库函数。

---

### 编程题 2

> 创建两个list<int>，使用merge操作将它们合并为一个有序链表，并观察原链表的变化。

**答案：**

```cpp
#include <iostream>
#include <list>

template <typename C>
void show(const char *name, const C &c) {
    std::cout << name << ": ";
    for (int x : c) std::cout << x << " ";
    std::cout << "(size=" << c.size() << ")\n";
}

int main() {
    std::list<int> a{1, 3, 5, 7};
    std::list<int> b{2, 4, 6, 8};

    show("merge 前 a", a);
    show("merge 前 b", b);

    a.merge(b);    // 要求 a、b 都已有序；结果进 a，b 被清空

    show("merge 后 a", a);   // 1 2 3 4 5 6 7 8 (size=8)
    show("merge 后 b", b);   // (size=0)  —— 关键观察点
    return 0;
}
```

**行为观察**：
1. `a.merge(b)` **前置条件**：a 和 b 都必须**已经有序**（这里双方各自升序）；
2. 归并是**稳定**的，相等元素保持相对顺序；
3. **不拷贝元素、不分配内存**：只是把 b 的结点**重新链接**进 a（改指针），所以是 O(a+b)；
4. **关键观察**：merge 后 **b 变为空链表**（size=0）——b 的结点全被"搬走"了。

**通俗解释**：merge 像两队**本来各自排好队**的人，现在按顺序重新串成一条队。操作只是"重新拉手"（改指针），不新增人也不拷贝人，所以 b 队搬完后自然就空了。前提是两队事先都得排好序，否则合并结果不保证有序。

---

### 编程题 3

> 使用splice操作将一个链表中的某个元素移动到另一个链表的指定位置，并演示同一链表中移动元素（注意安全性）。

**答案：**

```cpp
#include <iostream>
#include <list>

template <typename C>
void show(const char *name, const C &c) {
    std::cout << name << ": ";
    for (int x : c) std::cout << x << " ";
    std::cout << "\n";
}

int main() {
    // —— 跨链表：把 b 中的某个元素移到 a 的指定位置 ——
    std::list<int> a{1, 2, 5};
    std::list<int> b{10, 20, 30};
    auto pos = std::next(a.begin(), 2);          // a 中位置：指向 5
    auto it  = std::next(b.begin(), 1);          // b 中指向 20
    a.splice(pos, b, it);                        // 把 b 中的 20 移到 a 的 5 之前
    show("a", a);   // 1 2 20 5
    show("b", b);   // 10 30   （20 被搬走）

    // —— 同一链表内移动：把 1 从开头移到末尾 ——
    std::list<int> c{1, 2, 3, 4};
    auto src = c.begin();                        // 指向 1
    auto dst = c.end();                          // 末尾
    c.splice(dst, c, src);                       // 同一链表移动
    show("c", c);   // 2 3 4 1

    // —— 安全性演示：pos 不能落在被移动区间 [first,last) 内，否则未定义 ——
    // 下面的写法是安全的（把整段搬到自身末尾之前的末尾，不重叠）
    std::list<int> d{1, 2, 3, 4, 5};
    auto first = d.begin(); auto last = std::next(d.begin(), 3); // [1,2,3)
    d.splice(d.end(), d, first, last);           // 把 [1,2,3) 移到末尾
    show("d", d);   // 4 5 1 2 3
    return 0;
}
```

**要点与安全说明**：

1. `splice(pos, other, it)`：把 `other` 中的结点 `it` **转移**到 `*this` 的 `pos` 之前；
2. **O(1)**：只改指针，不拷贝/搬移元素；被搬元素所在列表自动删除该结点；
3. **同一链表 splice**：允许（上面把 `c` 的 `1` 移到末尾）；
4. **安全性（标准规定）**：若在**同一容器**内做区间 `splice`，且 `pos` 是**被移动区间 `[first, last)` 内的迭代器**，行为**未定义**——所以移动区间时，目标位置必须放在区间之外（如上面的 `d.end()`）；
5. 跨链表 splice 无此限制，但要保证 `pos` 不在源链表里（除非是同一链表操作）。

**通俗解释**：splice 就是"把人从一队里拽出来，插到另一队/同队指定位置，只改握手顺序"。因为不搬行李（不拷贝元素），所以 O(1)。安全规则记住一句：**同一队里，你不能把自己正在拉的那段"往自己怀里塞"**——目标位置落在被拉走的那段中间会乱套（未定义）。

---

# 第三套：set / map

## 一、问答题

### 第 1 题

> 当set的Key为自定义类型时，需要提供哪些比较方式？请列举三种，并说明各自的实现细节。

**答案：**

`set` 需要 `Compare` 提供**严格弱序（strict weak ordering）**，即 `comp(a,b)` 为 true 表示 a 排在 b 前。自定义类型作 Key，三种提供方式：

**方式一：模板特化 `std::less<Key>`**（默认比较器）

```cpp
namespace std {
    template <>
    struct less<Point> {
        bool operator()(const Point &a, const Point &b) const {
            return a.x < b.x || (a.x == b.x && a.y < b.y);  // 先 x 后 y
        }
    };
}
std::set<Point> s;   // 直接用默认模板参数 std::less<Point>
```

**方式二：重载 `operator<`**（作为自由函数或成员）

```cpp
struct Point {
    int x, y;
    friend bool operator<(const Point &a, const Point &b) {
        return a.x < b.x || (a.x == b.x && a.y < b.y);
    }
};
std::set<Point> s;   // 默认 std::less<Point> 内部用 a < b
```

**方式三：自定义函数对象（作为模板参数传入）**

```cpp
struct PointCmp {
    bool operator()(const Point &a, const Point &b) const {
        return a.x < b.x || (a.x == b.x && a.y < b.y);
    }
};
std::set<Point, PointCmp> s;
```

**实现细节/注意事项**：
- 无论哪种方式，核心都是实现**严格弱序**：要求 `comp(a,b)==false && comp(b,a)==false` 时认为两者"等价"（去重依据是"既不小于也不大于"，不是 `==`）；
- **必须严格弱序**：自反性 `comp(a,a)==false`、非对称、传递性。用 `a.x < b.x || (a.x == b.x && a.y < b.y)` 这种**字典序（先 x 后 y）**写法最稳妥；
- 比较逻辑必须**与 `operator==` 一致地定义"等价"**，否则插入/查找会错乱（比如两个对象 `==` 相等但 `comp` 认为不等，就会出现"重复元素"）；
- C++20 还可用 `operator<=>`（三路比较）+ 默认生成，配合 `std::set<Point>`（前提是用 `std::less` 默认比较，`<=>` 的语义正好满足）——这是新标准更推荐的做法：
  ```cpp
  struct Point { int x, y; auto operator<=>(const Point&) const = default; };
  ```

**通俗解释**：set 的红黑树要"比大小"才能把元素放对位置。三种方式本质都是告诉它"怎么比"：① 篡改官方默认裁判（特化 less），② 让元素自己会报大小（重载 <），③ 换一个裁判（自定义函数对象）。只要比法满足"排他、传递、不会既小于又大于"，树就建得对。

---

### 第 2 题

> multiset与set的区别是什么？multiset支持哪些额外的边界操作（lower_bound、upper_bound、equal_range）？

**答案：**

**multiset 与 set 的区别**：

|                | `set`        | `multiset` |
| -------------- | ------------ | ---------- |
| 允许重复 key   | ❌ 唯一       | ✅ 可重复   |
| `insert(k)`    | 已存在则忽略 | 总是插入   |
| 底层           | 红黑树       | 红黑树     |
| `count(k)`     | 0 或 1       | 可以 >1    |
| 查找/插入/删除 | O(log n)     | O(log n)   |

其余特征相同：自动有序、元素只读、双向迭代器。

**multiset 的边界操作（set 同样有）**：

- `lower_bound(k)`：返回指向**第一个 ≥ k** 的元素的迭代器（所有 `< k` 的元素之后）；
- `upper_bound(k)`：返回指向**第一个 > k** 的元素的迭代器（所有 `== k` 的元素之后）；
- `equal_range(k)`：返回 `pair<lower_bound(k), upper_bound(k)>`，即**所有等于 k 的元素区间 `[first, last)`**。

```cpp
std::multiset<int> ms{1, 3, 3, 3, 5, 7};
auto lo = ms.lower_bound(3);   // 指向第一个 3
auto up = ms.upper_bound(3);   // 指向 5
for (auto it = lo; it != up; ++it) std::cout << *it << " ";   // 3 3 3
auto [b, e] = ms.equal_range(3);   // C++17 结构化绑定，效果同上
```

**通俗解释**：set 像"每人只有一张"的证件柜，multiset 像"同一个人可以占多格"的柜子（比如统计分数分布）。lower_bound/upper_bound/equal_range 是"划定同一批人的范围"的三把尺子：下界尺找到**第一次出现**的位置，上界尺找到**最后一次出现再往后一格**的位置，equal_range 直接给你一整段。

---

### 第 3 题

> map的底层结构是什么？map中的key是否允许重复？value呢？

**答案：**

**底层结构**：`std::map` 底层是一棵**红黑树**（`_Rb_tree`），每个节点存一个 `pair<const Key, T>`（key 与 value 的键值对）。因此 map 的 key 自动有序，查找/插入/删除 O(log n)。

**key 是否允许重复？——不允许。**
- map 的 key **唯一**：插入已存在的 key 不会覆盖原值（`insert` 失败并返回指向旧元素的迭代器）；
- key 还是 **const 的**：`pair<const Key, T>`，不能通过迭代器修改 key（保证红黑树有序性不被破坏）。

**value 是否允许重复？——允许。**
- value 只是"挂在 key 下的数据"，**没有唯一性约束**，多个不同 key 可以挂相同 value；
- value 是可修改的：`m[k] = ...`、`m.at(k) = ...` 都可以改。

**区分**：若想 key 可重复，用 `std::multimap`（key 允许多个，且不支持 `[]`）。

**通俗解释**：map 像一本**按字母排好的电话簿**，每个名字（key）只占一条，不能重名（key 唯一且是"只读标头"），但同一个人可以对应多个电话号码（value 可重复、可修改）。因为书页按名字排序，查名字（key）是 O(log n)。

---

### 第 4 题

> map的operator[]有哪些功能？为什么说它既可用于查找也可用于插入和修改？它是否有const版本？

**答案：**

**`operator[]` 的功能**：`T& operator[](const Key& k)`（以及右值 key 版本 `T& operator[](Key&&)`）——

1. **若 key 存在**：返回该 key 对应的 **value 的引用**，可读可改（查找 + 修改）；
2. **若 key 不存在**：**自动插入一个"值初始化"的键值对** `pair(k, T{})`（value 用默认构造/值初始化，如 int→0、string→""），然后返回其引用（插入 + 初始化）。

```cpp
std::map<std::string, int> m;
m["apple"];                     // 不存在 → 插入 {"apple", 0}
m["apple"] = 5;                 // 存在 → 修改为 5
int n = m["apple"];             // 查找（读取引用）
```

**为什么"既查找又可插入和修改"**：因为 `[]` 的语义是"**确保有这个键，并给我它的值引用**"——找不到就补插一个默认值，找得到就直接给你可写引用。查改插三合一。

**有没有 const 版本？——没有。**
- 标准**不提供** `const` 版本的 `operator[]`。原因：`[]` 在"键不存在"时会**插入元素**，这会改变容器，const 容器不允许；
- 而且即使键存在，`[]` 返回的是**可写引用**，这也不符合 const 语义；
- **const 场景的正确查找姿势**：
  - `m.find(k)` → 返回迭代器（`== end()` 表示没有）；
  - `m.at(k)`（C++11 起）→ 返回 value 引用，**key 不存在时抛 `std::out_of_range`**（有 const 版本）；
  - `m.count(k)` → 0/1。

```cpp
const std::map<std::string,int> cm{{"a",1}};
// cm["a"];           // ❌ 编译错：无 const operator[]
int x = cm.at("a");   // ✅ 1
auto it = cm.find("a");
if (it != cm.end()) x = it->second;
```

**通俗解释**：`[]` 像"点单"：菜单上有就上菜并把盘子递给你（可改），没有就**默认给你来一份标准菜**并记账（插入）。所以它既能查又能增改。但"只许看不许动"的菜单（const map）没有这个服务——因为你一 `[]` 就可能私自加菜；想看菜色得用 `find`（找不到不报错）或 `at`（找不到会吼一声，抛异常）。

---

### 第 5 题

> multimap为什么不提供下标访问运算符？

**答案：**

`std::multimap` **不提供 `operator[]`**，根本原因是它允许 **key 重复**，而 `[]` 的语义与"一对多"冲突：

1. **`[]` 要求"一个 key 确定一个 value"**：`m[k]` 返回 `T&`——但这个 key 在 multimap 里可能对应**多个 value**，无法确定返回哪一个；
2. **`[]` 的"不存在就插入"语义**：如果按 `[]` 的惯例，key 不存在时插入一个默认值——可 multimap 允许重复，下次再 `[]` 同一个 key 又该插入第二个默认值吗？语义无法自洽；
3. **无法提供可写引用**：既然一个 key 对应多个 value，"改 value"也无从谈起（改哪一个？）。

因此 multimap 只提供**显式接口**：
- `insert({k, v})`：插入一对（可重复插同一 key）；
- `count(k)`、`find(k)`、`lower_bound(k)`、`upper_bound(k)`、`equal_range(k)`：查找某 key 的所有 value 区间；
- `erase(k)`：删除该 key 的所有项。

```cpp
std::multimap<std::string,int> mm;
mm.insert({"Alice", 90});
mm.insert({"Alice", 85});
for (auto it = mm.lower_bound("Alice"); it != mm.upper_bound("Alice"); ++it)
    std::cout << it->second << " ";    // 90 85
```

**通俗解释**：`[]` 像"报名字取一个号码"，这要求**名字唯一对应一个号码**。multimap 允许同名多条记录，报一个名字根本不知道取哪一条，所以直接不提供这个功能；想拿数据就用 `equal_range` 把**同名所有记录**整段端走。

---

## 二、编程题

### 编程题 1

> 定义一个Point类（含x,y坐标和距离原点的距离），将其作为set的Key，分别使用模板特化std::less<Point>、重载operator<、自定义函数对象三种方式实现排序，并测试插入和查找。

**答案：**

```cpp
#include <iostream>
#include <set>
#include <cmath>

struct Point {
    int x, y;
    double dist() const { return std::sqrt(double(x)*x + double(y)*y); }
    // 供“重载 operator<”方式使用（方式二）
    friend bool operator<(const Point &a, const Point &b) {
        if (a.dist() != b.dist()) return a.dist() < b.dist();  // 先按距离
        if (a.x != b.x) return a.x < b.x;                       // 再按 x
        return a.y < b.y;                                       // 最后按 y
    }
};

// 方式一：模板特化 std::less<Point>（注意要定义在 namespace std 里）
namespace std {
    template <>
    struct less<Point> {
        bool operator()(const Point &a, const Point &b) const {
            return std::tie(a.dist(), a.x, a.y) < std::tie(b.dist(), b.x, b.y);
        }
    };
}

// 方式三：自定义函数对象
struct PointDistLess {
    bool operator()(const Point &a, const Point &b) const {
        return std::tie(a.dist(), a.x, a.y) < std::tie(b.dist(), b.x, b.y);
    }
};

template <typename Set>
void test(const char *name, Set &s) {
    s.insert(Point{3, 4});   // 距离 5
    s.insert(Point{1, 1});   // 距离 √2
    s.insert(Point{0, 1});   // 距离 1
    std::cout << name << " 有序输出: ";
    for (auto &p : s) std::cout << "(" << p.x << "," << p.y << ")[" << p.dist() << "] ";
    std::cout << "\n查找 (3,4): " << (s.find(Point{3,4}) != s.end() ? "找到" : "未找到") << "\n\n";
}

int main() {
    std::set<Point>                     s1;   // 用默认 std::less<Point>（方式一特化生效）
    std::set<Point, PointDistLess>      s2;   // 方式三
    // 方式二已经体现在 operator< 中（s1 与 s2 都可验证），再显式建一个用 operator< 的：
    std::set<Point, std::less<Point>>   s3;   // 实际与 s1 同（特化 less 走 operator< 之外的版本）
    test("方式一(特化less)/方式二(operator<)", s1);
    test("方式三(函数对象)", s2);
    return 0;
}
```

**要点**：
- 三种方式必须保证**同样的严格弱序规则**，否则不同 set 的排序不一致；
- `std::tie(a,b,c) < std::tie(...)` 是 C++11 里做**字典序比较**的优雅写法（等价于手写 `if(a!=...)...`）；
- 排序规则建议：**距离 → x → y**，保证"等价判定"明确（距离相同再看坐标），查找 `find(Point{3,4})` 时能用同一规则定位。

**通俗解释**：给 Point 定排序就是给它定一条"谁站前面"的规则。三种方式只是"把规则写在哪儿"不同：写在标准库的 `std::less` 专版（方式一）、写在 Point 自己的 `<`（方式二）、写在一个独立裁判类里（方式三）。规则一致，树就一致，find 才能按同一套规则找到人。

---

### 编程题 2

> 统计一段文本中每个单词出现的次数（忽略大小写），使用map存储单词和次数，并输出按字母顺序排列的结果。

**答案：**

```cpp
#include <iostream>
#include <sstream>
#include <map>
#include <string>
#include <algorithm>
#include <cctype>

int main() {
    std::string text = "The quick brown fox jumps over the lazy dog. "
                       "The DOG is quick!";
    std::map<std::string, int> freq;

    std::stringstream ss(text);
    std::string word;
    while (ss >> word) {
        // 去掉标点，并转小写
        std::string clean;
        for (char c : word)
            if (std::isalnum(static_cast<unsigned char>(c)))
                clean.push_back(std::tolower(static_cast<unsigned char>(c)));
        if (!clean.empty())
            ++freq[clean];          // map::operator[] 不存在则插入 0 再自增
    }

    // map 按键（字母序）自动有序，直接遍历
    for (const auto &[w, n] : freq)          // C++17 结构化绑定
        std::cout << w << " : " << n << "\n";
    return 0;
}
```

输出（片段）：`dog : 2`、`fox : 1`、`the : 2`……

**要点**：
- **忽略大小写**：先逐字符 `tolower` 归一化再计数；
- **去标点**：用 `isalnum` 过滤，避免 `dog.` 与 `dog` 算两个词；
- **map 自动有序**：`map<string,int>` 的 key（string）按字母序排列，遍历即得字母序结果；
- `++freq[clean]`：`[]` 找不到键会插入 `{clean, 0}` 再自增，正好"统计+插入"一步到位。

**通俗解释**：把整段文字拆成一个一个词，每遇到一个词就翻到电话簿那一页（key），把那页的"记数牌"加 1；电话簿按字母自动排好，最后顺着翻一遍就是字母序的统计表。大小写问题靠先"全部转小写、去掉标点"来归一。

---

### 编程题 3

> 使用multimap存储学生姓名（key）和多个课程成绩（value），实现按姓名查找所有成绩的功能。

**答案：**

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::multimap<std::string, int> scores;
    scores.insert({"Alice", 90});
    scores.insert({"Alice", 85});
    scores.insert({"Alice", 95});
    scores.insert({"Bob", 70});
    scores.insert({"Bob", 88});

    std::string name = "Alice";
    auto [beg, end] = scores.equal_range(name);   // C++17 结构化绑定

    std::cout << name << " 的所有成绩: ";
    int sum = 0, cnt = 0;
    for (auto it = beg; it != end; ++it) {
        std::cout << it->second << " ";
        sum += it->second; ++cnt;
    }
    std::cout << "\n平均分: " << (cnt ? double(sum)/cnt : 0.0) << "\n";
    return 0;
}
```

输出：`Alice 的所有成绩: 90 85 95`、`平均分: 90`。

**要点**：
- `equal_range(name)` 返回 `pair<iterator, iterator>`，即"该 key 所有记录"的区间；
- 因为 `multimap` 对相同 key 的值在底层红黑树里**相邻存放**，equal_range 才能给出连续区间；
- 若 name 不存在，`beg == end`，循环自然不执行；
- 查找某 key 的**第一条**用 `find`，**个数**用 `count`。

**通俗解释**：multimap 像"同名多卡"的成绩册，一个学生名下可以挂好几条成绩记录，而且同名记录在树里排在一起。`equal_range` 就是"把这一堆同名记录从第一张到最后一张整段端出来"的筐。

---

# 第四套：哈希与无序容器、priority_queue

## 一、问答题

### 第 1 题

> 什么是哈希函数？常见的哈希函数构造方法有哪些？

**答案：**

**哈希函数**：把**任意大小的数据（key）**映射到**固定范围的下标（bucket index，即桶号）**的映射函数 `h(k) -> [0, bucket_count)`。理想的哈希函数应当：计算快、分布均匀（不同 key 尽量散开，减少冲突）、对相似输入（如 "abc"/"abd"）结果差异大。

**常见构造方法**：

1. **除留余数法（Division Method）**：`h(k) = k mod m`，m 通常取素数、避开 2 的幂（减少低位聚集）。最简单、最常用。
2. **乘法散列法（Multiplication Method）**：`h(k) = floor(m * frac(k * A))`，A 取黄金分割常数 `(√5-1)/2 ≈ 0.618`。对 m 的选择不敏感。
3. **平方取中法（Mid-Square）**：`h(k) = 取 k² 的中间几位`。
4. **数字分析法（Digit Analysis）**：分析 key 各位数字分布，选取分布均匀的几位作为哈希值（适合已知 key 集）。
5. **折叠法（Folding）**：把 key 分成若干段再求和（或异或）取模。
6. **随机数法 / 位运算混合（常用于字符串与复合类型）**：如 `BKDRHash`、`MurmurHash`、`FNV-1a`，以及组合多个成员时的 `hash_combine`（见后文编程题）——以位运算（移位、异或、乘以大质数）把各位信息充分"搅拌"。

**补充**：C++ 标准库自带 `std::hash` 对整数、浮点、`std::string`、指针等提供专门实现，一般无需自己造。

**通俗解释**：哈希函数像"把学号（key）换算成储物柜编号（桶号）"的算法。好的算法让每个学号尽量均匀地落到不同柜子，这样开柜取物才快；算法不好，很多学号挤到同一个柜子（冲突），取物就要排队翻找。

---

### 第 2 题

> 什么是哈希冲突？解决哈希冲突的常用方法有哪些？STL无序容器采用哪种？

**答案：**

**哈希冲突**：不同的 key 经哈希函数映射到**同一个桶**，即 `h(k1) == h(k2)` 且 `k1 != k2`。因为桶数有限而 key 无限，冲突不可避免，只能缓解。

**常用解决方法**：

1. **链地址法（Chaining，也称拉链法/分离链接法）**：每个桶挂一个链表/红黑树，冲突的元素全部挂进同一个桶的链里。查询时先算桶号再在链里线性找（均摊 O(1)）。
2. **开放定址法（Open Addressing）**：冲突时在桶数组里**寻找下一个空位**。常见探测序列：
   - 线性探测（+1、+2、+3…）；
   - 二次探测（+1²、+2²…）；
   - 双重散列（用第二个哈希函数决定步长）。
3. **再哈希法（Rehashing）**：准备多个哈希函数，冲突就换一个再算。
4. **建立公共溢出区**：冲突的元素放入一个公共溢出表。

**STL 无序容器采用哪种？——链地址法（拉链法）。**
- `unordered_set/unordered_map/unordered_multiset/unordered_multimap` 的每个桶（bucket）是一个**单链表**（libstdc++ 实现为 `_Hashtable`，桶里挂链表；MSVC 类似）；
- 元素个数超过 `bucket_count * max_load_factor` 时触发**整体再哈希（rehash）**：扩桶、重算、重新挂链。
- （C++ 标准只要求"均摊 O(1)"，没有强制具体冲突方案，但主流实现都是拉链法。）

**通俗解释**：哈希冲突像"两个学号算出来是同一个储物柜号"。拉链法就是**在柜子里多放一层抽屉**，撞号的都塞进同一柜子的不同抽屉；开放定址法是"这个柜子被占了就找隔壁空柜子"。STL 用的是前者：柜子里挂一条链，抽屉挨个排。

---

### 第 3 题

> 装载因子是什么？它对哈希表的性能有何影响？一般控制在什么范围？

**答案：**

**装载因子（Load Factor）**：`α = 已存元素个数 n / 桶数 bucket_count`，即"每个桶平均装多少元素"。

**对性能的影响**：
- **α 越大**：平均链长（或探测次数）越大，冲突越多，查找/插入/删除退化（拉链法下近似 O(1+α)）；当 α 接近桶数级别时性能塌方；
- **α 越小**：桶越多、内存浪费越大（桶数组本身占内存），但冲突少、速度快；
- 是"**时间 vs 空间**"的权衡：α 大省内存费时间，α 小费内存省时间。

**一般控制范围**：
- STL 里 `max_load_factor()` 默认 **1.0**（元素数 ≤ 桶数时触发 rehash）；
- 实践中链地址法通常把 α 控制在 **0.5 ~ 1.0** 之间：
  - 追求速度（查多插少）取 0.5~0.75；
  - 追求内存节省可取到 1.0 甚至略高（但不建议 >1，链会变长）。
- 可用 `unordered_map::reserve(n)` 预置桶数（按 `n / max_load_factor` 预留）避免频繁 rehash。

**通俗解释**：装载因子就是"平均每个储物柜里塞了多少东西"。塞得越满（α 大），开柜后还要翻更多抽屉（慢）；柜子越多（α 小），地方宽敞但柜子也占地方（费内存）。STL 默认控制在"平均每柜 1 件"附近，是速度和空间的平衡点。

---

### 第 4 题

> unordered_set与set的主要区别是什么？在什么场景下优先选择unordered_set？

**答案：**

| 维度                         | `set`                         | `unordered_set`                        |
| ---------------------------- | ----------------------------- | -------------------------------------- |
| 底层                         | 红黑树（自平衡 BST）          | 哈希表（桶 + 链表）                    |
| 元素是否有序                 | ✅ 自动有序                    | ❌ 无序（遍历顺序不确定）               |
| 查找/插入/删除               | O(log n)                      | 平均 O(1)，最坏 O(n)（大量冲突时）     |
| 迭代器                       | 双向迭代器                    | 正向迭代器（且无 begin/end 的有序性）  |
| Key 要求                     | 提供**比较器**（`<`/Compare） | 提供**哈希函数 + 相等比较**（hash/==） |
| `lower_bound` 等有序边界操作 | ✅                             | ❌                                      |
| 内存                         | 树节点                        | 桶数组 + 节点                          |

**优先选择 unordered_set 的场景**：
1. **不需要有序遍历**（只要"查重/在不在"）；
2. **查找频率高、数量大**，想要平均 O(1) 的查找（比如 100 万级元素的成员判断，unordered 通常明显快于 set，除非数据量小到缓存差异掩盖）；
3. 需要**常数时间插入删除**且不在乎顺序。

**仍应选 set 的场景**：需要有序输出、需要 `lower_bound/equal_range` 等边界操作、需要顺序相关的遍历/区间查询、或对最坏时延敏感（哈希冲突可导致 O(n)）的实时系统。

**通俗解释**：set 像"按字母排好的档案柜"，查找要按字母逐层比较（O(log n)）；unordered_set 像"按编号直接开柜"（算哈希 O(1)），拿取飞快但柜子之间没有顺序。你只关心"在不在、几个"时用 unordered 更快；要按顺序打印、查区间时就得上 set。

---

### 第 5 题

> 当unordered_set的Key为自定义类型时，需要提供哪两个模板参数的自定义实现？请说明每种实现的三种方式。

**答案：**

`unordered_set<Key>` 实际是 `unordered_set<Key, Hash, KeyEqual, Allocator>`，自定义类型需要提供两个东西：

**① 哈希函数（Hash）**——把 Key 变成 `size_t`；
**② 相等比较（KeyEqual）**——判断两个 Key 是否相等（默认 `std::equal_to<Key>`，即用 `==`）。

**实现方式各三种：**

**Hash 的三种方式**：

1. **模板特化 `std::hash<Key>`**：
   ```cpp
   namespace std {
       template <> struct hash<Person> {
           size_t operator()(const Person &p) const {
               return std::hash<std::string>{}(p.name) ^
                      (std::hash<int>{}(p.age) << 1);
           }
       };
   }
   ```
2. **自定义函数对象**（作为模板实参传入）：
   ```cpp
   struct PersonHash {
       size_t operator()(const Person &p) const { /* 同上 */ }
   };
   std::unordered_set<Person, PersonHash> s;
   ```
3. **lambda**（作为模板实参，C++11 起）：
   ```cpp
   auto h = [](const Person &p) { return /*...*/; };
   std::unordered_set<Person, decltype(h)> s(0, h);
   ```

**KeyEqual 的三种方式**：

1. **为类型重载 `operator==`**（默认 `std::equal_to<Key>` 走它）：
   ```cpp
   struct Person {
       std::string name; int age;
       bool operator==(const Person &o) const {
           return name == o.name && age == o.age;
       }
   };
   ```
2. **模板特化 `std::equal_to<Person>`**：
   ```cpp
   namespace std {
       template <> struct equal_to<Person> {
           bool operator()(const Person &a, const Person &b) const {
               return a.name == b.name && a.age == b.age;
           }
       };
   }
   ```
3. **自定义函数对象 / lambda** 作为 `KeyEqual` 模板实参。

**注意**：哈希相等与 `==` 必须**一致**——`a == b` 的两个对象必须哈希值也相同（允许不同对象同哈希，那叫冲突），否则"同 key 查不到"。

**通俗解释**：哈希表要开柜子，需要两样：①"算柜号"的规则（Hash），②"确认柜里东西是不是同一件"的规则（KeyEqual，通常是 `==`）。每种规则都能通过"改标准版 / 自备道具 / 现场写 lambda"三种方式提供。关键红线：**两件东西算相等，柜号必须一样**，否则同一件东西可能被塞进两个柜子。

---

### 第 6 题

> 如何根据实际需求选择合适的容器？从有序性、下标支持、查找时间复杂度、迭代器类型四个方面分析。

**答案：**

**① 有序性（是否需要按序遍历/区间查询）**
- 需要自动有序、范围查询（`lower_bound` 等）→ `set/map/multiset/multimap`（红黑树）；
- 不要求顺序、只求快 → `unordered_*`；
- 顺序按插入维护（不需要排序）→ `vector/deque/list`。

**② 下标支持（能否 `c[i]`）**
- 需要 O(1) 按下标访问 + 连续内存 → `vector`；
- 需要头尾都 O(1) 插入删除且支持下标 → `deque`；
- 不需要下标（只按迭代器/值访问）→ `list/forward_list/set/...`；
- 关联容器里只有 `map/unordered_map` 提供 `[]`（key→value 下标）。

**③ 查找时间复杂度**
- 需要按 key 平均 O(1) 查找 → `unordered_set/unordered_map`；
- 需要按 key O(log n) 且有序 → `set/map`；
- 需要按值线性查找 → 序列容器（vector 可用 `std::find` O(n)）；
- 内存/最坏时延敏感 → 红黑树（O(log n) 有界）比哈希（最坏 O(n)）更可控。

**④ 迭代器类型**
- 随机访问（`it+n`、`it1-it2`、`[]`）→ `vector/deque/array/string`（能用全局 `std::sort`）；
- 双向 → `list/set/map/...`（能用 `reverse`）；
- 正向 → `forward_list/unordered_*`；
- 输入/输出迭代器 → 流迭代器（`istream_iterator/ostream_iterator`）与插入迭代器。

**综合决策速查**（常见组合）：
- "频繁随机访问 + 尾部增删" → `vector`；
- "头尾都要增删 + 需要下标" → `deque`；
- "中间频繁插入删除" → `list`；
- "按 key 查、要排序" → `map/set`；
- "按 key 查、不排序、量大求快" → `unordered_map/unordered_set`；
- "FIFO/LIFO/按优先级" → `queue/stack/priority_queue`（容器适配器）。

**通俗解释**：选容器像选仓库，先问四个问题：货要不要排好序？（树 vs 哈希）；要不要按编号 O(1) 取货？（vector/deque）；取货快不快？（O(1)/O(log n)/O(n)）；搬运工具（迭代器）能不能跳着走？——四个答案一凑，容器基本就定了。

---

### 第 7 题

> priority_queue的底层数据结构是什么？默认是大根堆还是小根堆？如何改变比较方式？

**答案：**

**底层数据结构**：`priority_queue` 是**容器适配器**，默认底层容器是 `std::vector`，在其上利用**堆（heap）** 算法（`std::make_heap/push_heap/pop_heap`）维护一个二叉堆（通常为数组式大根堆/小根堆）。

- 声明：`template<class T, class Container = vector<T>, class Compare = less<Container::value_type>>`；
- 三个模板参数：元素类型、底层容器（须支持随机访问 + `front/push_back/pop_back`）、比较器。

**默认是大根堆还是小根堆？——大根堆。**
- 默认比较器 `std::less<T>`（"a < b"），而 priority_queue 的约定是：**`top()` 返回"按 Compare 比较最大的那个"**；
- 所以默认 `top()` 是**最大值**，即大根堆。

**如何改变**：
1. **换成 `std::greater<T>` 得到小根堆**（`top()` 是最小值）：
   ```cpp
   std::priority_queue<int, std::vector<int>, std::greater<int>> pq;   // 小根堆
   ```
2. 自定义类型：提供 `operator<`（默认 less 可用）或自定义比较器（见下一题）；
3. 也可换底层容器（如 `std::deque<int>`），但必须支持随机访问迭代器。

**注意陷阱**：比较器是"**优先级判定**"而非"排列顺序"——`less` 让大的先出，`greater` 让小的先出，正好和直觉相反，初学者常写反。

**通俗解释**：priority_queue 像"插队队列"：vector 是排队场地，堆算法保证每次 `top()` 都是"按规则最大/最小的那个人"在最前面。默认规则 `less`（谁大谁优先）就是大根堆；换 `greater` 就变成小根堆（谁小谁优先）。

---

### 第 8 题

> 当哈希表的元素个数超过 bucket_count() * max_load_factor() 时，哈希表是如何进行扩容（重新分配桶数组、重新计算哈希值、元素重新挂链）的？

**答案：**

`unordered_*` 在插入元素后若满足 `size() > bucket_count() * max_load_factor()`，会触发 **rehash（再哈希/扩容）**，流程如下：

1. **判定触发**：每次插入后检查 `n + 1 > bucket_count * max_load_factor`（默认 max_load_factor=1.0，即"元素数将超过桶数"）；
2. **确定新桶数**：`rehash` 选择一个**新的素数桶数**（约等于 `旧桶数 * 2` 附近的质数，`_M_next_resize` 机制），使新装载因子回落到阈值以内；
3. **重新分配桶数组**：为新桶数 `allocate` 一块新的**桶数组**（旧桶数组不再使用）；
4. **重新计算哈希值**：对每个已存元素**用新的 `bucket_count` 重新取模**（`hash(key) % new_bucket_count`），因为桶数变了，旧桶号全部失效，必须重算；
5. **元素重新挂链**：遍历旧桶的所有链表，把每个结点按新桶号**搬进新桶的链表**（头部/尾部插入新桶链表），最后释放旧桶数组；
6. **更新内部状态**：`bucket_count`、`max_load_factor` 关系、迭代器（rehash 会**使所有迭代器失效**，但元素的引用/指针**仍然有效**——因为元素结点本身没被移动，只是换桶）。

**要点**：
- rehash 是 O(n)（n 个元素全部重挂），但由于桶数翻倍增长，**均摊 O(1)**；
- 可用 `reserve(n)` 预先 `rehash(ceil(n / max_load_factor))` 避免多次扩容；
- 迭代器失效、引用不失效，这一点与 vector 扩容（引用也失效）不同。

**通俗解释**：哈希表扩容像"储物柜不够用了，换一栋柜子更多的仓库"：先定好新仓库的柜子数量（素数，2 倍左右），把所有旧柜子里的东西**重新按新柜号算法算一遍该进哪个柜**（因为柜号 = 哈希值 mod 柜子数，柜子数变了号就变了），再一件件搬进新柜。东西（结点）本身没变，只是换了柜子——所以手里的"把玩参考"（引用）还有效，但"楼层导览"（迭代器）作废了。

---

### 第 9 题

> 针对包含多个成员变量的自定义类（如 x 和 y），如何利用位运算（如 ^、左移/右移）或 boost::hash_combine 设计一个分布均匀的哈希函数？

**答案：**

**目标**：把多个成员各自的信息"搅拌"进一个 `size_t`，让相似对象（只差一点）的哈希值差异尽量大、分布尽量均匀。

**方法一：位运算组合（移位 + 异或 + 乘大质数）**

```cpp
// 经典 hash_combine 思想（boost 同款核心公式）
inline size_t hash_combine(size_t seed, size_t value) {
    seed ^= value + 0x9e3779b9 + (seed << 6) + (seed >> 2);  // 黄金比例常数搅动
    return seed;
}

struct Point {
    int x, y;
    size_t hash() const {
        size_t h = 0x9e3779b9;                 // 初值（黄金比例的十六进制）
        h = hash_combine(h, std::hash<int>{}(x));
        h = hash_combine(h, std::hash<int>{}(y));
        return h;
    }
};
// 若在 std 里特化：operator() { return p.hash(); }
```

**为什么用 `^`、`<<`、`>>`、乘大质数**：
- **移位 `<<6`/`>>2`**：把 seed 的高/低位信息"搅匀"，避免低位信息丢失；
- **异或 `^`**：把新成员信息"叠加"进 seed；
- **加常数 `0x9e3779b9`**：这是**黄金分割比例** `(√5-1)/2 * 2^32` 的整数近似，是哈希领域公认的"搅拌常数"，让相邻输入映射到差异大的输出；
- **乘大质数**：很多实现会用 `h = h * 31 + member_hash`（如 Java `String.hashCode`）或 `h = h * 0x9E3779B1 + ...`，利用乘法把低位信息扩散到高位。

**方法二：直接调用 boost::hash_combine / boost::hash_value**

```cpp
#include <boost/functional/hash.hpp>
struct PointHash {
    size_t operator()(const Point &p) const {
        size_t h = 0;
        boost::hash_combine(h, p.x);
        boost::hash_combine(h, p.y);
        return h;
    }
};
```

**方法三：std::hash 各成员异或（简单但要注意对称性）**

```cpp
size_t operator()(const Point &p) const {
    return std::hash<int>{}(p.x) ^ (std::hash<int>{}(p.y) << 1);  // 右移一位防对称
}
```
注意 `x ^ y` 这种**朴素异或**有缺陷：`(1,2)` 与 `(2,1)` 同哈希（对称），最好给不同成员乘不同系数/移位。

**设计要点总结**：
1. 每个成员都过 `std::hash<member_type>` 得到原始哈希；
2. 用 `hash_combine`（移位+异或+黄金常数）逐成员混入；
3. 保证**相等的对象哈希必相同**（`a==b` → `h(a)==h(b)`）；
4. 避免对称性（`(x,y)` vs `(y,x)` 别同哈希，除非相等定义允许）。

**通俗解释**：哈希函数像"把两道菜的味道搅成一锅汤"。直接 x^y 像只放两种料、味道太单薄还容易撞味（对称）；hash_combine 则像"边加料边搅拌、再撒点黄金配比秘制调料"，让每个成员的信息都充分扩散到整个哈希值里，味道层次丰富、撞味概率低。

---

## 二、编程题

### 编程题 1

> 定义一个Person类（姓名、年龄），将其作为unordered_set的Key，请实现自定义hash<Person>和equal_to<Person>（分别用模板特化和函数对象两种方式），并测试插入和查找。

**答案：**

```cpp
#include <iostream>
#include <unordered_set>
#include <string>

struct Person {
    std::string name;
    int age;
    bool operator==(const Person &o) const {     // 供默认 equal_to 使用
        return name == o.name && age == o.age;
    }
};

// —— 方式A：模板特化 std::hash<Person> ——
namespace std {
    template <>
    struct hash<Person> {
        size_t operator()(const Person &p) const {
            size_t h = 0x9e3779b9;
            h ^= std::hash<std::string>{}(p.name) + 0x9e3779b9 + (h << 6) + (h >> 2);
            h ^= std::hash<int>{}(p.age) + 0x9e3779b9 + (h << 6) + (h >> 2);
            return h;
        }
    };
}
// 方式A 测试：直接用默认哈希（特化生效）
void testA() {
    std::unordered_set<Person> s;
    s.insert({"Alice", 30});
    s.insert({"Bob", 25});
    std::cout << "特化hash: " << s.count({"Alice", 30}) << "\n";   // 1
}

// —— 方式B：函数对象 ——
struct PersonHash {
    size_t operator()(const Person &p) const {
        size_t h = 0x9e3779b9;
        h ^= std::hash<std::string>{}(p.name) + 0x9e3779b9 + (h << 6) + (h >> 2);
        h ^= std::hash<int>{}(p.age) + 0x9e3779b9 + (h << 6) + (h >> 2);
        return h;
    }
};
struct PersonEqual {   // 函数对象版 equal
    bool operator()(const Person &a, const Person &b) const {
        return a.name == b.name && a.age == b.age;
    }
};
void testB() {
    std::unordered_set<Person, PersonHash, PersonEqual> s;
    s.insert({"Alice", 30});
    s.insert({"Bob", 25});
    auto it = s.find({"Alice", 30});
    std::cout << "函数对象hash: " << (it != s.end() ? "找到" : "未找到")
              << " " << it->name << "\n";
    s.erase({"Bob", 25});
    std::cout << "删除后 Bob 数量: " << s.count({"Bob", 25}) << "\n";   // 0
}

int main() { testA(); testB(); return 0; }
```

**要点**：两种方式给出了 Hash 与 Equal 的不同"载体"；`operator==` 是 KeyEqual 的基础（方式B 里用了独立函数对象）。所有方式必须保证"`==` 相等 → 哈希相同"。

**通俗解释**：方式A 是把哈希规则"写进标准库的官方手册"（特化 std::hash），用的时候什么都不用传；方式B 是"自带两本手册"（PersonHash、PersonEqual）塞给容器。效果一样，只是规则放的位置不同。

---

### 编程题 2

> 给定一个整数数组，使用unordered_map统计每个数字出现的次数，并找出出现次数最多的数字。

**答案：**

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>

int main() {
    std::vector<int> arr{1, 3, 5, 1, 3, 3, 6, 1, 5};

    std::unordered_map<int, int> freq;
    for (int x : arr) ++freq[x];        // 统计次数

    int bestNum = arr.front(), bestCnt = 0;
    for (const auto &[num, cnt] : freq) // C++17 结构化绑定
        if (cnt > bestCnt) { bestCnt = cnt; bestNum = num; }

    std::cout << "出现最多的数字: " << bestNum
              << " (出现 " << bestCnt << " 次)\n";   // 3 (3次)（若有并列取第一个）
    return 0;
}
```

**要点**：
- `unordered_map<int,int>` 让"数字 → 次数"的累加是平均 O(1)；
- 找最多只需遍历一遍 freq（O(唯一数字个数)）；
- 多个数字并列最多时，上面的写法输出**遍历中第一个达到最大**的（若要求输出任意一个，也满足题意）。

**通俗解释**：先建一张"数字 → 计次"的速查表（哈希表），每看到一个数就在对应格子 +1；然后翻一遍表，记住谁次数最多，就是答案。

---

### 编程题 3

> 使用priority_queue实现一个任务调度器，任务有优先级（int），要求优先级高的先执行。自定义任务类，并实现比较器。

**答案：**

```cpp
#include <iostream>
#include <queue>
#include <string>

struct Task {
    int priority;
    std::string desc;
};

// 方式一：重载 operator<（配合默认 std::less<Task>，大的先出）
bool operator<(const Task &a, const Task &b) {
    return a.priority < b.priority;   // 优先级高的“更大”，先出队
}

// 方式二：自定义函数对象比较器
struct TaskCmp {
    bool operator()(const Task &a, const Task &b) const {
        return a.priority < b.priority;   // 与 operator< 相同语义
    }
};

// 方式三：lambda（构造时传入）
auto taskCmp = [](const Task &a, const Task &b) { return a.priority < b.priority; };

int main() {
    // 三种用法选一即可；这里演示方式二
    std::priority_queue<Task, std::vector<Task>, TaskCmp> pq;
    pq.push({3, "低优先级任务"});
    pq.push({9, "紧急任务"});
    pq.push({5, "中等任务"});

    while (!pq.empty()) {
        std::cout << pq.top().priority << " : " << pq.top().desc << "\n";
        pq.pop();
    }
    return 0;
}
```

输出（优先级从高到低）：
```
9 : 紧急任务
5 : 中等任务
3 : 低优先级任务
```

**要点**：
- 默认 `std::less<Task>` 走 `operator<`，"更大"的（priority 高）先出 → **大根堆**；
- 方式三 lambda 用法：
  ```cpp
  std::priority_queue<Task, std::vector<Task>, decltype(taskCmp)> pq(taskCmp);
  ```
- 注意比较器语义与直觉相反：`a < b` 表示"a 的优先级更低"，这样高的反而先出。

**通俗解释**：任务调度器就是"永远让最紧急的插队"的队列。比较器定义了"谁算更紧急"：`a.priority < b.priority` 表示"a 没 b 紧急"（更小），所以 b 排前面、先出队。堆算法保证取 top 永远是"最紧急"的那一个。

---

# 第五套：无序容器补充、priority_queue、迭代器

## 一、问答题

### 第 1 题

> 什么是哈希函数？列举常见的哈希函数构造方法（至少三种）。

**答案：**

（与第四套第 1 题相同，这里给出完整版。）

**哈希函数**：把任意大小的 key 映射到固定范围下标 `h(k) -> [0, m)` 的函数，要求**计算快、分布均匀、相似输入输出差异大**。

**常见构造方法（至少三种）**：

1. **除留余数法**：`h(k) = k % m`，m 取**素数**且避开 2 的幂，最常用、最简单；
2. **乘法散列法**：`h(k) = ⌊m · (k·A mod 1)⌋`，`A ≈ 0.618…`（黄金分割），对 m 不敏感；
3. **平方取中法**：取 `k²` 的中间若干位；
4. **折叠法**：把 key 切成多段再相加（或异或）取模；
5. **数字分析法**：分析 key 各位分布，挑分布均匀的位组合；
6. **位运算混合（复合类型/字符串常用）**：`BKDRHash`（`h = h*131 + c`）、`FNV-1a`、`MurmurHash`、`hash_combine`（移位+异或+黄金常数）——用移位、异或、乘大质数把信息充分搅拌。

C++ 标准库已为整数/浮点/string/指针提供 `std::hash` 专门实现。

**通俗解释**：见第四套第 1 题——哈希函数就是把"学号"变成"储物柜号"的算法，方法不同只是"怎么算号"不同；好的算法让号分得又匀又快。

---

### 第 2 题

> 什么是哈希冲突？解决哈希冲突的常用方法有哪些？STL 无序容器采用哪种？

**答案：**

（与第四套第 2 题相同，完整版。）

**哈希冲突**：不同 key 经哈希得到同一桶号（`h(k1)==h(k2)`，`k1≠k2`）。桶有限而 key 无限，冲突不可避免。

**解决方法**：
1. **链地址法（拉链法）**：每桶挂链表，冲突元素全进同桶链表；
2. **开放定址法**：冲突时探测下一个空位（线性探测 / 二次探测 / 双重散列）；
3. **再哈希法**：换哈希函数再算；
4. **公共溢出区**：冲突元素放公共溢出表。

**STL 采用：链地址法（拉链法）**。`unordered_*` 每桶一个单链表（libstdc++ 的 `_Hashtable`）；元素超 `bucket_count * max_load_factor` 触发 rehash（扩桶重挂）。

**通俗解释**：见第四套第 2 题——冲突就是"两个学号算出同一个柜号"，拉链法是在柜子里加抽屉。

---

### 第 3 题

> 装载因子（Load Factor）是什么？它对哈希表的性能有何影响？一般控制在什么范围？

**答案：**

（与第四套第 3 题相同。）

**装载因子** `α = 元素数 / 桶数` = 平均每桶元素数。

**影响**：α 越大链越长/探测越多，冲突越多、越慢（时间↑）；α 越小桶越多、内存浪费越大（空间↑）。是时间与空间的权衡。

**范围**：STL `max_load_factor()` 默认 **1.0**；实践常控在 **0.5 ~ 1.0**（追求速度 0.5~0.75，省内存可到 1.0）。可用 `reserve(n)` 预置桶数。

**通俗解释**：见第四套第 3 题——装载因子是"平均每柜塞几件"，塞太满翻找慢，柜太多费空间。

---

### 第 4 题

> unordered_set 与 set 的主要区别有哪些？在什么场景下优先选择 unordered_set？

**答案：**

（与第四套第 4 题相同。）

|          | `set`            | `unordered_set`      |
| -------- | ---------------- | -------------------- |
| 底层     | 红黑树           | 哈希表               |
| 有序性   | 有序             | 无序                 |
| 查找     | O(log n)         | 平均 O(1)，最坏 O(n) |
| Key 要求 | Compare（`<`）   | Hash + `==`          |
| 边界操作 | ✅ lower_bound 等 | ❌                    |
| 迭代器   | 双向             | 正向                 |

**优先 unordered_set**：无需有序、只求查重/在不在、量大查频繁、要平均 O(1)。**优先 set**：需有序输出/区间查询、要 `lower_bound/equal_range`、对最坏时延敏感。

**通俗解释**：见第四套第 4 题——"编号开柜"快但无序 vs "字典翻页"慢但有顺序。

---

### 第 5 题

> unordered_map 与 map 的主要区别有哪些？unordered_map 是否支持下标操作？其下标操作有哪些功能？

**答案：**

**unordered_map 与 map 的主要区别**：

|          | `map`      | `unordered_map` |
| -------- | ---------- | --------------- |
| 底层     | 红黑树     | 哈希表          |
| key 有序 | ✅ 按键排序 | ❌ 无序          |
| 查找     | O(log n)   | 平均 O(1)       |
| 迭代器   | 双向       | 正向            |
| 边界操作 | ✅          | ❌               |

（key 唯一、value 可重复可改、`[]` 语义等在两者中一致。）

**是否支持下标操作？——支持。**

`unordered_map` 提供 `T& operator[](const Key&)` 和右值 key 版本，功能与 `map` 的 `[]` 完全一致：
1. **key 存在**：返回 value 引用（可读可改）；
2. **key 不存在**：插入"值初始化"的键值对并返回引用；
3. 因此同样"查改插三合一"，同样**没有 const 版本**（const 用 `find` / `at`）。

```cpp
std::unordered_map<std::string,int> m;
m["a"] = 1;         // 插入
++m["a"];           // 修改 → 2
int v = m["a"];     // 查找 → 2
```

**注意**：遍历顺序与 map 不同（unordered 无顺序）；频繁 rehash 时可能略微影响插入性能（可用 `reserve`）。

**通俗解释**：unordered_map 是"不排序的电话簿"——按键查更快（平均 O(1)）但没有字母序；`[]` 的功能和 map 一样：没有就补一条默认记录、有就给你可改的值引用。

---

### 第 6 题

> 为什么 unordered_multimap 不支持下标操作？

**答案：**

与第三套第 5 题（multimap 为什么没有 `[]`）同理，unordered_multimap **key 允许重复**，与 `[]` 语义根本冲突：

1. `[]` 要求"一个 key 唯一对应一个 value"，而它一个 key 对应**多个 value**，无法确定返回哪一个；
2. `[]` 的"不存在就插入"惯例在"允许重复"的容器里没有自洽语义（每次 `[]` 同一个 key 都该再插一个默认值吗？）；
3. 无法提供有意义的"可修改引用"。

正确用法：`insert` 插入，`equal_range(key)` 取该 key 的所有 value 区间，`count` 查个数，`find` 取第一个，`erase` 删全部。

**通俗解释**：`[]` 是"报名字取一个号码"，前提是名字唯一对应一个号；unordered_multimap 同名多条，报名字无法确定取哪条，所以干脆不提供 `[]`，改用 `equal_range` 把同名所有记录整段端走。

---

### 第 7 题

> 当 unordered_set / unordered_map 的 Key 为自定义类型时，需要提供哪两个模板参数的自定义实现？各自有哪几种实现方式？

**答案：**

需要提供**哈希函数（Hash）** 与 **相等比较（KeyEqual）** 两个模板参数（`unordered_set<Key, Hash, KeyEqual, Alloc>`）。

**Hash 的三种方式**：
1. **模板特化 `std::hash<Key>`**（定义在 `namespace std`）；
2. **自定义函数对象**（struct 重载 `operator()`）作为模板实参；
3. **lambda** 作为模板实参（配合 `decltype`）。

**KeyEqual 的三种方式**：
1. **重载 `operator==`**（默认 `std::equal_to<Key>` 走它）；
2. **模板特化 `std::equal_to<Key>`**；
3. **自定义函数对象 / lambda** 作为模板实参。

**硬性约束**：`a == b` 则 `h(a) == h(b)`（相等必同哈希；允许不同对象同哈希=冲突）。两种方式均可同时用于 `unordered_map`（key 部分）与 `unordered_set`。

（代码示例见第四套编程题 1 的 Person 演示。）

**通俗解释**：见第四套第 5 题——开柜需要"算柜号规则"（Hash）和"确认同物规则"（==），各有三种"写规则"的载体。

---

### 第 8 题

> 对于 std::string 类型，为什么不需要自定义哈希函数？

**答案：**

因为**标准库已为 `std::string` 提供了特化版 `std::hash<std::string>`**：

1. **标准库自带特化**：C++11 起 `<string>`（实际在 `<functional>` 里特化）提供了 `std::hash<std::string>`，默认 `unordered_set<std::string>` / `unordered_map<std::string, ...>` 直接用，无需任何额外代码；
2. **实现质量高**：标准库哈希对 string 采用成熟算法（libstdc++ 用 `_Hash_bytes`（MurmurHash 类），MSVC 用 FNV-1a 等），分布均匀、速度快，比自己手写更好；
3. **同理**：基本类型（int/double/bool/指针）、`std::string_view`、`std::filesystem::path`、智能指针等也都有标准特化。

**注意**：标准规定 `std::hash` 对"基本类型、string 等"必须可用；只有**自定义类型**（如自己的 `Person`）才需要自己提供（见第 7 题）。

**通俗解释**：`std::string` 是"出厂就配好储物柜算法"的类型——C++ 标准库把它的哈希算法写好了（还很专业），你直接用就行；只有自己造的新类型（Person 这种）才需要自己"配算法"。

---

## 二、编程题

### 编程题 1

> 定义一个 Student 类（含学号 id、姓名 name），将其作为 unordered_set 的 Key。要求：
> - 使用模板特化 std::hash<Student> 和 std::equal_to<Student>；
> - 使用函数对象的方式分别实现哈希和相等比较；
> - 测试插入、查找、删除操作。

**答案：**

```cpp
#include <iostream>
#include <unordered_set>
#include <string>

struct Student {
    int id;
    std::string name;
};

// ---------- 方式A：模板特化 ----------
namespace std {
    template <> struct hash<Student> {
        size_t operator()(const Student &s) const {
            size_t h = 0x9e3779b9;
            h ^= std::hash<int>{}(s.id) + 0x9e3779b9 + (h << 6) + (h >> 2);
            h ^= std::hash<std::string>{}(s.name) + 0x9e3779b9 + (h << 6) + (h >> 2);
            return h;
        }
    };
    template <> struct equal_to<Student> {
        bool operator()(const Student &a, const Student &b) const {
            return a.id == b.id && a.name == b.name;
        }
    };
}

// ---------- 方式B：函数对象 ----------
struct StudentHash {
    size_t operator()(const Student &s) const {
        size_t h = 0x9e3779b9;
        h ^= std::hash<int>{}(s.id) + 0x9e3779b9 + (h << 6) + (h >> 2);
        h ^= std::hash<std::string>{}(s.name) + 0x9e3779b9 + (h << 6) + (h >> 2);
        return h;
    }
};
struct StudentEqual {
    bool operator()(const Student &a, const Student &b) const {
        return a.id == b.id && a.name == b.name;
    }
};

template <typename S>
void run(const char *tag, S &s) {
    std::cout << "---- " << tag << " ----\n";
    s.insert({1001, "Alice"});
    s.insert({1002, "Bob"});
    s.insert({1001, "Alice"});                     // 重复，应被忽略
    std::cout << "size=" << s.size() << " (期望2)\n";

    auto it = s.find({1002, "Bob"});
    std::cout << "查找(1002,Bob): " << (it != s.end() ? it->name : "无") << "\n";

    s.erase({1001, "Alice"});
    std::cout << "删除(1001,Alice)后 size=" << s.size()
              << ", 存在? " << s.count({1001, "Alice"}) << "\n\n";
}

int main() {
    std::unordered_set<Student> sA;                      // 走特化
    std::unordered_set<Student, StudentHash, StudentEqual> sB;  // 走函数对象
    run("模板特化", sA);
    run("函数对象", sB);
    return 0;
}
```

**要点**：两种方式结果一致；重复插入被"等值判定"过滤（这里 id+name 全同才算重复）；删除同样用等值规则定位。

**通俗解释**：同第四套编程题 1——一套规则写在标准库手册里（特化），一套自己带手册（函数对象）；"重复与否"由 equal 规则决定，"存哪个柜"由 hash 规则决定，两套都必须一致才能正确去重、查找、删除。

---

### 编程题 2

> 给定一个整数数组，使用 unordered_map<int, int> 统计每个数字出现的次数，然后输出出现次数最多的数字（如果有多个，输出任意一个）。

**答案：**

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>

int main() {
    std::vector<int> arr{2, 4, 2, 5, 4, 4, 8, 2, 9};

    std::unordered_map<int, int> freq;
    for (int x : arr) ++freq[x];

    int best = 0, bestCnt = 0;
    for (const auto &[num, cnt] : freq)      // C++17 结构化绑定
        if (cnt > bestCnt) { bestCnt = cnt; best = num; }

    std::cout << "最多次数: " << best << " (" << bestCnt << " 次)\n";
    return 0;
}
```

**要点**：`++freq[x]` 平均 O(1) 统计；"多个并列任意一个"——上面 `>` 保证输出**第一个达到最大**的（符合"任意一个"）。

**通俗解释**：见第四套编程题 2——建"数字→次数"速查表，再翻表记最大。

---

### 编程题 3

> 将 100 万个随机整数插入到 unordered_set<int> 和 set<int> 中，分别记录插入耗时，并解释性能差异的原因。

**答案：**

```cpp
#include <iostream>
#include <chrono>
#include <random>
#include <set>
#include <unordered_set>

int main() {
    const int N = 1'000'000;
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, N * 8);   // 值域大、重复少

    auto t0 = std::chrono::steady_clock::now();
    std::set<int> s;
    for (int i = 0; i < N; ++i) s.insert(dist(rng));
    auto t1 = std::chrono::steady_clock::now();

    auto t2 = std::chrono::steady_clock::now();
    std::unordered_set<int> us;
    us.reserve(N);   // 预置桶数，避免频繁 rehash（公平起见可注释掉对比）
    for (int i = 0; i < N; ++i) us.insert(dist(rng));
    auto t3 = std::chrono::steady_clock::now();

    auto ms = [](auto a, auto b) {
        return std::chrono::duration_cast<std::chrono::milliseconds>(b - a).count();
    };
    std::cout << "set:            " << ms(t0, t1) << " ms\n";
    std::cout << "unordered_set:  " << ms(t2, t3) << " ms\n";
    return 0;
}
```

**预期结果**：unordered_set 通常明显快于 set（几十 ms vs 几百 ms 量级，与机器相关）。

**性能差异原因**：
1. **复杂度不同**：set 每次插入是红黑树 **O(log n)**（约 log₂10⁶ ≈ 20 次节点比较 + 指针旋转）；unordered_set 是**平均 O(1)**（算哈希 + 挂链），只有 rehash 时才有 O(n) 的额外代价（已用 `reserve` 规避）；
2. **内存布局**：红黑树每次插入都要 `new` 一个节点（堆分配），且节点分散、缓存不友好；哈希表桶数组连续、多数插入命中已有内存，缓存更友好；
3. **操作开销**：set 的比较器走 `operator<`（20 次左右整数比较 + 大量指针操作）；unordered 只需一次哈希 + 少量链表比较；
4. **最坏情况对比**：若数据高度冲突（例如所有值都 mod 同一桶），unordered 可能退化为 O(n²) 塌方，而 set 始终 O(log n)——所以"量大、分布均匀、求速度"用 unordered，"数据差/对最坏时延敏感"用 set。

**通俗解释**：set 像"每次放书都要从书架顶往下逐层比（O(log n)），还得在堆上现搭书位"；unordered_set 像"按书号一算就直奔柜子（O(1)）"，而且桶位是提前排好的（reserve 后不用临时扩建）。百万量级下"直奔柜子"显然比"逐层比大小"快得多；但前提是书号（哈希）分布别太差，否则全挤一个柜子反而更慢。

---

# 第六套：容器选择 / 迭代器 / 算法 / lambda / bind / function / mem_fn

## 一、问答题

### 第 1 题

> 如何根据实际需求选择合适的容器？请从**有序性**、**下标支持**、**查找时间复杂度**、**迭代器类型**四个方面详细分析。

**答案：**

（与第四套第 6 题相同，完整版。）

- **有序性**：需要自动排序/范围查询 → `set/map`（树）；不需要顺序只求快 → `unordered_*`；按插入顺序 → `vector/deque/list`。
- **下标支持**：O(1) 随机下标 → `vector`（连续、最快）；头尾都增删且要下标 → `deque`；关联容器仅 `map/unordered_map` 提供 `[]`（key 下标）。
- **查找复杂度**：平均 O(1) → `unordered_*`；O(log n) 且有序 → `set/map`；线性 O(n) → 序列容器 `find`；最坏时延敏感选红黑树（有界 O(log n)）。
- **迭代器类型**：随机访问 → `vector/deque/array/string`（可全局 `sort`）；双向 → `list/set/map`；正向 → `forward_list/unordered_*`；输入/输出 → 流迭代器。

（决策速查与通俗解释同第四套第 6 题：先问"要不要排好序、能不能按号取、取多快、搬运工具能不能跳着走"。）

---

### 第 2 题

> priority_queue 的底层数据结构是什么？默认是大根堆还是小根堆？如何改变为小根堆？

**答案：**

- **底层**：容器适配器，默认 `vector` 存储 + 堆算法（`make_heap/push_heap/pop_heap`）维护二叉堆；
- **默认**：`std::less<T>` 比较器 → **大根堆**（`top()` 返回最大值）；
- **改为小根堆**：把比较器换成 `std::greater<T>`：
  ```cpp
  std::priority_queue<int, std::vector<int>, std::greater<int>> pq;  // top() 是最小值
  ```
- 自定义类型可通过 `operator<` 或自定义比较器（见第 3 题）。

**通俗解释**：见第四套第 7 题——默认"谁大谁优先"（大根堆），换 `greater` 变成"谁小谁优先"（小根堆）；比较器写法和直觉相反，容易写反。

---

### 第 3 题

> 当 priority_queue 存储自定义类型时，需要如何提供比较方式？请列举三种写法。

**答案：**

**方式一：重载 `operator<`**（配合默认 `std::less<T>`）

```cpp
struct Task { int priority; std::string desc; };
bool operator<(const Task &a, const Task &b) { return a.priority < b.priority; }
std::priority_queue<Task> pq;     // 默认 less，优先级大的先出
```

**方式二：自定义函数对象比较器**（作为第三个模板参数）

```cpp
struct TaskCmp {
    bool operator()(const Task &a, const Task &b) const {
        return a.priority < b.priority;
    }
};
std::priority_queue<Task, std::vector<Task>, TaskCmp> pq;
```

**方式三：lambda**（构造时传入实例）

```cpp
auto cmp = [](const Task &a, const Task &b) { return a.priority < b.priority; };
std::priority_queue<Task, std::vector<Task>, decltype(cmp)> pq(cmp);
```

**注意**：三者语义统一为"**a 比 b 不优先**才返回 true"（`a < b` 表示 a 优先级更低），这样 `top()` 才取到最优先的；比较器必须满足严格弱序。

**通俗解释**：见第四套编程题 3——三种写法只是把"谁更紧急"的规则写在不同位置（Task 自己、独立裁判类、构造时的 lambda），规则一样，出队顺序就一样。

---

### 第 4 题

> 迭代器分为哪几类？分别说明各类型的特征，并列举支持该迭代器的 STL 容器。

**答案：**

C++ 标准把迭代器按能力分为**五类**（支持能力逐级增强）：

| 类别                                | 支持的操作                                             | 特征/用途                                                  | 典型容器/对象                                    |
| ----------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------- | ------------------------------------------------ |
| **输入迭代器（input）**             | `*`、`++`、`==/!=`、`->`                               | **只读、单向、一次性**（只能顺序读，不能回头），常用于读流 | `istream_iterator`、`istreambuf_iterator`        |
| **输出迭代器（output）**            | `*`（写）、`++`                                        | **只写、单向、一次性**，用于顺序写目标                     | `ostream_iterator`、`back_inserter` 等插入迭代器 |
| **前向迭代器（forward）**           | 输入+输出能力、可保存副本重复遍历                      | 单向多遍遍历                                               | `forward_list`、`unordered_*`                    |
| **双向迭代器（bidirectional）**     | 前向 + `--`                                            | 可前后移动                                                 | `list`、`set/map/multiset/multimap`              |
| **随机访问迭代器（random access）** | 双向 + `it+n`、`it-n`、`it[n]`、`it1-it2`、`< <= > >=` | 可 O(1) 跳到任意位置                                       | `vector`、`deque`、`array`、`string`、原始指针   |

**补充**：
- **连续迭代器（contiguous，C++17 新增概念）**：在随机访问基础上额外保证元素**内存连续**（`vector`、`array`、`string`、`basic_string_view`），可以用裸指针语义；
- 迭代器能力用 `std::iterator_traits<It>::iterator_category` 标记（如 `random_access_iterator_tag`），算法据此选择最合适实现；
- 能力**可传递**：随机访问 ⊃ 双向 ⊃ 前向 ⊃ 输入；某个算法需要随机访问时，双向迭代器容器（list）编译会失败（这正是 list 不能用 `std::sort` 的原因）。

**通俗解释**：迭代器像"阅读器上的游标"，按能干的活分五级：只能顺着读一遍（输入）、只能顺着写（输出）、能存书签重复读（前向）、能前进也能后退（双向）、能直接跳到第 n 页（随机访问）。能力越高，能用它的算法越多。

---

### 第 5 题

> 什么是"插入迭代器"？back_inserter、front_inserter、inserter 三者的区别是什么？它们底层分别调用了容器的什么函数？

**答案：**

**插入迭代器（Insert Iterator / Inserter）** 是**输出迭代器适配器**：给它赋值 `*it = value` 时，它不"覆盖"，而是**把 value 插入目标容器**。专门配合 `copy`、`transform` 等算法"往容器里灌数据"。

**三者区别与底层调用**：

| 适配器              | 底层调用           | 行为                    | 前置要求                                           | 注意                                         |
| ------------------- | ------------------ | ----------------------- | -------------------------------------------------- | -------------------------------------------- |
| `back_inserter(c)`  | `c.push_back(v)`   | 每次都**尾插**          | 容器必须有 `push_back`（vector/deque/list/string） | 保持原顺序                                   |
| `front_inserter(c)` | `c.push_front(v)`  | 每次都**头插**          | 容器必须有 `push_front`（deque/list/forward_list） | **会反转顺序**（每次插到最前）               |
| `inserter(c, pos)`  | `c.insert(pos, v)` | 在**指定位置 pos** 插入 | 容器必须有 `insert`（除 array/forward_list 外）    | 保持顺序；pos 只在构造时定一次，不随插入推进 |

**示例**：

```cpp
std::vector<int> v{1,2,3}, out;
std::copy(v.begin(), v.end(), std::back_inserter(out));  // out: 1 2 3

std::list<int> l;
std::copy(v.begin(), v.end(), std::front_inserter(l));   // l: 3 2 1（注意反转！）

std::set<int> s;
std::copy(v.begin(), v.end(), std::inserter(s, s.begin()));  // s: 1 2 3（关联容器 pos 只是提示）
```

**通俗解释**：普通输出迭代器是"写到哪算哪、覆盖旧值"；插入迭代器是"自动在指定位置加塞新货"。尾插器（back）排最后、头插器（front）插最前（所以整体会倒序）、位置插器（inserter）插在指定地点。它们底层其实就是替你调 `push_back/push_front/insert`。

---

### 第 6 题

> ostream_iterator 是如何工作的？请结合 copy 算法的源码说明其 operator=、operator*、operator++ 的作用。

**答案：**

**ostream_iterator** 是一个**输出迭代器**：每给它"赋值"一个值，它就把这个值写入绑定的输出流（可选带分隔符）。

**核心成员**：绑定一个 `std::basic_ostream*` 和一个分隔符 `const charT* _M_delim`。

**三个关键运算符的作用**：

```cpp
// 1) operator= ：真正执行“写值”
ostream_iterator& operator=(const T& value) {
    *_M_stream << value;          // 把值写进输出流
    if (_M_delim) *_M_stream << _M_delim;   // 若设置了分隔符，随后输出
    return *this;
}
// 2) operator* ：无意义地返回自身（让算法能写 *it = value）
ostream_iterator& operator*() { return *this; }
// 3) operator++ ：什么都不做（输出迭代器无需前进），返回自身
ostream_iterator& operator++() { return *this; }
ostream_iterator& operator++(int) { return *this; }
```

**结合 `copy` 源码看**：

```cpp
template<class InputIt, class OutputIt>
OutputIt copy(InputIt first, InputIt last, OutputIt d_first) {
    while (first != last)
        *d_first++ = *first++;   // 关键：对 d_first 做“解引用-赋值-自增”
    return d_first;
}
```

对 `d_first`（ostream_iterator）而言：
- `*d_first` → 调用 `operator*()`（返回自身）；
- `= *first` → 调用 `operator=(value)`，**真正把值写入流**；
- `++` → 调用 `operator++()`（空操作）。

所以 `*d_first++ = *first++;` 一次循环 = "读一个源值 → 写一个到流（带分隔符）"。整个 copy 就是"把源区间值流式输出"。

**示例**：
```cpp
std::vector<int> v{1,2,3};
std::copy(v.begin(), v.end(), std::ostream_iterator<int>(std::cout, " "));
// 输出: 1 2 3
```

**通俗解释**：ostream_iterator 像一台"自动打印+盖章机"。算法 `copy` 每次对它说"记下这个数（赋值）"，它就打印并把分隔符（如空格）也打上；`*` 和 `++` 只是让这台机器"配合流程"用的空动作（输出本来就没有位置概念，不用前进）。

---

### 第 7 题

> istream_iterator 的默认构造函数（无参）代表什么？如何使用它作为结束标志？

**答案：**

**默认构造（无参）代表"流结束哨兵（end-of-stream）"**：`std::istream_iterator<int>` 默认构造出的迭代器**不绑定任何流**，表示"没有更多数据可读"，等价于"EOF 哨兵"。

**用法**：把"从输入流构造的迭代器"与"默认构造的迭代器"用 `!=`/`==` 比较，作为结束标志：

```cpp
std::istream_iterator<int> it(std::cin);   // 绑定 cin，从当前位置开始读
std::istream_iterator<int> eof;            // 无参构造 = 结束标志
std::vector<int> v(it, eof);               // 直接构造容器：读到 EOF 为止
// 等价写法：
// std::vector<int> v;
// std::copy(it, eof, std::back_inserter(v));
```

**工作原理**：
- `istream_iterator` 在**构造时**（或首次解引用时）就读入第一个值，解引用返回当前缓存的值；
- 当流到达**文件尾**、或**读取失败**（如输入类型不匹配），迭代器内部标记为"已到结束状态"；
- 处于结束状态的迭代器与 `eof` **比较相等**，于是循环 `while (it != eof)` 自然终止。

**注意**：
- 默认构造的迭代器**不能解引用**（`*eof` 是 UB）；
- 输入类型不匹配（比如要 int 却输入字母）会**立即**把迭代器置为结束状态并让流进入 fail 状态；
- 配合 `istream_iterator` 的常见模式就是"从流读入容器"（`vector<int> v{istream_iterator<int>(cin), istream_iterator<int>()}`，注意括号歧义用 `{}` 或双层括号）。

**通俗解释**：无参构造的 istream_iterator 是一张"没有输入源的终止牌"。读数据的迭代器每读到一个值就往前走一步；走到文件末尾（读不到值了）它就变得和"终止牌"一样。算法/循环用"读数据的 != 终止牌"判断"还有没有下一个"，一相等就停。这避免了手动检查 `cin >> x` 的返回值。

---

### 第 8 题

> 反向迭代器 reverse_iterator 与普通迭代器的关系是什么？其 base() 成员函数有什么作用？

**答案：**

**关系**：`reverse_iterator` 是一个**迭代器适配器**，它**包裹**一个普通（正向）迭代器，把"前进"变成"后退"：
- `*rit` 实际上访问的是**底层迭代器 `current` 的前一个元素**；
- `++rit` 等价于 `--current`（反向迭代器前进 = 正向迭代器后退）；
- 对应关系：`c.rbegin() == reverse_iterator(c.end())`，`c.rend() == reverse_iterator(c.begin())`。

**base() 的作用**：返回**被包裹的底层正向迭代器** `current`。

对应关系（重要，存在 **1 个元素的偏移**）：

```
      begin()  rbegin()            rend()  end()
         │        │                  │      │
         ▼        ▼                  ▼      ▼
    ┌─────┬─────┬─────┬─────┬─────┐
    │ a0  │ a1  │ a2  │ a3  │ a4  │
    └─────┴─────┴─────┴─────┴─────┘
                 ▲                  ▲
            rbegin().base()   rend().base()
                (= end())
```

- `rbegin().base() == end()`
- `rend().base() == begin()`
- `*rbegin()` 访问的是 `a4`，而 `rbegin().base()` 指向 `end()`（a4 之后）——**base 总比反向迭代器逻辑指向的元素"靠后一格"**。

**为什么存在 1 个元素偏移**：`reverse_iterator` 内部保存的 `current` 指向"**当前逻辑元素的下一个**"。这样设计保证：反向迭代器的 `operator*` 只需 `--current` 再解引用，`operator++` 只需 `--current`，**成对合法区间**（`rbegin()` 到 `rend()`）能用 `++` 走完整段而不越界（因为 `end()` 是合法可退一位的位置）。

**用途**：需要在正/反迭代器间转换时用 `base()`。例如删除 `rit` 指向的元素，**不能** `erase(rit)`（erase 只收正向迭代器），而应 `erase(std::prev(rit.base()))`（或 `erase(--rit.base())`）——因为 `rit.base()` 指向的是"逻辑元素的下一个"，要删的是它前一个。

```cpp
std::vector<int> v{1,2,3,4,5};
for (auto rit = v.rbegin(); rit != v.rend(); ++rit)
    std::cout << *rit << " ";          // 5 4 3 2 1
std::cout << "\n";
// rbegin().base() 指向 end()
auto base = v.rbegin().base();         // 等价于 v.end()
```

**通俗解释**：reverse_iterator 像一个"倒着走的助手"，它的脚（底层迭代器）其实站在"当前元素的后一格"，每次说"向前走"实际是往后退。`base()` 就是"看看助手脚下的位置"——永远比它嘴上说的那个元素靠后一格。要把它脚下那个"真正的元素"干掉时，得往前退一位（`prev(base())`），否则会删错（删到下一个）。

---

## 二、编程题

### 编程题 1

> 使用 istream_iterator<int> 从标准输入读取整数，存入 vector，然后使用 ostream_iterator<int> 输出到屏幕，每个数后跟一个空格。

**答案：**

```cpp
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    // 从 cin 读到 EOF（Ctrl+D/Ctrl+Z）为止，全部装入 vector
    std::istream_iterator<int> in(std::cin);
    std::istream_iterator<int> eof;
    std::vector<int> v(in, eof);       // 直接构造

    // 输出到屏幕，每数后跟一个空格
    std::ostream_iterator<int> out(std::cout, " ");
    std::copy(v.begin(), v.end(), out);
    std::cout << "\n";
    return 0;
}
```

输入 `1 2 3 4 5`（回车后 Ctrl+D）→ 输出 `1 2 3 4 5 `。

**要点**：`vector(in, eof)` 用了"迭代器区间构造"一次读完；`ostream_iterator(std::cout, " ")` 第二个参数是分隔符（每写一个数后自动输出）。

**通俗解释**：一台"自动读数机"（istream_iterator）从键盘流水线读整数并直接装进仓库（vector），装到没了（EOF）就停；另一台"自动打印机"（ostream_iterator）把仓库里的数一个个打印出来，每个数后面自动补一个空格。

---

### 编程题 2

> 将 vector<int> 中的所有元素插入到 list<int> 的头部（使用 front_inserter），再插入到 set<int> 中（使用 inserter 并指定插入位置）。

**答案：**

```cpp
#include <iostream>
#include <iterator>
#include <algorithm>
#include <list>
#include <set>
#include <vector>

int main() {
    std::vector<int> v{1, 2, 3};

    // 1) 插入到 list 头部（front_inserter）
    std::list<int> l;
    std::copy(v.begin(), v.end(), std::front_inserter(l));
    std::cout << "list: ";
    for (int x : l) std::cout << x << " ";   // 3 2 1（头插导致反转）
    std::cout << "\n";

    // 2) 插入到 set（inserter + 指定位置）
    std::set<int> s;
    std::copy(v.begin(), v.end(), std::inserter(s, s.begin()));
    std::cout << "set: ";
    for (int x : s) std::cout << x << " ";   // 1 2 3（有序、去重）
    std::cout << "\n";
    return 0;
}
```

**要点**：
- `front_inserter(l)` 每次 `push_front` → 顺序反转（3 2 1）；
- `inserter(s, s.begin())` 对关联容器：`pos` 只是**提示位置**，元素仍按 set 的排序规则插入（这里 1 2 3）；重复元素被忽略。

**通俗解释**：front_inserter 是"每次都塞到最前面"，所以 1、2、3 依次头插后变 3、2、1；inserter 对 set 则是"按规则找位置"（红黑树自动排序），给的 begin() 只是参考坐标，最终仍排成 1 2 3。

---

### 编程题 3

> 定义 Task 类（含优先级 priority 和描述 desc），使用 priority_queue<Task> 实现一个任务调度器，优先级高的先执行。要求自定义比较器（三种方式任选两种）。

**答案：**

```cpp
#include <iostream>
#include <queue>
#include <string>

struct Task {
    int priority;
    std::string desc;
};

// 方式一：重载 operator<（默认 std::less 用）
bool operator<(const Task &a, const Task &b) {
    return a.priority < b.priority;
}

// 方式二：函数对象
struct TaskCmp {
    bool operator()(const Task &a, const Task &b) const {
        return a.priority < b.priority;
    }
};

int main() {
    // 用方式二
    std::priority_queue<Task, std::vector<Task>, TaskCmp> pq;
    pq.push({1, "后台清理"});
    pq.push({9, "实时上报"});
    pq.push({5, "日志刷新"});

    std::cout << "执行顺序（优先级高先执行）:\n";
    while (!pq.empty()) {
        std::cout << "  [" << pq.top().priority << "] " << pq.top().desc << "\n";
        pq.pop();
    }
    return 0;
}
```

输出：
```
执行顺序（优先级高先执行）:
  [9] 实时上报
  [5] 日志刷新
  [1] 后台清理
```

**方式三（lambda，若要演示）**：
```cpp
auto cmp = [](const Task &a, const Task &b) { return a.priority < b.priority; };
std::priority_queue<Task, std::vector<Task>, decltype(cmp)> pq(cmp);
```

**要点**：比较器语义"`a < b` 表示 a 优先级更低"，所以 priority 大的先出；三种方式任选两种即可（这里给了 operator< 和函数对象）。

**通俗解释**：同第五套编程题 3——调度器永远让"最紧急"的先出；比较器定义了"紧急程度"的判定。

---

### 编程题 4

> 给定 vector<int> vec = {1,2,3,4,5}，使用反向迭代器逆序输出所有元素。

**答案：**

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    for (auto rit = vec.rbegin(); rit != vec.rend(); ++rit)
        std::cout << *rit << " ";     // 5 4 3 2 1
    std::cout << "\n";
    return 0;
}
```

**要点**：`rbegin()` 指向最后一个元素，`rend()` 指向第一个元素之前；`++` 反向前进；范围 for 用 `for (int x : vec)` 则是正向。

**通俗解释**：反向迭代器就是"倒着走的游标"——从队尾出发，每次前进实际是向队头走，走到队头之前（rend）停，于是从 5 打印到 1。

---

### 编程题 5

> 使用 copy 算法配合 ostream_iterator 将 vector<string> 的内容输出到文件（使用 ofstream）。

**答案：**

```cpp
#include <iostream>
#include <fstream>
#include <iterator>
#include <algorithm>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> words{"hello", "world", "from", "ostream_iterator"};

    std::ofstream ofs("out.txt");
    if (!ofs) { std::cerr << "无法打开文件\n"; return 1; }

    // 把 words 写到文件，单词间用换行分隔
    std::copy(words.begin(), words.end(),
              std::ostream_iterator<std::string>(ofs, "\n"));

    // 顺便也打印一份到屏幕
    std::copy(words.begin(), words.end(),
              std::ostream_iterator<std::string>(std::cout, " "));
    std::cout << "\n";
    return 0;
}
```

out.txt 内容：
```
hello
world
from
ostream_iterator
```

**要点**：`ostream_iterator` 绑定的是 `ofstream`（同样是 `ostream` 派生），分隔符 `"\n"` 让每词一行；`copy` 内部就是 `*out = *first; ++out;`，遇 `operator=` 写文件。

**通俗解释**：ostream_iterator 不挑"打印机"——既可以是屏幕（cout）也可以是文件（ofstream），只要是 ostream 家族就行。copy 算法把 vector 里的词一个个"打印"到文件里，每个词后自动换行。

---

# 第七套：算法 / lambda / bind / function / allocator / 迭代器综合

## 一、问答题

### 第 1 题

> 算法 for_each 的第三个参数可以是什么类型？它是否允许修改容器元素？

**答案：**

`std::for_each(first, last, f)` 的第三个参数 `f` 是**一元可调用对象（unary callable）**，可以是：
1. **函数指针**：`void (*)(int&)`；
2. **函数对象**（重载 `operator()` 的类）；
3. **lambda 表达式**（最常用）；
4. **`std::function`**；
5. **`std::bind` 绑定结果**、`std::mem_fn` 等函数适配器结果。

```cpp
std::vector<int> v{1,2,3};
std::for_each(v.begin(), v.end(), [](int& x){ x *= 2; });   // 允许改
```

**是否允许修改容器元素？——取决于"以什么方式接收元素"**：
- 若 `f` 的形参是 `T&`（左值引用）：**可以**修改（例如 `[](int& x){ x *= 2; }`）；
- 若 `f` 的形参是 `const T&` 或 `T`（按值）：**不能**修改原元素（改的只是副本）；
- 注意：for_each 传的是**元素引用**，所以只要形参是 `T&` 就能改；而 `std::for_each` 的**第三个参数若本身声明为 `const` 可调用对象**，其 `operator()` 是 const 的，则内部也不能改。

**补充**：`for_each` 返回 `f`（可调用对象），可以用它收集副作用（如累计和）。

**通俗解释**：for_each 像"挨个点名，让你对每人喊一句话"。第三个参数就是"喊什么话"——可以是个函数、一个带 () 的对象、或 lambda。只要这句话里拿的是"本人的名牌"（引用）就能改本人；拿的是"复印件"（按值/const 引用）就改不了。

---

### 第 2 题

> remove_if 算法真的会删除元素吗？它的返回值是什么？如何与容器的 erase 配合真正删除元素？

**答案：**

**remove_if 不会真正删除元素！**

`remove_if(first, last, pred)` 是**只搬不移**的算法：把"不满足 pred 的元素"依次**覆盖写到前面**，返回"**新的逻辑末尾**"迭代器。它：
- **不改变容器 size**（元素个数不变，只是把该留的压到前段）；
- 不调用容器的 erase，不会释放内存；
- 区间 `[返回值, 原end)` 里是"逻辑上已删除"的残留值（内容未定义）。

**与 erase 配合（erase-remove_if 惯用法）真正删除**：

```cpp
std::vector<int> v{1,2,3,4,5,6};
v.erase(std::remove_if(v.begin(), v.end(),
                       [](int x){ return x > 3; }), v.end());
// v: {1,2,3}
```

- `remove_if` 返回新逻辑末尾 `it`；
- `erase(it, v.end())` 把残留尾巴真正删掉（此时才减小 size）。

**返回值**：`ForwardIt`——指向"新逻辑末尾"（保留序列最后一个有效元素的后一个位置）。

**注意**：
- 对 `list` 有专门的 `l.remove_if(pred)` 成员函数（真正删除且不使其他迭代器失效）；
- `remove`（删除等于某值）同理：`erase(remove(begin,end,val), end)`。

**通俗解释**：remove_if 像"整理队伍：把所有要留的人往前排，要删的人挤到队尾"——它**不宣布减员**（size 不变），只是把要删的赶到尾巴，并告诉你"有效队伍到哪结束"（返回值）。真正"裁员"要再喊一句 `erase(队尾这段)` 把尾巴那批清掉。两步合起来才叫"真删"。

---

### 第 3 题

> lambda 表达式的完整语法格式是什么？捕获列表有哪几种形式？值捕获与引用捕获的区别是什么？何时必须使用引用捕获？使用引用捕获需要注意什么（尤其是生命周期问题）？

**答案：**

**完整语法**（C++11，C++14/17 有扩展）：

```
[capture-list] (parameter-list) mutable(可选) noexcept(可选) -> return-type(可选) { body }
```

- **捕获列表 `[ ... ]`**：声明要"抓取"的外部变量及方式；
- **参数列表**：可省略（空）；
- **mutable**：允许在 lambda 内**修改按值捕获的副本**（默认按值捕获的副本是 const，不能改）；
- **noexcept / 异常规范**（C++17 起）可写；
- **返回类型**：可省略（自动推导）；C++14 起可用 `auto` 做泛型参数；C++14 起有**初始化捕获** `[x = expr]`；C++17 起可用 `[*this]`（按值捕获整个对象）。

**捕获列表形式**：

| 形式                  | 含义                            |
| --------------------- | ------------------------------- |
| `[]`                  | 不捕获任何外部变量              |
| `[a, b]`              | 按值捕获 a、b                   |
| `[&a, &b]`            | 按引用捕获 a、b                 |
| `[=]`                 | 全部按值捕获                    |
| `[&]`                 | 全部按引用捕获                  |
| `[=, &a]`             | 除 a 按引用外，其余按值         |
| `[&, a]`              | 除 a 按值外，其余按引用         |
| `[this]` / `[*this]`  | 捕获 this 指针 / 按值捕获 *this |
| `[x = expr]`（C++14） | 初始化捕获                      |

**值捕获 vs 引用捕获的区别**：
- **值捕获**：在 lambda **创建时**拷贝一份外部变量副本，之后外部变量怎么变都不影响 lambda 内的副本；
- **引用捕获**：不拷贝，lambda 内部用的是**外部变量的引用**，外部变化立即反映；lambda 内通过引用改的也是外部原变量。

```cpp
int x = 1;
auto by_val = [x]{ return x; };   // 捕获时拷贝 x=1
auto by_ref = [&x]{ return x; };  // 用引用，看外部实时值
x = 100;
by_val();   // 1（拷贝的）
by_ref();   // 100（引用的）
```

**何时必须使用引用捕获**：
1. 需要在 lambda 内**修改外部变量**（且要影响外部）；
2. 变量**不能拷贝**（如 `std::unique_ptr`、`std::ofstream`、`std::mutex` 等 move-only 对象）——只能引用捕获；
3. 拷贝代价高、只读访问大对象时（性能考虑）。

**引用捕获的注意（生命周期！）**：
- lambda 只是**持有引用**，不延长被引用对象的生命周期；
- **若 lambda 逃出被引用变量的作用域后再被调用，引用就悬垂（dangling），导致未定义行为**；
- 典型危险场景：把捕获了局部变量的 lambda 存进 `std::function` 返回出去、或投递到异步任务/线程里执行（见本套第 15 题）；
- 安全做法：需要"越活"就用**值捕获**或 `[x = std::make_shared<T>(...)]`（初始化捕获 + 智能指针）；函数返回 lambda 时尤其要避免 `[&]`。

**通俗解释**：值捕获像"拍照存档"（拍完照片里是你当时的模样，之后你变了照片不变）；引用捕获像"装了一面实时监控"（随时看你最新状态、也能遥控你）。危险的是"监控摄像头被搬到你已经拆掉的房子里还在用"——引用捕获的对象没了，lambda 再用就出事（悬垂）。所以 lambda 要是"活到别处去"，最好用照片（值捕获）而不是监控（引用捕获）。

---

### 第 4 题

> 如何在 lambda 中修改值捕获的变量？有哪些方法？

**答案：**

按值捕获的变量在 lambda 里是 **const 副本**，默认不能改。要修改，方法有：

**方法一：加 `mutable`**（最常用）

```cpp
int x = 0;
auto f = [x]() mutable { ++x; return x; };  // mutable 允许改副本
f();  // 返回 1，外部 x 仍为 0（改的是副本）
```

**方法二：按引用捕获**（改的是外部原变量）

```cpp
int x = 0;
auto f = [&x]() { ++x; };   // 不用 mutable，直接改外部 x
f();  // x == 1
```

**方法三：初始化捕获 + 值副本（C++14）**——为"在 lambda 里创建可变局部状态"：

```cpp
int x = 0;
auto f = [y = x]() mutable { ++y; return y; };  // 把 x 拷进 y，f 内部改 y
```

**区别**：
- `mutable` 改的是**副本**，不影响外部；引用捕获改的是**原变量**；
- 想"lambda 有自己的可演化状态、又不影响外部"→ `mutable` 或初始化捕获。

**通俗解释**：按值捕获的副本被"盖了 const 章"，`mutable` 就是"撕掉这张章"，让你能改这份复印件（原件不受影响）；如果你本来就想直接改原件，就用引用捕获（方法二）。

---

### 第 5 题

> bind1st 和 bind2nd 的作用是什么？它们为什么在 C++11 后被弃用？请用 std::bind 实现同样的功能。

**答案：**

**作用**：`std::bind1st(f, v)` / `std::bind2nd(f, v)` 把二元函数 `f` 的**第一个/第二个参数固定为 `v`**，得到一元函数对象。典型用于把"比较/查找"算法适配成谓词：

```cpp
// C++03 时代写法
std::vector<int> v{1,3,5,7};
auto it = std::find_if(v.begin(), v.end(),
                       std::bind2nd(std::greater<int>(), 4));  // 找 >4 的
```

**为什么弃用（C++11 弃用、C++17 已删除）**：
1. **与函数指针/成员函数配合极差**：bind1st/2nd 依赖 `std::unary_function/binary_function`（C++17 也删了），对函数指针、成员函数、仿函数组合支持残缺；
2. **可读性差、无法表达复杂绑定**（任意参数位置、多个固定值）；
3. **被 `std::bind` 全面取代**：bind 能任意绑定位置、任意个数、支持引用包装、支持成员函数。

**用 std::bind 实现同样功能**：

```cpp
#include <functional>
using std::placeholders::_1;

// bind2nd(greater<int>(), 4)  ≡  bind(greater<int>(), _1, 4)
auto it = std::find_if(v.begin(), v.end(),
                       std::bind(std::greater<int>(), _1, 4));

// bind1st(less<int>(), 0)  ≡  bind(less<int>(), 0, _1)   // 找 0 < x 即 x > 0
```

（更现代的做法是直接写 lambda：`[4](int x){ return x > 4; }`，可读性最好。）

**通俗解释**：bind1st/2nd 是老式"固定参数"工具：只能把二元函数的第一个或第二个参数钉死。功能太死板（钉死的只能是某一边、还不支持成员函数），C++11 出了万能的 `std::bind`（任意位置、任意参数都能钉），老工具就没用了，C++17 直接删掉。现在要么用 bind，要么干脆写 lambda。

---

### 第 6 题

> std::bind 的占位符（如 _1、_2）代表什么？bind 默认采用值传递，如何传递引用？

**答案：**

**占位符**：`_1, _2, …, _N` 位于 `std::placeholders`，表示"**调用时由第 N 个实参填入的坑**"：

```cpp
using namespace std::placeholders;
auto f = std::bind(add, _2, _1);   // 调用 f(a,b) → add(b, a)：_1 取第1个实参 a，_2 取第2个实参 b
f(10, 20);                          // 等价于 add(20, 10)
```

**bind 默认值传递**：绑定 `add` 的参数（如某个外部变量）时，**默认按值拷贝**一份存进绑定对象；之后外部变量变化不影响它。

**如何传递引用**：用 `std::ref(x)` / `std::cref(x)` 包装：

```cpp
void inc(int& n) { ++n; }
int x = 0;
auto f = std::bind(inc, std::ref(x));   // 传引用（std::cref 传 const 引用）
f();   // x == 1（若不用 ref，x 仍为 0）
```

**底层**：`std::ref` 返回 `std::reference_wrapper<T>`——一个**内部存 `T*` 的可拷贝包装类**，bind 拷贝它（只拷指针），调用时 `operator T&()` 解引用成引用。所以"值拷贝"只拷贝了指针，行为上是引用（详见本套第 14 题）。

**通俗解释**：占位符是"占坑的 _1、_2"——调用时第几个实参就填进哪个坑；`bind` 默认是"拍照存档"（拷贝值），想"装监控"（传引用）就用 `std::ref` 把变量包装成"只存地址的卡片"交给 bind，卡片虽被拷贝，但里面是地址，指向的仍是原变量。

---

### 第 7 题

> std::function 可以接收哪些可调用对象？请举例说明。

**答案：**

`std::function` 是一个**通用多态函数包装器**，可以存放**任意"可调用对象（callable）"**，只要签名兼容。可接收的类型包括：

1. **普通函数指针 / 全局函数**：
   ```cpp
   int add(int a, int b);
   std::function<int(int,int)> f1 = add;
   ```
2. **静态成员函数**：
   ```cpp
   std::function<int(int,int)> f2 = &Calc::staticAdd;
   ```
3. **成员函数**（需配合对象/指针用 bind 或 lambda）：
   ```cpp
   struct Foo { int sub(int x){return x-1;} };
   Foo foo;
   std::function<int(int)> f3 = std::bind(&Foo::sub, &foo, _1);
   std::function<int(Foo&,int)> f4 = &Foo::sub;      // 也可把对象作为第一参数
   ```
4. **lambda 表达式**：
   ```cpp
   std::function<int(int)> f5 = [](int x){ return x*2; };
   ```
5. **函数对象（仿函数）**：
   ```cpp
   struct Mul { int operator()(int x) const { return x*10; } };
   std::function<int(int)> f6 = Mul{};
   ```
6. **std::bind 的绑定结果**：
   ```cpp
   std::function<int(int)> f7 = std::bind(add, 1, _1);
   ```
7. **std::mem_fn 的包装结果**、其他可调用对象：
   ```cpp
   std::function<bool(const std::string&)> f8 = std::mem_fn(&std::string::empty);
   ```
8. **另一个 std::function**（拷贝/赋值）。

**特性**：
- 空 `std::function`（默认构造、未初始化）调用时抛 `std::bad_function_call`；
- 类型擦除：无论底层是什么，`std::function<int(int)>` 统一以 `int(int)` 接口调用。

**通俗解释**：std::function 像"万能插座"——不管来电的是灯（函数）、风扇（lambda）还是音响（函数对象），只要插头规格（签名 `int(int)` 这种）对，都能插进去用。它把你的可调用对象"擦掉具体身份"，统一成一个标准接口来调用。

---

### 第 8 题

> 使用 bind 绑定成员函数时，传递对象地址和传递对象副本有什么区别？在多线程环境下应如何选择？

**答案：**

**传递对象地址 `&obj`**：
- bind 只保存**指针**，调用时通过指针访问原对象；
- **不拷贝对象**，代价低；
- **风险**：若原对象先于 bind 结果销毁/失效，调用时是**悬垂指针 → 未定义行为**；多线程下还要保证访问安全。

**传递对象副本 `obj`**：
- bind **拷贝整个对象**，绑定结果持有独立副本；
- 调用的是副本，**与外部原对象互不影响**；
- **代价**：可能昂贵（大对象拷贝）；要求类型可拷贝；若对象含不可拷贝成员（mutex、unique_ptr）则无法用此方式。

```cpp
struct Worker { void run(int n) const; };
Worker w;
auto f1 = std::bind(&Worker::run, &w, _1);   // 地址：共享原对象，可能悬垂
auto f2 = std::bind(&Worker::run, w, _1);    // 副本：独立，安全但贵
```

**多线程环境下的选择**：
- **首选"共享所有权"**：用 `std::shared_ptr<Worker>`，bind 绑定 `shared_ptr`（会拷贝指针+引用计数），调用 `f(wp, ...)` 前先用 `wp.get()` 或直接让 bind 持 shared_ptr：
  ```cpp
  auto sp = std::make_shared<Worker>();
  auto f = std::bind(&Worker::run, sp, _1);  // bind 可接受 shared_ptr，内部转成 .get() 调用
  ```
  这样对象生命周期由引用计数管理，**不会悬垂**，也不整对象拷贝；
- 若传**地址**：必须保证对象**生命周期长于所有回调执行完毕**，并用**互斥锁/原子**保护共享状态——这是最容易出 UB 的写法；
- 若传**副本**：天然线程安全（各线程操作各自副本），但注意"修改不共享、丢失同步"的语义；
- 总结口诀：**要安全 → shared_ptr 共享；要隔离 → 值拷贝；要性能且能保证生命周期 → 传地址（并加锁）**。

**通俗解释**：传地址像"给你一张指向对象的地图"（轻量，但房子拆了你拿着地图也白搭→悬垂）；传副本像"把对象复印一份给你"（安全、独立，但贵）。多线程场景最稳妥的是"大家共同持有同一栋房子，谁最后走谁拆"——即 `shared_ptr`，既不用整栋复制，也不会提前拆房。

---

### 第 9 题

> 什么是函数对象？C++ 中所有可以称为函数对象的类型有哪些？

**答案：**

**函数对象（Function Object，也叫仿函数 Functor）**：任何**可以像函数一样被调用**（`obj(args...)` 语法合法）的对象或类型。在 C++ 中，"可调用对象（callable）"包含：

1. **函数指针**：`void (*)(int)`、全局/静态函数名；
2. **函数引用**；
3. **重载了 `operator()` 的类/结构体对象**（经典仿函数）：
   ```cpp
   struct Less { bool operator()(int a, int b) const { return a < b; } };
   ```
4. **lambda 表达式**（本质是编译器生成的匿名函数对象，重载了 `operator()`）；
5. **`std::function` 对象**（可调用对象包装器，本身也可调用）；
6. **`std::bind` 的返回结果**（绑定后的可调用包装对象）；
7. **`std::mem_fn` 的返回结果**（成员函数指针包装器）；
8. **`std::reference_wrapper` 包装的可调用对象**（`std::ref(func)`）；
9. **成员函数指针/成员对象指针**（本身可用 `(obj.*pmf)(...)` 调用；常配合 bind/mem_fn/function 使用）；
10. **标准库预定义函数对象**：`std::less`、`std::greater`、`std::plus`、`std::logical_and` 等（<functional>）。

**判定方法**：任何支持 `decltype(&T::operator())` 的类对象、或可以隐式转换为函数指针的，都是可调用对象；C++17 用 `std::is_invocable` 检测。

**通俗解释**：函数对象就是"长得像函数、能被 `()` 调用"的任何东西——不只是传统函数，还包括带了"()"按钮的类对象、lambda、绑好参数的 bind 结果、标准库的 less 等。它们都能装进 `std::function`、传给算法当策略。

---

### 第 10 题

> 成员函数适配器 mem_fn 的作用是什么？它和 bind 绑定成员函数有什么区别？在 for_each 中如何使用 mem_fn？

**答案：**

**mem_fn 的作用**：`std::mem_fn(&Class::memberFunc)` 把**成员函数指针**包装成一个"**普通可调用对象**"，调用时**第一个参数就是对象**（对象引用/指针/智能指针都行），返回成员函数的结果：

```cpp
struct Number {
    int v;
    void print() const { std::cout << v << " "; }
    bool isEven() const { return v % 2 == 0; }
};

auto f = std::mem_fn(&Number::print);
f(num);            // 传入对象引用
f(&num);           // 传入对象指针
f(shared_ptr<Number>(&num));  // 传入智能指针（可调用）
```

**mem_fn 与 bind 绑定成员函数的区别**：

|              | `mem_fn(&C::f)`                                | `bind(&C::f, obj, ...)`    |
| ------------ | ---------------------------------------------- | -------------------------- |
| 对象由谁决定 | **调用时**传（第一参数），灵活                 | **绑定固定**（obj 已定死） |
| 是否拷贝对象 | 不拷贝，运行时给谁处理谁                       | 绑定 obj 副本/引用         |
| 额外参数     | 成员函数实参紧跟对象                           | 用占位符拼接               |
| 适用场景     | 需要"对象作为参数"的算法（for_each/transform） | 对象固定、只需留参数坑     |

**在 for_each 中使用**：

```cpp
std::vector<Number> nums{{1},{2},{3},{4}};

// 用 mem_fn 调用无参成员函数
std::for_each(nums.begin(), nums.end(), std::mem_fn(&Number::print));  // 1 2 3 4

// 配合 remove_if 用成员函数做谓词（mem_fn 返回成员函数的返回值，即 bool）
nums.erase(std::remove_if(nums.begin(), nums.end(),
                          std::mem_fn(&Number::isEven)), nums.end());
// 剩下 1 3
```

**通俗解释**：成员函数本来"必须挂在某个对象上"才能调用（`obj.f()`）；mem_fn 把它"拆下来做成一个独立按钮"，按的时候把"挂在谁身上"（第一个参数）临时告诉你。所以它能直接塞进 for_each——for_each 挨个把元素递过来，正好当"主人"。bind 则是先把主人定死，只留"按按钮传的参数"。

---

### 第 11 题

> 空间配置器为什么将内存分配和对象构造分开？这种设计有什么好处？

**答案：**

**设计理念**：把"**分配原始内存**"（`allocate/deallocate`，只涉及内存）与"**对象构造/析构**"（`construct/destroy`，调用构造/析构函数）**分开为两个阶段**。

**好处**：

1. **解耦内存管理与对象生命周期**：内存是"空间"，对象是"空间里的有生命实体"。同一块内存可以被反复"造对象—析构—再造"，而不必每次分配/释放内存；
2. **性能提升（避免无谓构造）**：容器扩容时只需**搬移**已有对象到新内存，不需要为"即将覆盖的位置"先构造临时对象；`vector` 预留的 capacity 空间是"未构造的原始内存"，不占对象构造开销；
3. **支持在指定地址就地构造**（placement new / `allocator_traits::construct`）：`emplace` 系列直接"在预留内存上构造"，省去临时对象与拷贝/移动；
4. **灵活定制分配策略**：可替换 allocator（如池化、共享内存、对齐分配），而构造逻辑不变；
5. **正确管理非平凡类型**：未初始化内存上直接赋值是 UB（会调析构），分开后能保证"先 construct 后使用、先析构再 deallocate"的顺序正确性。

**现代写法注意**：`std::allocator::construct/destroy` 在 C++17 弃用、C++20 删除；现代代码应使用 **`std::allocator_traits<A>::construct/destroy`** 或 C++20 的 **`std::construct_at` / `std::destroy_at`**（这正是本题"分开设计"在标准库中的现代落地）。

**通俗解释**：内存像"毛坯房"，对象像"入住的人"。好的管理是"**先租毛坯房，需要时再让住户入住，住户走了只清空不退房**"——这样换住户（构造/析构对象）不用重新找房（分配内存），省时省力。容器预留的 capacity 就是"已租但没住人的毛坯房"。

---

### 第 12 题

> 简述二级空间配置器（SGI STL）的基本原理，包括allocate、deallocate、construct、destroy的调用关系。

**答案：**

**SGI STL 空间配置器**分两级（这是历史教材内容，现代 libstdc++ 默认已改用 `operator new`，但原理仍有教学价值）：

**一级配置器 `malloc_alloc`**：处理**大块内存**（> 128 字节），直接调用 `malloc/free`，并支持"OOM 处理函数"（分配失败先调用回调，回调可尝试释放内存或抛出 `bad_alloc`）。

**二级配置器 `default_alloc`**：处理**小块内存**（≤ 128 字节），采用**内存池 + 16 条自由链表**避免频繁 malloc/free 的碎片与开销（详见第 16 题）。

**allocate/deallocate/construct/destroy 的调用关系**：

```
容器（如 vector）需要内存/对象
   │
   ├─ 构造对象阶段：allocator::construct(ptr, args...)
   │       └─ placement new(ptr) T(args...)     ← 只调构造函数，不分配内存
   │
   └─ 分配内存阶段：allocator::allocate(n)
           └─ size <= 128 ? 二级配置器(default_alloc) : 一级配置器(malloc_alloc)
                   │
                   ├─ default_alloc::allocate: 找自由链表；空则 refill → chunk_alloc 从内存池取
                   └─ malloc_alloc::allocate: malloc

销毁/释放的调用关系（对称）：
   ├─ destroy(ptr)        ──> 调用析构函数（可能析构区间）
   └─ deallocate(ptr, n)  ──> size<=128 ? 归还自由链表 : free
```

即：**allocate/deallocate 管"内存"**，**construct/destroy 管"对象生命周期"**；容器用 `construct` 在 `allocate` 到的内存上造对象，用 `destroy` 析构后再 `deallocate` 还内存。**必须先 construct 再使用、先 destroy 再 deallocate**。

**通俗解释**：这套配置器把"批地（allocate）、盖房（construct）、拆房（destroy）、退地（deallocate）"分成四个独立工序。小块地皮不走昂贵的"土地局"（malloc），而是用自建小区（内存池）+ 预约表（自由链表）快速周转；大块地皮才走土地局。现代实现大多不再自建小区，直接用系统 malloc（通常自带内存缓存），所以这套池化方案主要作为历史原理学习。

---

### 第 13 题

> **反向迭代器（reverse_iterator）与正向迭代器的对应关系**：rbegin() 和 rend() 对应的底层基础迭代器 base() 是什么位置？为什么存在 1 个元素的物理位置偏移？

**答案：**

（与第六套第 8 题相同，完整版。）

- `c.rbegin()` 等价于 `reverse_iterator(c.end())`，即 `rbegin().base() == c.end()`；
- `c.rend()` 等价于 `reverse_iterator(c.begin())`，即 `rend().base() == c.begin()`。

**1 个元素偏移的成因**：`reverse_iterator` 内部保存的 `current` 总是指向**当前逻辑元素的下一个位置**（"逻辑元素之后"）。因此：

```
*r  == *std::prev(r.base())     // 解引用要先退一位
++r == --r.base()               // 反向前进 = 底层后退
```

设计理由：保证 `[rbegin(), rend())` 是**合法迭代区间**——`rend()` 的底层是 `begin()`，而"合法可退一位"的位置正好能配合 `operator*`（先 `--` 再解引用），使整个区间用 `++` 走完不会越界；同时 `rbegin()` 对应的 `end()` 是合法的"最后一个元素之后"。

**使用注意**：用 `base()` 做删除等转换时，`r.base()` 指向的是"逻辑元素的下一个"，要删 `*r` 对应的元素应 `erase(std::prev(r.base()))` 或 `erase(--r.base())`。

**通俗解释**：见第六套第 8 题——反向迭代器"脚站在当前元素后面一格"，`base()` 看的就是脚下位置；所以它比嘴上说的元素靠后一格，这个"1 格偏移"是为了保证倒着走也能安全遍历整个区间。删除时记得先退一位，别删到下一个元素。

---

### 第 14 题

> std::bind 为什么默认会对传入参数进行值拷贝？若需传递引用，std::ref 底层包装类 std::reference_wrapper 是如何工作的？

**答案：**

**为什么默认值拷贝**：`std::bind` 必须把"被绑定的参数"**存储**在返回的绑定对象里，以备将来调用（可能跨越很长生命周期）。为了**不依赖外部对象的存活**、保证绑定对象自洽可用，标准默认**按值复制**所有被绑定参数——这样绑定结果"自带一份存档"，外部变量没了它照样能用。这也是 bind 强调"对象语义、值语义"的体现。

**如何传引用**：用 `std::ref(x)` / `std::cref(x)` 把 x 包成 `std::reference_wrapper<T>`，bind 拷贝的是"包装器"而不是值。

**reference_wrapper 的工作原理**：

```cpp
template <class T>
class reference_wrapper {
    T* _M_ptr;                    // 只存指针！
public:
    reference_wrapper(T& t) : _M_ptr(std::addressof(t)) {}
    // 关键：隐式转换为 T&，调用时解出引用
    operator T& () const noexcept { return *_M_ptr; }
    T& get() const noexcept { return *_M_ptr; }
    // 若 T 是可调用对象，还转发 operator()
    template <class... Args>
    auto operator()(Args&&... args) const
        -> decltype(std::invoke(*_M_ptr, std::forward<Args>(args)...)) { ... }
};
```

- 内部**只保存一个 `T*`**（极廉价、可拷贝）；
- bind 拷贝 reference_wrapper 时**只拷贝指针**（值拷贝的是"地址卡片"）；
- 调用绑定对象时，通过隐式 `operator T&` **解引用成原对象的引用**——所以行为上等于"传引用"。

**要点**：`std::ref` 本身不延长生命周期，只是"包一个指针"；原对象销毁后用它仍是悬垂（UB），生命周期仍需自行保证。

**通俗解释**：bind 默认"复印存档"（值拷贝）是为了让绑定结果能独立存活。`std::ref` 则把变量换成一张"只写地址的卡片"（reference_wrapper 内部就一个指针），bind 复印这张卡片很便宜，但调用时按卡片上的地址找到**原变量**——所以"卡片被复制了，人还是那个人"。卡片不保命，原主人没了，卡片照样是废纸（悬垂）。

---

### 第 15 题

> 在异步回调或函数返回 std::function 时，如果 Lambda 引用捕获了局部变量（如 [&]），会发生什么未定义行为？

**答案：**

**会发生"悬垂引用（dangling reference）"导致的未定义行为（UB）**：

- lambda 用 `[&]` 捕获的是**局部变量的引用**；
- lambda 被存进 `std::function` 返回出去、或投递到异步任务/线程里后，**原函数已返回、局部变量已销毁**；
- 之后（在另一个作用域/另一个线程）再调用这个 lambda，它去访问**已销毁的局部变量**——内存可能已被复用或释放，读写出错 → UB（可能崩溃、数据错乱、或"碰巧正常"这种最坏情况）。

**典型危险代码**：

```cpp
std::function<int()> makeCounter() {
    int local = 100;
    return [&] { return local++; };   // ❌ local 是栈上局部变量，函数返回即销毁
}
auto f = makeCounter();
f();   // UB：访问已销毁的 local
```

```cpp
// 异步投递同样危险
int val = 42;
std::async(std::launch::async, [&]{ use(val); });   // 若线程执行晚于作用域结束 → UB
```

**正确做法**：
1. **值捕获**：`[=]` 或 `[local]`（拷贝副本，独立存活）；
2. **初始化捕获 + 智能指针（C++14）**：`[x = std::make_shared<int>(42)]`，让 lambda 自己"持有"数据（引用计数管理生命周期）；
3. 若确实要引用外部对象，用 **`shared_ptr` 共享所有权**（`[sp]` 捕获 shared_ptr 副本，引用计数保命）；
4. 确保**外部对象的生命周期 >= lambda 的调用期**（如对象是静态/全局、或明确同步等待任务完成）。

**通俗解释**：`[&]` 的 lambda 像"一张写着『去老地址取货』的纸条"——但纸条被寄到别处去用的时候，老地址（局部变量）早就拆了。纸条再拿去取货就出事了（UB）。正确做法：要么把货**复印一份**带在身上（值捕获），要么把货装进**集装箱（shared_ptr）**让纸条一起带着，谁最后用谁负责销毁。

---

### 第 16 题

> **SGI STL 二级空间配置器细节**：
> 1. 一级配置器（malloc_alloc）和二级配置器（default_alloc）的阈值划分是多少（128 字节）？
> 2. 二级配置器维护的 16 个自由链表（free_list）节点大小分布如何（8 字节到 128 字节，步长 8 字节）？
> 3. 自由链表的内存池机制（chunk_alloc）在内存不足时是如何进行后备处理的？

**答案：**

> （这是经典教材《STL 源码剖析》的内容，属于**历史实现**；现代 libstdc++ 的默认 `std::allocator` 已不再采用这套两级池化，而是走 `operator new`/系统 malloc（其本身自带内存缓存）。这里按教材原理讲解。）

**1）两级阈值：128 字节**
- 请求大小 **> 128 字节** → 交给**一级配置器 `malloc_alloc`**：直接 `malloc`/`free`，并带 OOM 处理（失败先调 `__malloc_alloc_oom_handler`，无处理器则抛 `bad_alloc`）；
- 请求大小 **≤ 128 字节** → 交给**二级配置器 `default_alloc`**：走内存池 + 自由链表，避免小块频繁 malloc 的开销与碎片。

**2）16 条自由链表（free_list）**
- 每条链表负责**一种按 8 字节对齐的块大小**：
  ```
  #define __ALIGN 8
  第 0 条：8 字节
  第 1 条：16 字节
  第 2 条：24 字节
  ...
  第 15 条：128 字节
  ```
- 任意请求大小先**向上取整到 8 的倍数**（`round_up(n) = ((n + 7) & ~7)`），再定位到对应 free_list；
- 每块内存的第一个字（`*ptr`）用作**指向下一块的指针**，串成单链表；链头存在 `free_list[node]`。

**3）chunk_alloc 与内存池后备处理（allocate 时自由链表为空 → refill → chunk_alloc）**

- `allocate(size)`：若 free_list 有块，弹出返回；否则调 `refill(round_up(size))`；
- `refill(nobjs=20)`：调用 `chunk_alloc(nobjs*size, nobjs)` 向内存池要 **20 块**；只拿到 1 块就直接用，拿到多块则留 1 块返回、其余串进 free_list；
- `chunk_alloc` 流程（**重点：内存不足的后备处理链**）：
  1. 内存池剩余 `bytes_left >= nobjs*size` → 直接划走，结束；
  2. 剩余 `>= size`（不够 20 块）→ 能分几块分几块，结束；
  3. 内存池**完全不够** → 向系统 `malloc(2 * nobjs * size + ROUND_UP(heap_size >> 4))` 申请一大块（含"外快"减少下次申请），补充内存池；`heap_size` 累计跟踪；
  4. **malloc 失败** → 遍历**比当前更大的 free_list（下标 i > n 的链表）**，搜到非空块就**归还给内存池**，然后递归重试 chunk_alloc；
  5. 若更大的 free_list 也没有 → 调**一级配置器**（其 OOM 处理函数再试 malloc 或抛 `bad_alloc`）；
  6. 内存池累积超过 2MB（`heap_size > 2MB`）时，把多余部分归还给系统。

**deallocate**：大小 ≤ 128 则把块**放回对应 free_list**（头插）；> 128 走 `free`。

**通俗解释**：二级配置器像个"自助小零件仓库"——按零件尺寸（8/16/…/128 字节）分好 16 排抽屉，取小零件不用去大市场（malloc），抽屉里有就拿，没有就从自家**大库存（内存池）**一次批 20 个补齐。库存也没了才去系统市场进货；市场不卖，就先去翻"更大尺寸的抽屉"里有没有能拆用的，再不行才求助一级配置器（含 OOM 救济）。这套"先自给、再化缘、最后硬顶"的后备链，就是 chunk_alloc 的精髓。现代实现里系统 malloc 自带高效缓存，这套手工池化就退役了。

---

## 二、编程题

### 编程题 1

> 使用istream_iterator从标准输入读取整数，存入vector，然后使用ostream_iterator输出到屏幕，每两个数之间用逗号分隔。

**答案：**

```cpp
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    std::istream_iterator<int> in(std::cin), eof;
    std::vector<int> v(in, eof);            // 读入全部整数

    // 输出：前 n-1 个数后跟逗号+空格，最后一个数后换行
    if (!v.empty()) {
        std::ostream_iterator<int> out(std::cout, ", ");
        std::copy(v.begin(), v.end() - 1, out);   // 前 n-1 个带分隔符
        std::cout << v.back() << "\n";            // 最后一个不带
    }
    return 0;
}
```

输入 `1 2 3 4 5` → 输出 `1, 2, 3, 4, 5`。

**要点**：`ostream_iterator(std::cout, ", ")` 分隔符是逗号+空格；为了**结尾不出现多余的逗号**，用 `copy` 输出前 n-1 个（带分隔符），最后单独输出最后一个元素。

**通俗解释**：先用"自动读数机"把整数装进 vector；输出时让"自动打印机"每打印一个数就带一个逗号，但最后一个数不能带——所以先批量打前 n-1 个，最后一个单独打。

---

### 编程题 2

> 实现一个自定义的插入迭代器，使其能够将元素插入到任意容器的指定位置（类似inserter），但要求底层调用容器的insert并返回迭代器。

**答案：**

```cpp
#include <iostream>
#include <iterator>
#include <vector>
#include <algorithm>

// 自定义插入迭代器：绑定 容器& + 位置迭代器，赋值时调用容器.insert
template <typename Container>
class my_insert_iterator {
    Container* cont;
    typename Container::iterator pos;
public:
    using iterator_category = std::output_iterator_tag;
    using value_type        = void;
    using difference_type   = void;
    using pointer           = void;
    using reference         = void;

    my_insert_iterator(Container& c, typename Container::iterator p)
        : cont(&c), pos(p) {}

    // 核心：赋值 → 调用 insert，并让 pos 指向"刚插入元素之后"，
    //      这样连续赋值会按顺序依次插入（与 std::inserter 行为一致）
    my_insert_iterator& operator=(const typename Container::value_type& val) {
        pos = cont->insert(pos, val);   // 返回新插入元素迭代器
        ++pos;                          // 移到其后，保持顺序
        return *this;
    }
    my_insert_iterator& operator*()     { return *this; }
    my_insert_iterator& operator++()    { return *this; }
    my_insert_iterator& operator++(int) { return *this; }
};

template <typename Container>
my_insert_iterator<Container> my_inserter(Container& c,
                                          typename Container::iterator p) {
    return my_insert_iterator<Container>(c, p);
}

int main() {
    std::vector<int> v{10, 40};
    std::vector<int> src{20, 30};
    // 把 src 依次插入到 v 中 40 之前，保持顺序
    std::copy(src.begin(), src.end(), my_inserter(v, std::next(v.begin(), 1)));
    for (int x : v) std::cout << x << " ";   // 10 20 30 40
    return 0;
}
```

**要点**：
- 重载 `operator=` 调 `cont->insert(pos, val)`，**用返回值刷新 pos 并 `++pos`**——否则连续赋值都会插在同一位置导致逆序；
- `operator*`/`operator++` 是输出迭代器的空操作（与 ostream_iterator 同理）；
- 类型别名（`iterator_category` 等）方便适配 STL 算法；C++17 里可用 `std::iterator` 已弃用，应手动写这些别名。

**通俗解释**：这是"自己造一台插入器"：它记住目标容器和位置；算法每次"给它一个值"（`operator=`），它就调用容器的 `insert` 真正插进去，并**把位置往后挪一格**（这样下次插入排后面，顺序不乱）。`*` 和 `++` 只是配合算法的空壳动作。

---

### 编程题 3

> 编写lambda表达式，实现以下功能：
> - 捕获外部变量（值传递）并修改（使用mutable）。
> - 捕获外部变量（引用传递）并修改。
> - 混合捕获（除某个变量外其他全部引用）。

**答案：**

```cpp
#include <iostream>

int main() {
    // 1) 值捕获 + mutable：改的是副本，外部不变
    int a = 10;
    auto f1 = [a]() mutable { a += 5; return a; };
    std::cout << "mutable 内部结果: " << f1()      // 15
              << ", 外部 a: " << a << "\n";        // 10（不变）

    // 2) 引用捕获：直接改外部
    int b = 10;
    auto f2 = [&b]() { b += 5; };
    f2();
    std::cout << "引用捕获后外部 b: " << b << "\n"; // 15

    // 3) 混合捕获：[&] 全部引用，但 c 单独按值
    int c = 100, d = 1, e = 2;
    auto f3 = [&, c]() mutable {
        c += 10;      // 改的是 c 的副本
        d += 10;      // 引用，改外部 d
        e += 10;      // 引用，改外部 e
    };
    f3();
    std::cout << "混合捕获: c=" << c << " d=" << d << " e=" << e << "\n";
    // 输出: c=100 d=11 e=12（c 未变，d、e 变了）
    return 0;
}
```

**要点**：
- `[a]() mutable`：按值捕获 a 的**副本**，mutable 允许改副本，外部 a 不变；
- `[&b]`：改外部 b；
- `[&, c]`：除 c 按值外其余按引用（`[=, &c]` 则是除 c 引用外其余按值）。

**通俗解释**：值捕获+mutable = "拿着复印件改（原件不动）"；引用捕获 = "直接改原件"；`[&, c]` = "除了 c 用复印件，其它都用监控（引用）"。

---

### 编程题 4

> 使用bind绑定一个普通函数（如int add(int a, int b)）和一个成员函数（如void print()），分别用function接收并调用。

**答案：**

```cpp
#include <iostream>
#include <functional>

int add(int a, int b) { return a + b; }

struct Foo {
    int id;
    Foo(int i) : id(i) {}
    void print() const { std::cout << "Foo id = " << id << "\n"; }
};

int main() {
    // 1) 绑定普通函数 add，第一个参数固定为 10
    std::function<int(int)> f1 = std::bind(add, 10, std::placeholders::_1);
    std::cout << "f1(5) = " << f1(5) << "\n";          // 15

    // 2) 绑定成员函数 print()，对象用地址绑定
    Foo foo(42);
    std::function<void()> f2 = std::bind(&Foo::print, &foo);
    f2();                                               // Foo id = 42

    // 3) 也可以把对象作为调用时的参数（占位符方式）
    std::function<void(const Foo&)> f3 = std::bind(&Foo::print, std::placeholders::_1);
    f3(foo);                                            // Foo id = 42
    return 0;
}
```

**要点**：
- 普通函数 bind 后留 `_1` 占位；
- 成员函数 bind：第一个实参是**对象指针/引用**（这里 `&foo`），没有剩余参数就用 `void()`；
- 都可用 `std::function` 统一接收（类型擦除后签名一致即可调用）。

**通俗解释**：bind 把"多参数函数"预先钉住一些参数，变成"少参数函数"；`std::function` 则把结果统一装进"标准插座"里调用。成员函数特殊在"必须挂在对象上"——bind 时把对象地址（或引用）也一并钉住。

---

### 编程题 5

> 使用bind和function实现基于对象的回调机制，定义Figure类，包含两个function成员（显示和计算面积），通过bind将不同形状（矩形、圆、三角形）的成员函数注册进去，实现多态调用（静态多态）。

**答案：**

```cpp
#include <iostream>
#include <functional>

// 门面类：内部是两个 std::function，通过 bind 注册不同形状的成员函数
class Figure {
public:
    std::function<void()> display;            // 显示回调
    std::function<double()> area;             // 面积回调
};

struct Rectangle {
    double w, h;
    Rectangle(double w, double h) : w(w), h(h) {}
    void show() const { std::cout << "矩形(" << w << "x" << h << ")"; }
    double calcArea() const { return w * h; }
};

struct Circle {
    double r;
    explicit Circle(double r) : r(r) {}
    void show() const { std::cout << "圆(r=" << r << ")"; }
    double calcArea() const { return 3.14159265 * r * r; }
};

struct Triangle {
    double a, b;
    Triangle(double a, double b) : a(a), b(b) {}
    void show() const { std::cout << "三角形(" << a << "," << b << ")"; }
    double calcArea() const { return a * b / 2.0; }
};

int main() {
    Rectangle rect(3, 4);
    Circle    cir(5);
    Triangle  tri(4, 5);

    // 用 bind 把各形状的成员函数“注册”进 Figure 的 function 成员
    Figure figs[3];
    figs[0] = Figure{std::bind(&Rectangle::show, &rect),
                     std::bind(&Rectangle::calcArea, &rect)};
    figs[1] = Figure{std::bind(&Circle::show, &cir),
                     std::bind(&Circle::calcArea, &cir)};
    figs[2] = Figure{std::bind(&Triangle::show, &tri),
                     std::bind(&Triangle::calcArea, &tri)};

    for (auto &f : figs) {
        f.display();
        std::cout << " 面积 = " << f.area() << "\n";
    }
    return 0;
}
```

输出：
```
矩形(3x4) 面积 = 12
圆(r=5) 面积 = 78.5398
三角形(4,5) 面积 = 10
```

**要点**：
- `Figure` 用 `std::function` 把"显示/算面积"抽象成统一接口；
- 不同形状通过 `bind` 把自己的成员函数（绑定对象地址）注册进去——这是**静态多态/基于对象的回调（OOP 用函数指针替代虚函数表的一种风格）**；
- `Figure` 本身**没有任何继承关系**，却实现了"同接口、异实现"的多态效果。

**通俗解释**：Figure 像一台"通用演播台"，上面有两个空按钮（display、area）。每个形状把自己的"表演节目"（成员函数）通过 bind 绑到按钮上。按下统一按钮，各自的节目自动上演——这就是"用函数对象实现的鸭子类型多态"，不需要继承和虚函数也能各显神通。

---

### 编程题 6

> 使用mem_fn适配器结合for_each和remove_if，对vector<Number>执行打印和删除偶数元素的操作。

**答案：**

```cpp
#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>

struct Number {
    int v;
    Number(int v) : v(v) {}
    void print() const { std::cout << v << " "; }
    bool isEven() const { return v % 2 == 0; }   // 供 remove_if 作谓词
};

int main() {
    std::vector<Number> nums{Number(1), Number(2), Number(3),
                             Number(4), Number(5), Number(6)};

    // mem_fn + for_each：逐个调用 Number::print
    std::for_each(nums.begin(), nums.end(), std::mem_fn(&Number::print));
    std::cout << "\n";

    // mem_fn + remove_if + erase：删除偶数
    // mem_fn(&Number::isEven) 作为谓词，返回 bool（成员函数的返回值）
    nums.erase(std::remove_if(nums.begin(), nums.end(),
                              std::mem_fn(&Number::isEven)),
               nums.end());

    std::cout << "删除偶数后: ";
    std::for_each(nums.begin(), nums.end(), std::mem_fn(&Number::print));
    std::cout << "\n";
    return 0;
}
```

输出：
```
1 2 3 4 5 6 
删除偶数后: 1 3 5 
```

**要点**：
- `mem_fn(&Number::print)`：把成员函数指针变成"以对象为第一参数"的可调用对象，for_each 恰好逐个传对象；
- `mem_fn(&Number::isEven)`：返回 `bool`（成员函数返回值），可直接作 remove_if 的谓词；
- 配合 erase-remove_if 惯用法真正删除。

**通俗解释**：mem_fn 把"成员函数"从对象身上"解耦"成独立工具：打印工具（print）让 for_each 挨个调用；判断工具（isEven）返回真假，让 remove_if 决定谁走。不用写 lambda，也不用手写函数对象，一行 `mem_fn` 就完成适配。

---

### 编程题 7

> 模拟实现一个简易的vector，使用空间配置器的allocate、deallocate、construct、destroy来管理内存和对象生命周期，不依赖标准库的allocator。

**答案：**

```cpp
#include <iostream>
#include <memory>     // 用 std::allocator 模拟"空间配置器"
#include <cassert>

// 简易 vector：只实现 push_back / operator[] / size / ~
// 使用 allocator 的 allocate/deallocate 与 construct/destroy 分开管理
template <typename T>
class MiniVector {
public:
    MiniVector() : data(nullptr), sz(0), cap(0) {}
    ~MiniVector() {
        clear();                       // 先析构所有对象
        if (data) alloc.deallocate(data, cap);   // 再释放内存
    }

    void push_back(const T& val) {
        if (sz == cap) grow();         // 扩容：重新 allocate + 移动/拷贝构造
        // 在"预留内存的 sz 位置"就地构造对象（不产生临时对象）
        std::allocator_traits<decltype(alloc)>::construct(alloc, data + sz, val);
        ++sz;
    }

    void clear() noexcept {
        for (size_t i = 0; i < sz; ++i)
            std::allocator_traits<decltype(alloc)>::destroy(alloc, data + i);
        sz = 0;
    }

    T& operator[](size_t i)       { return data[i]; }
    const T& operator[](size_t i) const { return data[i]; }
    size_t size() const { return sz; }
    size_t capacity() const { return cap; }

private:
    std::allocator<T> alloc;      // 空间配置器（这里用标准 allocator 充当）
    T* data;
    size_t sz, cap;

    void grow() {
        size_t newCap = cap ? cap * 2 : 1;
        T* newData = alloc.allocate(newCap);          // 1. 分配原始内存（未构造）
        for (size_t i = 0; i < sz; ++i) {
            // 2. 在新内存上构造（这里用拷贝构造；实际可用 move）
            std::allocator_traits<decltype(alloc)>::construct(
                alloc, newData + i, data[i]);
            // 3. 析构旧对象
            std::allocator_traits<decltype(alloc)>::destroy(alloc, data + i);
        }
        if (data) alloc.deallocate(data, cap);        // 4. 释放旧内存
        data = newData;
        cap = newCap;
    }
};

int main() {
    MiniVector<int> v;
    for (int i = 1; i <= 5; ++i) v.push_back(i * 10);
    std::cout << "size=" << v.size() << " cap=" << v.capacity() << " → ";
    for (size_t i = 0; i < v.size(); ++i) std::cout << v[i] << " ";
    std::cout << "\n";
    return 0;
}
```

**要点（对应教材的"四阶段"）**：
- **allocate**：只申请原始内存，不构造对象（`alloc.allocate(newCap)`）；
- **construct**：在指定地址就地构造（`allocator_traits::construct(alloc, ptr, val)`，底层是 placement new）——这是 `push_back`/`grow` 里"放对象"的步骤；
- **destroy**：只调析构、不动内存（`allocator_traits::destroy`）；
- **deallocate**：归还内存。

**现代写法说明**：这里用 `std::allocator_traits<...>::construct/destroy` 而非 `alloc.construct`——因为**后两者在 C++17 弃用、C++20 已删除**（这正是我作为"用时兴标准"角色的纠正）。若用 C++20，也可直接写 `std::construct_at(data + sz, val)` / `std::destroy_at(data + i)`。

**通俗解释**：这个 MiniVector 完整演示了"空间配置器四步走"：allocate=租毛坯房、construct=让住户入住、destroy=住户搬走（清空房间）、deallocate=退房。扩容时"租新房→搬家具（构造）→旧房清人（析构）→退旧房（deallocate）"，和标准 vector 的扩容一模一样，只是我们亲手用 allocator 走了一遍全程。

---

# 全卷小结（知识点地图）

| 主题                 | 核心考点                                                     |
| -------------------- | ------------------------------------------------------------ |
| Query 体系           | 智能指针+抽象基类的值语义、动态多态五条件、继承与集合运算、门面类与友元 |
| vector/deque/list    | 迭代器=指针语义、deque 中控器+缓冲区、扩容四阶段、迭代器失效、链表特殊操作 |
| set/map              | 红黑树、严格弱序、key 只读、`[]` 语义（无 const）、multimap 无 `[]` 的原因 |
| 哈希/无序容器        | 哈希函数与冲突、拉链法、装载因子、rehash、自定义 Hash/Equal 的三种方式 |
| 迭代器/适配器        | 五类迭代器、插入迭代器、流迭代器、reverse_iterator 的 base() 与 1 格偏移 |
| 算法/lambda          | erase-remove 惯用法、for_each 可调用对象、捕获列表与生命周期陷阱、mutable |
| bind/function/mem_fn | 占位符、值/引用传递、reference_wrapper、成员函数绑定、函数对象全谱、基于对象回调 |
| 空间配置器           | 内存与对象分离、SGI 两级配置器（128B/16 链表/chunk_alloc 后备链）——历史内容 |

**最后再强调三处"以新代旧"**：① `bind1st/bind2nd` 已随 C++17 删除，用 `bind` 或 lambda；② `allocator::construct/destroy` 已随 C++20 删除，用 `allocator_traits` / `std::construct_at` / `std::destroy_at`；③ SGI 二级配置器是历史教材实现，现代 `std::allocator` 走系统 malloc/operator new，原理仍需理解但不代表当前默认行为。

