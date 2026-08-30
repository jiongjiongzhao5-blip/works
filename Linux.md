# 作业一（HTTP 基础）

### 原题
> 1. 使用浏览器或抓包工具（如 Wireshark、Fiddler）访问 http://example.com，截图并标注出以下信息：请求行、请求头、空行、请求体（若有）；响应行、响应头、响应体。
> 2. 请从"协议"、"传输"、"超文本"三个字段解释 HTTP 的中文全称及其特点。
> 3. 请说明一个完整 URI 由哪几部分组成？并举例说明每部分的作用。
> 4. 对比 GET 和 POST 请求方式的区别，至少从语义、参数位置、安全性、幂等性四个方面回答。
> 5. 列出常见的 HTTP 状态码类别（1xx~5xx）及其代表含义，并各举一个具体状态码。

### 解答

**第 1 题：抓包标注**

我无法代替你截图，下面给出你会在工具里看到的**真实报文内容**和**标注方法**。访问 `http://example.com` 后，Wireshark 过滤栏输入 `http`，选中一个请求包，右键 → **Follow → TCP Stream**，你会看到原始文本，按下面逐段标注：

```
GET / HTTP/1.1                              ← 请求行（方法 + URI + 版本）
Host: example.com                           ← 请求头
User-Agent: Mozilla/5.0 (…)
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, deflate
Connection: keep-alive
                                            ← 空行（请求头与请求体之间，\r\n）
(此处无内容)                                 ← 请求体（GET 请求通常没有请求体）
```
```
HTTP/1.1 200 OK                             ← 响应行（版本 + 状态码 + 原因短语）
Content-Type: text/html; charset=UTF-8      ← 响应头
Content-Length: 1256
Server: ECS (…)
Date: … 
                                            ← 空行
<!doctype html>                             ← 响应体（HTML 页面）
<html>…example.com…</html>
```

标注方法：在 Wireshark 里 **Follow TCP Stream** 看到的请求在上、响应在下，用文本框/箭头把上面六类成分圈出来即可。Fiddler 则在左侧点开请求后，右侧 **Inspectors → Headers/TextView** 分别看请求头、响应头和响应体。另外注意：**报文里的"空行"是一个 `\r\n`**，它的作用是分隔"头部"和"主体"，在 HTTP 语法里是必须有的。

**第 2 题：HTTP 全称解析**

HTTP = **H**yper**T**ext **T**ransfer **P**rotocol = **超文本传输协议**，从三个词解释：

- **超文本（HyperText）**：传输的内容不限于纯文本，而是包含**超链接**、图片、音频、视频、样式等多媒体的"超"文本，还能通过链接从一个页面跳到另一个页面。HTTP 可以传输任意类型数据（由 `Content-Type` 声明）。
- **传输（Transfer）**：数据按"**请求—响应**"模式在客户端与服务器之间传输。客户端发请求，服务器回响应；HTTP 默认运行在 TCP 之上（HTTP/1.0/1.1/2 基于 TCP，HTTP/3 基于 UDP 上的 QUIC）。
- **协议（Protocol）**：一套双方共同遵守的**规则/约定**，规定了报文格式（请求行/请求头/空行/请求体）、请求方法（GET/POST…）、状态码（200/404…）等。没有这套约定，浏览器和服务器就没法互相理解。

HTTP 的主要特点：**简单快速**（协议简单，性能好）、**无状态**（默认不记忆历史请求，用 Cookie/Session/Token 补记）、**明文传输**（HTTP 本身不加密，HTTPS 才是加密版）、**灵活**（支持任意媒体类型）、**基于请求-响应**。

**第 3 题：URI 的组成**

一个完整 URI 的通用格式：

```
scheme://userinfo@host:port/path?query#fragment
```

以 `http://user:pwd@example.com:8080/news/2024?id=10&page=2#top` 为例：

| 部分                 | 示例            | 作用                               |
| -------------------- | --------------- | ---------------------------------- |
| scheme（协议）       | `http`          | 告诉客户端用什么协议访问           |
| userinfo（用户信息） | `user:pwd@`     | 身份认证信息（实际少用）           |
| host（主机）         | `example.com`   | 服务器域名或 IP                    |
| port（端口）         | `8080`          | 服务器监听的端口（默认 80 可省略） |
| path（路径）         | `/news/2024`    | 定位服务器上的具体资源             |
| query（查询串）      | `?id=10&page=2` | 以 `?` 开头、`&` 分隔的键值对参数  |
| fragment（片段）     | `#top`          | 定位页面内锚点，**不会**发给服务器 |

补充概念：**URI ⊃ URL ⊃ URN**。URI（统一资源标识符）只要求"唯一标识"，URL（统一资源定位符）是其中"能定位到资源"的一种，日常说的网址基本都是 URL。

**第 4 题：GET vs POST**

| 对比项       | GET                                                          | POST                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **语义**     | 获取/查询资源，**不应**改变服务器状态                        | 提交/创建资源，通常**会**改变服务器状态                      |
| **参数位置** | 参数拼在 **URL 查询串**里（`?a=1&b=2`）                      | 参数放在**请求体**（Body）里                                 |
| **安全性**   | 参数在 URL 中，会出现在**浏览器历史、书签、服务器日志、代理日志**里，泄露风险高 | 参数在 Body 里，相对不易被看到；**但都是明文**，真正安全必须用 HTTPS |
| **幂等性**   | **幂等**：执行 N 次与 1 次结果相同（只读）                   | **非幂等**：重复提交可能产生多条记录/重复扣款                |
| 其他         | URL 长度受限（约 2KB~8KB 取决于服务器）、可被缓存、可收藏    | 长度理论上不限、不易缓存、不收藏                             |

**第 5 题：状态码分类**

| 类别 | 含义                           | 例子                                                         |
| ---- | ------------------------------ | ------------------------------------------------------------ |
| 1xx  | 信息性，请求已收到，继续处理   | **100 Continue**                                             |
| 2xx  | 成功，请求已被成功处理         | **200 OK**                                                   |
| 3xx  | 重定向，需要进一步操作完成请求 | **301** 永久重定向 / **304** 未修改（用缓存）                |
| 4xx  | 客户端错误，请求有误           | **404 Not Found**（还有 400 语法错、401 未认证、403 禁止、429 太频繁） |
| 5xx  | 服务端错误，服务器处理失败     | **500 Internal Server Error**（还有 502 网关错、503 服务不可用、504 超时） |

### 通俗解释
HTTP 就像"写信—回信"：请求报文是**你写的信**（收信地址、称呼、正文分节），响应报文是**对方的回信**；空行是信纸上的"抬头与正文之间的空行"，用来分隔头尾；状态码就是"回执上的结果说明"（已投递 200 / 地址错误 404 / 邮局罢工 500）。GET 像"去查资料"，POST 像"交报名表"——前者重复做没副作用（幂等），后者重复交可能重复登记。

> 🆕 时兴技术点：HTTP 已演进到 **HTTP/2**（多路复用、二进制帧、头部压缩）与 **HTTP/3（QUIC）**；明文 HTTP 已基本退出生产环境，默认使用 **HTTPS**；跨平台命令行抓包/调试推荐 `curl -v`、`mitmproxy`。

---

# 作业二（OSI 与 TCP/IP）

### 原题
> 1. 绘制 OSI 七层模型与 TCP/IP 四层模型的对应关系图，并写出每一层的名称及主要功能。
> 2. 列出至少 6 个 TCP/IP 协议族中的常见协议，指明它们分别属于哪一层，并说明其基本功能及默认端口（如果有）。
> 3. 简述 TCP 协议如何通过"确认与重传"机制保证数据可靠到达，可举例说明。
> 4. 解释 TCP 滑动窗口机制的作用，它如何实现流量控制？
> 5. 思考：为什么 TCP 需要重传定时器？如果超时后未收到确认，TCP 会怎么做？

### 解答

**第 1 题：模型对应关系**

```
    OSI 七层模型                TCP/IP 四层模型
┌───────────────────┐
│ 7 应用层           │──┐
│ 6 表示层           │  ├──→ 应用层   (HTTP/FTP/DNS/SMTP…)
│ 5 会话层           │──┘
├───────────────────┤
│ 4 传输层           │────→ 传输层   (TCP/UDP)
├───────────────────┤
│ 3 网络层           │────→ 网际层   (IP/ICMP/ARP…)
├───────────────────┤
│ 2 数据链路层       │──┐
│ 1 物理层           │──┴→ 网络接口层(以太网/PPP…)
└───────────────────┘
```

各层名称与功能：

| OSI 层 | 名称       | 主要功能                                        |
| ------ | ---------- | ----------------------------------------------- |
| 7      | 应用层     | 为用户提供网络服务接口（HTTP、FTP、DNS、SMTP）  |
| 6      | 表示层     | 数据格式转换、加密、压缩（TLS/SSL 前身功能）    |
| 5      | 会话层     | 建立、管理、终止会话                            |
| 4      | 传输层     | 端到端传输，提供可靠(TCP)/不可靠(UDP)服务、端口 |
| 3      | 网络层     | 逻辑寻址（IP）与路由选择                        |
| 2      | 数据链路层 | 相邻节点间成帧、MAC 寻址、差错控制              |
| 1      | 物理层     | 比特流的物理传输（电压、光信号、线缆）          |

**第 2 题：TCP/IP 常见协议**

| 协议        | 层                | 功能                   | 默认端口            |
| ----------- | ----------------- | ---------------------- | ------------------- |
| HTTP        | 应用层            | 网页传输               | 80                  |
| HTTPS       | 应用层            | 加密的网页传输         | 443                 |
| FTP         | 应用层            | 文件传输               | 21（数据 20）       |
| SMTP        | 应用层            | 发送邮件               | 25                  |
| POP3 / IMAP | 应用层            | 接收邮件               | 110 / 143           |
| DNS         | 应用层            | 域名解析为 IP          | 53                  |
| DHCP        | 应用层            | 自动分配 IP            | 67/68               |
| SSH         | 应用层            | 安全远程登录           | 22                  |
| Telnet      | 应用层            | 明文远程登录           | 23                  |
| SNMP        | 应用层            | 网络管理               | 161                 |
| TCP         | 传输层            | 面向连接的可靠传输     | —（端口由上层分配） |
| UDP         | 传输层            | 无连接的不可靠传输     | —                   |
| IP          | 网际层            | 逻辑寻址与路由         | —                   |
| ICMP        | 网际层            | 差错报告与探测（ping） | —                   |
| ARP         | 链路层/网络层之间 | 把 IP 解析成 MAC 地址  | —                   |
| OSPF / BGP  | 网际层            | 路由协议               | —                   |

**第 3 题：确认与重传机制**

TCP 的可靠传输靠"**序号 + 确认 + 超时重传**"三件套：

1. 发送方给每个字节编号（seq），发送一个报文段后**启动定时器**；
2. 接收方收到后返回 **ACK（确认号 = 下一个期望字节的序号）**；
3. 发送方若在超时时间内没收到该 ACK，就**重传**该报文段；收到 ACK 就继续发后面的。

举例：A 发送 `seq=1000`、长度 200 字节的数据段，期望 B 回 `ack=1200`。若 B 没收到或 ACK 丢了，A 的重传定时器到期后重发该段，直到收到确认。注意 TCP 的确认是**累积确认**：ack 表示"此序号之前的所有字节我都收到了"。

**第 4 题：滑动窗口与流量控制**

- **滑动窗口**：发送方维护一个**发送窗口**，只有落在窗口内的数据才能发送；数据被 ACK 后窗口就"向前滑动"，从而源源不断地流水式发送，避免"发一包等一包"的低效率（即流水线/管道化）。
- **流量控制**：接收方在自己的 TCP 头里携带**通告窗口 rwnd**（"我还能接收多少字节"）。发送方的发送窗口大小取 **min(rwnd, 拥塞窗口 cwnd)**，从而**限制发送速率，防止接收方缓冲区溢出**。这是"接收端控制发送端"的流量控制。
- 补充：拥塞控制（慢启动、拥塞避免、快速重传/快速恢复）则是"根据网络状况"控制发送速率的另一套机制，靠 cwnd 实现，两者共同决定实际发送窗口。

**第 5 题：重传定时器**

- **为什么需要**：网络可能丢包、延迟、乱序，发送方必须知道"数据到底有没有到"。靠**超时**来兜底——如果在规定时间内没等到 ACK，就认为丢包并重传。
- **超时后 TCP 做什么**：
  1. 重传该报文段；
  2. **RTO（重传超时时间）指数退避**（超时时间翻倍，如 1s→2s→4s…），避免网络拥塞时反复碰撞；
  3. **拥塞窗口 cwnd 减为 1（或减半）**、慢启动阈值 ssthresh 减半，进入拥塞控制流程；
  4. 更快的做法是**快速重传**：收到 **3 个重复 ACK** 时不等超时立即重传（此时大概率只是丢包而非拥塞）。

### 通俗解释
OSI/TCP-IP 模型就像**邮递系统分部门**：应用层是"写信人"，传输层是"邮局分拣中心"（保证包裹到没到、要不要补寄），网络层是"地址导航"（IP 路由），链路层是"快递员"（按门牌号/MAC 送上门），物理层是"马路和货车"。TCP 的"确认+重传"好比快递**签收回执**：寄件人没收到回执就再寄一份；滑动窗口好比"**不等上一单签收，批量发好几单**"，而接收方用 rwnd 说"我这仓库只剩这么多位子"，寄件人就不会一次塞爆仓库。

---

# 作业三（TCP 握手挥手与字节序）

### 原题
> 1. 详细描述 TCP 三次握手的每一步（包括 SYN、ACK 序列号变化），并说明为什么需要三次握手，两次为什么不行？（请用反例论证）
> 2. 详细描述 TCP 四次挥手的每一步，并解释为什么断开连接需要四次，而非三次？主动关闭方为何要进入 TIME_WAIT 状态？其等待时间 2MSL 有何作用？
> 3. 使用 Wireshark 抓取一次 TCP 连接建立和断开的过程，截屏并标注三次握手和四次挥手的每个报文。
> 4. 解释"大端模式"和"小端模式"的含义，并用 C 语言编写一段代码，检测你当前机器的字节序（提示：使用 union 或指针强转）。
> 5. 说明"主机字节序"和"网络字节序"的区别，为什么需要转换？写出常用转换函数（如 htonl、ntohl 等）的作用。

### 解答

**第 1 题：三次握手**

设客户端 A 初始序号 x，服务器 B 初始序号 y：

| 步骤 | 方向 | 报文                      | 含义                                                         |
| ---- | ---- | ------------------------- | ------------------------------------------------------------ |
| 1    | A→B  | `SYN, seq=x`              | A 请求建立连接，并通告自己的初始序号 x                       |
| 2    | B→A  | `SYN+ACK, seq=y, ack=x+1` | B 同意连接（SYN），同时确认 A 的序号（ack=x+1），并通告自己的序号 y |
| 3    | A→B  | `ACK, seq=x+1, ack=y+1`   | A 确认 B 的序号                                              |

第 3 步后双方进入 ESTABLISHED，可传数据。

**为什么必须三次、两次不行（反例论证）**：
- 两次握手只能保证 B 的序号被 A 收到，却**无法让 B 确认 A 的序号已被 A 自己收到**（B 需要 A 回一条 ACK）。
- **经典反例（防止过期连接）**：A 发出 SYN 后该报文在网络中滞留，A 超时重发 SYN，并成功完成连接、传完数据、正常关闭。**此时旧的 SYN 才姗姗到达 B**。若只用两次握手，B 一收到旧 SYN 就盲目建立连接、分配资源，可这条请求早已作废——这就是"**僵尸连接**"浪费资源。而三次握手下，A 收到这条旧 SYN 的回应后，发现与自己无关，会发 **RST** 拒绝，B 随即释放资源。
- 一句话：三次握手同时完成了**"双向同步初始序号" + "双向确认对方收发能力" + "规避过期的连接请求"**。

**第 2 题：四次挥手**

设 A 主动关闭：

| 步骤 | 方向 | 报文           | 含义                                        |
| ---- | ---- | -------------- | ------------------------------------------- |
| 1    | A→B  | `FIN, seq=u`   | A 说"我没有数据要发了"                      |
| 2    | B→A  | `ACK, ack=u+1` | B 确认，但 B **可能还有数据要发** → 半关闭  |
| 3    | B→A  | `FIN, seq=w`   | B 数据发完了，也说"我也没有要发的了"        |
| 4    | A→B  | `ACK, ack=w+1` | A 确认；B 进入 CLOSED，A 进入 **TIME_WAIT** |

**为什么是四次而非三次**：因为 TCP 是**全双工**，两个方向的连接要**分别关闭**。A 发 FIN 只代表"A→B 方向没数据了"，但 B→A 方向 B 可能还有数据要发；只有 B 也发完自己的数据后，再发自己的 FIN，才表示两个方向都关闭。第 2 步的 ACK 和第 3 步的 FIN 被**拆分成了两个报文**（不能合并，因为第 2 步时 B 还没发完数据），所以是四次。

**为什么主动关闭方要等 2MSL**（MSL = 报文最长存活时间）：
1. **保证最后一个 ACK 能送达**：若第 4 步的 ACK 丢失，B 会重发 FIN，A 必须仍保留连接状态以重发 ACK。2MSL 覆盖"ACK 到达 B + B 重发 FIN 再到达 A"的最坏往返。
2. **让旧连接残留报文在网络中彻底消失**：防止它们被误认为**新连接**（尤其端口被立即复用时）的报文。
3. 数值：RFC 建议 MSL=2 分钟，2MSL=4 分钟；**Linux 内核实际把 MSL 定为 30s，TIME_WAIT 为 60s**。高并发服务可用 `SO_REUSEADDR` / `SO_REUSEPORT` 或调内核参数缓解 TIME_WAIT 堆积。

**第 3 题：Wireshark 观察握手/挥手**

步骤：选择网卡 → 开始抓包 → 用浏览器/`curl` 访问某网站 → 停止 → 过滤 `tcp`（或 `tcp.port==443/80`）→ 找到 `SYN` 包，右键 **Follow TCP Stream**。
- **三次握手**：流开头的连续 3 个包，标志位依次为 `[SYN]`、`[SYN, ACK]`、`[ACK]`，seq/ack 正好对应上面的 x、x+1、y、y+1。
- **四次挥手**：流末尾的连续 4 个包，标志位依次为 `[FIN, ACK]`、`[ACK]`、`[FIN, ACK]`、`[ACK]`。
把每个包用箭头标出方向 + 标志位 + seq/ack 即可。

**第 4 题：大小端检测**

- **大端（Big-Endian）**：高字节存低地址，符合人类书写习惯（`0x12345678` 内存顺序 `12 34 56 78`）。
- **小端（Little-Endian）**：低字节存低地址（`78 56 34 12`）。x86/x86-64 主机默认**小端**。

C 代码（union 版 + 指针强转版）：

```c
#include <stdio.h>

int main() {
    // 方法1：union 共享同一块内存
    union {
        int i;
        unsigned char c[4];
    } u;
    u.i = 0x01020304;                       // 写入一个各字节都不同的数
    if (u.c[0] == 0x01)
        printf("大端 (Big-Endian)\n");
    else if (u.c[0] == 0x04)
        printf("小端 (Little-Endian)\n");

    // 方法2：指针强转
    int x = 0x01020304;
    unsigned char *p = (unsigned char *)&x;
    if (*p == 0x01)
        printf("指针法：大端\n");
    else if (*p == 0x04)
        printf("指针法：小端\n");
    return 0;
}
```

（在 x86 机器上会输出"小端"。）

**第 5 题：主机序 vs 网络序**

- **主机字节序**：本机 CPU 存放多字节整数的方式（x86 小端，网络设备多是大端）。
- **网络字节序**：**统一规定为大端**，保证不同架构机器间数据含义一致。
- **为什么要转换**：同样一个 `0x1234`，小端机器存成 `34 12`，大端机器存成 `12 34`；若不转换，A 发过去 B 读出来的数值就不对。所以**凡是填入网络报文/套接字结构体的整数（IP、端口、长度），都要转成网络序**。

| 函数       | 作用                                          |
| ---------- | --------------------------------------------- |
| `htonl(h)` | 主机序 32 位 → 网络序（如 IP 地址 `in_addr`） |
| `htons(h)` | 主机序 16 位 → 网络序（如端口号）             |
| `ntohl(n)` | 网络序 32 位 → 主机序                         |
| `ntohs(n)` | 网络序 16 位 → 主机序                         |

典型用法：`srv.sin_port = htons(8888); srv.sin_addr.s_addr = htonl(INADDR_ANY);`

### 通俗解释
三次握手像**打电话前的三方确认**："喂，能听到吗？"(SYN) →"能听到，你听到我吗？"(SYN+ACK) →"听到，开聊"(ACK)，双方确认"听得到也说得出"再开始。四次挥手像**挂电话**："我说完了"(FIN) →"知道"(ACK) →"我也说完了"(FIN) →"好"(ACK)，多等 2MSL 是为了确保最后那句"好"对方真的收到、且余音（残留报文）散尽再占线。大小端就像"数字倒着写还是正着写"，网络统一"正着写"，htonl/ntohl 就是"翻译器"。

---

# 作业四（socket 基础 + Echo 服务器）

### 原题
> 1. 分别简述 socket()、bind()、listen()、accept()、connect() 五个函数的主要功能及调用时机。
> 2. 当调用 send() 函数时，数据从用户态到网卡发送经历了哪些过程？recv() 又是如何从网卡接收数据到用户缓冲区的？
> 3. 编写一个最简单的 TCP 回射服务器（Echo Server）：客户端发送任意字符串，服务器原样返回。要求服务器能循环处理一个客户端的多次发送，且客户端可从标准输入读取。
> 4. 将你的回射服务器修改为并发处理多个客户端（可用多线程或 fork，但不必用 select/epoll），至少能同时服务两个客户端。

### 解答

**第 1 题：五个函数**

| 函数        | 功能                                                         | 调用时机/方                                            |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| `socket()`  | 创建套接字，返回 fd                                          | 通信第一步，双方都要调用                               |
| `bind()`    | 把套接字绑定到本机 IP:端口                                   | **服务器**必调（固定端口）；客户端可选（内核自动分配） |
| `listen()`  | 把套接字设为被动监听，进入等待连接队列                       | **服务器**在 bind 之后、accept 之前调用                |
| `accept()`  | 从已完成三次握手的队列中取出一个连接，**返回一个新的连接套接字** | **服务器**循环调用                                     |
| `connect()` | 发起三次握手，主动连接服务器                                 | **客户端**调用                                         |

要点：`accept` 返回的新套接字才是与具体客户端通信的 fd，监听套接字（listen fd）继续负责"接新连接"。

**第 2 题：send/recv 的数据路径**

`send()` 的发送过程（用户态 → 网卡）：
```
用户缓冲区 ──send()──> 内核发送缓冲(socket buffer)
   ──TCP 层：切分报文段、加序号/端口、计算校验和──>
   ──IP 层：封装 IP 头、路由──>
   ──网卡驱动：封装以太网帧、写入发送队列──> 网卡发出
```
注意：**阻塞式 `send()` 返回时只代表数据已拷贝进内核发送缓冲**，并不代表已发到对端；真正的发送由内核异步完成。

`recv()` 的接收过程（网卡 → 用户态）：
```
网卡收到帧 ──> 网卡驱动/中断、DMA 拷入内核接收缓冲
   ──链路层/IP/TCP 逐层解封装、校验、重组──>
   ──数据进入该套接字的接收缓冲──recv()──> 拷贝到用户缓冲区
```
所以 `recv()` 本质是从**内核的接收缓冲**里"取"数据，一次能取多少取决于缓冲里已有多少。

**第 3 题：最简单的 TCP 回射服务器 + 客户端**

服务器（循环服务一个客户端，支持多次收发）：

```c
// echo_server.c —— 单客户端版
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    int cfd = accept(lfd, NULL, NULL);        // 只服务第一个客户端
    char buf[1024];
    ssize_t n;
    while ((n = recv(cfd, buf, sizeof(buf), 0)) > 0) {
        send(cfd, buf, n, 0);                 // 原样回射
    }
    close(cfd);
    close(lfd);
    return 0;
}
```

客户端（从标准输入读取，循环收发）：

```c
// echo_client.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main() {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in srv;
    memset(&srv, 0, sizeof(srv));
    srv.sin_family = AF_INET;
    srv.sin_port   = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);

    connect(sfd, (struct sockaddr *)&srv, sizeof(srv));

    char buf[1024];
    while (fgets(buf, sizeof(buf), stdin)) {  // 从标准输入读一行
        send(sfd, buf, strlen(buf), 0);
        int n = recv(sfd, buf, sizeof(buf), 0);
        if (n <= 0) break;
        fwrite(buf, 1, n, stdout);            // 打印回射内容
    }
    close(sfd);
    return 0;
}
```

**第 4 题：并发版（fork 多进程）**

```c
// echo_server_concurrent.c —— fork 多进程并发
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/wait.h>

void sigchld_handler(int sig) {               // 回收子进程，避免僵尸
    while (waitpid(-1, NULL, WNOHANG) > 0);
}

int main() {
    signal(SIGCHLD, sigchld_handler);
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    while (1) {
        int cfd = accept(lfd, NULL, NULL);    // 阻塞等待新连接
        if (fork() == 0) {                    // 子进程
            close(lfd);                       // 子进程不需要监听 fd
            char buf[1024];
            ssize_t n;
            while ((n = recv(cfd, buf, sizeof(buf), 0)) > 0)
                send(cfd, buf, n, 0);         // 回射
            close(cfd);
            _exit(0);                         // 子进程直接退出
        }
        close(cfd);                           // 父进程关闭客户 fd，继续 accept
    }
    return 0;
}
```

用**多线程**替代 fork 也一样可行：`accept` 后 `pthread_create` 派一个线程处理该连接（注意线程共享 fd 表，子线程里同样 `recv/send`，主线程继续 accept）。线程开销小于进程，但要注意共享变量加锁。

### 通俗解释
`socket/bind/listen/accept/connect` 就是"**装电话机**"：socket 是买电话机（建 fd），bind 是给电话机分配一个**固定号码**（端口），listen 是"待机接听"，accept 是"**接起一通来电**"并给每通来电一个专用话筒（新 fd），connect 是"**主动拨号**"。fork 并发版好比"每来一个电话就雇一名话务员专职接听"，可以同时服务很多人。

> 🆕 时兴技术点：现代并发更常用**线程池 + epoll** 或 C++20 **协程**（`co_await`），性能远超"一连接一进程"；Linux 上还可研究 **io_uring** 实现零拷贝异步 I/O。

---

# 作业五（select / 聊天室 / epoll）

### 原题
> 1. 简述 select() 函数的作用及参数含义，并说明其局限性（如文件描述符上限、效率问题等）。
> 2. 使用 select 实现一个简单的聊天室服务端：可以接受多个客户端连接，并将任一客户端发送的消息广播给所有其他客户端。客户端可同时发送和接收消息（使用 select 监听标准输入和 socket）。
> 3. 对比 select 与 epoll 的异同点（从接口、效率、触发方式等方面）。
> 4. （选做）将上述聊天室服务端改为 epoll 实现。

### 解答

**第 1 题：select 作用、参数与局限**

```c
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

- `nfds`：**最大 fd 值 + 1**；
- `readfds`：要监视"可读"的 fd 集合，**传入传出**（返回时变成"就绪集合"）；
- `writefds` / `exceptfds`：分别监视可写/异常，通常传 NULL；
- `timeout`：NULL 表示永久阻塞；`{0,0}` 表示轮询不阻塞；`{2,0}` 表示最多等 2 秒；
- 返回值：就绪 fd 总数；0 表示超时；-1 表示出错。

配套宏：`FD_ZERO(&set)` 清空、`FD_SET(fd,&set)` 加入、`FD_CLR(fd,&set)` 移除、`FD_ISSET(fd,&set)` 判断是否就绪。

**局限性**：
1. **fd 数量上限**：受 `FD_SETSIZE`（通常 **1024**）限制；
2. **每次调用都要把整个 fd_set 从用户态拷贝进内核**，连接多时开销大；
3. **O(n) 遍历**：内核要遍历所有 fd 检查状态；返回后用户还要再遍历一遍找就绪项，且只告诉"有几个"不告诉"是哪些"；
4. **fd_set 被内核改写**，每次循环必须重新构造，编码繁琐易错；
5. 触发模式单一（只有水平触发），事件粒度粗。

**第 2 题：select 版聊天室**

服务端（接受多客户端 + 广播给"其他"客户端）：

```c
// chat_server_select.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

#define MAX_CLIENTS 100

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    int clients[MAX_CLIENTS], nclients = 0;   // 已连接客户端 fd 表
    fd_set allset, rset;
    FD_ZERO(&allset);
    FD_SET(lfd, &allset);
    int maxfd = lfd;

    while (1) {
        rset = allset;                        // select 会改写，须每次重设
        if (select(maxfd + 1, &rset, NULL, NULL, NULL) < 0) continue;

        if (FD_ISSET(lfd, &rset)) {           // 新连接
            int cfd = accept(lfd, NULL, NULL);
            FD_SET(cfd, &allset);
            if (cfd > maxfd) maxfd = cfd;
            clients[nclients++] = cfd;
        }
        for (int i = 0; i < nclients; i++) {  // 遍历找就绪的客户端
            int fd = clients[i];
            if (FD_ISSET(fd, &rset)) {
                char buf[1024];
                int n = recv(fd, buf, sizeof(buf), 0);
                if (n <= 0) {                 // 断开/出错
                    close(fd);
                    FD_CLR(fd, &allset);
                    clients[i] = clients[--nclients];  // 末位补入
                    i--;
                } else {
                    // 广播给“其他”所有客户端（不含发送者自己）
                    for (int j = 0; j < nclients; j++) {
                        if (clients[j] != fd)
                            send(clients[j], buf, n, 0);
                    }
                }
            }
        }
    }
    return 0;
}
```

客户端（select 同时监听**标准输入**和 **socket**，可边发边收）：

```c
// chat_client_select.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

int main() {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in srv;
    srv.sin_family = AF_INET;
    srv.sin_port   = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);
    connect(sfd, (struct sockaddr *)&srv, sizeof(srv));

    fd_set rset;
    char buf[1024];
    while (1) {
        FD_ZERO(&rset);
        FD_SET(0, &rset);       // 标准输入可读（用户敲键盘）
        FD_SET(sfd, &rset);     // socket 可读（收到广播）
        int maxfd = sfd > 0 ? sfd : 0;

        if (select(maxfd + 1, &rset, NULL, NULL, NULL) < 0) break;

        if (FD_ISSET(0, &rset)) {             // 用户在终端输入
            if (fgets(buf, sizeof(buf), stdin) == NULL) break;
            send(sfd, buf, strlen(buf), 0);
        }
        if (FD_ISSET(sfd, &rset)) {           // 收到别人消息
            int n = recv(sfd, buf, sizeof(buf), 0);
            if (n <= 0) break;                // 服务器关闭
            fwrite(buf, 1, n, stdout);
        }
    }
    close(sfd);
    return 0;
}
```

**第 3 题：select 与 epoll 对比**

| 对比项          | select                | epoll                                                     |
| --------------- | --------------------- | --------------------------------------------------------- |
| 接口            | 单函数 + fd_set       | `epoll_create1` / `epoll_ctl` / `epoll_wait` 三个系统调用 |
| fd 上限         | FD_SETSIZE（约 1024） | 很大（取决于系统内存/文件句柄数）                         |
| 内核—用户态拷贝 | 每次整体拷贝 fd_set   | 注册一次，只返回就绪事件                                  |
| 效率            | O(n) 遍历全部 fd      | O(就绪数)，近似 O(1)                                      |
| 触发方式        | 仅水平触发（LT）      | **LT（默认）**与 **ET（边缘触发）** 两种                  |
| 平台            | Windows/Linux 均可用  | **仅 Linux**；macOS/BSD 用 kqueue                         |

**第 4 题（选做）：epoll 版聊天室服务端**

```c
// chat_server_epoll.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

#define MAX_EVENTS 1024
#define MAX_CLIENTS 100

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    int epfd = epoll_create1(0);
    struct epoll_event ev, events[MAX_EVENTS];
    ev.events = EPOLLIN;          // 监听可读事件（水平触发 LT）
    ev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);

    int clients[MAX_CLIENTS], nclients = 0;

    while (1) {
        int nready = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;

            if (fd == lfd) {                       // 新连接
                int cfd = accept(lfd, NULL, NULL);
                ev.events = EPOLLIN;
                ev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);   // 注册进 epoll
                clients[nclients++] = cfd;
            } else {                               // 客户端有数据
                char buf[1024];
                int n = recv(fd, buf, sizeof(buf), 0);
                if (n <= 0) {                      // 断开
                    close(fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    for (int k = 0; k < nclients; k++)
                        if (clients[k] == fd) { clients[k] = clients[--nclients]; break; }
                } else {                           // 广播
                    for (int j = 0; j < nclients; j++)
                        if (clients[j] != fd)
                            send(clients[j], buf, n, 0);
                }
            }
        }
    }
    return 0;
}
```

### 通俗解释
select 像"**老师挨个点名**"：不管学生有没有事都从头点名到尾（O(n)），点名册上限 1024 人，每次点名还要复印一份。epoll 像"**教室门口装了签到机**"：谁来了自动登记，老师只处理登记的（O(1)），签到机容量大得多。聊天室广播逻辑则是"你说话，别人都听见"——服务器收到谁的发言，就转发给除他以外的所有人。

> 🆕 时兴技术点：Linux 高并发目前以 epoll 为事实标准；**io_uring**（5.1+）提供更彻底的异步 + 零拷贝，适合极致性能场景；跨平台可直接用 libuv / Boost.Asio 等封装好的事件循环。

---

# 作业六（超时踢出 + 文件上传 + 大文件）

### 原题
> 1. 在上一天聊天室的基础上，增加超时踢出功能：每个客户端若连续 30 秒未发送任何消息，则服务器主动断开与其的连接，并通知其他用户。（提示：使用 time() 记录每个客户端的最后一次收发时间，在服务器主循环中定期检查。）
> 2. 编写程序实现客户端上传一个本地文本文件（如 upload.txt）到服务器，服务器接收后保存在当前目录下，文件名、大小、内容需保持一致。要求传输前先发送文件名和文件长度（可使用自定义协议）。
> 3. 请思考：如果文件较大（超过 1MB），一次性 read 再 send 可能带来什么问题？应如何改进？

### 解答

**第 1 题：超时踢出**

思路：给每个客户端配一个 `last_active` 时间戳（`time(NULL)` 秒级即可）；每次收到它的消息就刷新；`select` 的 `timeout` 设为较短间隔（如 1~5 秒），主循环周期性扫描：若 `now - last_active > 30` 则主动 `close` 并广播"xx 因超时被踢出"。

```c
// chat_server_timeout.c —— 在 select 聊天室基础上加超时踢出
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

#define MAX_CLIENTS 100
#define IDLE_LIMIT  30      // 连续 30 秒无消息则踢出

struct client {
    int fd;
    time_t last;            // 最后一次活动时间
};

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    struct client clients[MAX_CLIENTS];
    int nclients = 0;

    fd_set allset, rset;
    FD_ZERO(&allset);
    FD_SET(lfd, &allset);
    int maxfd = lfd;

    struct timeval tv;
    while (1) {
        tv.tv_sec  = 5;                 // 每 5 秒醒来检查一次空闲
        tv.tv_usec = 0;
        rset = allset;
        if (select(maxfd + 1, &rset, NULL, NULL, &tv) < 0) continue;

        if (FD_ISSET(lfd, &rset)) {     // 新连接
            int cfd = accept(lfd, NULL, NULL);
            FD_SET(cfd, &allset);
            if (cfd > maxfd) maxfd = cfd;
            clients[nclients].fd   = cfd;
            clients[nclients].last = time(NULL);
            nclients++;
        }

        for (int i = 0; i < nclients; i++) {
            int fd = clients[i].fd;
            if (FD_ISSET(fd, &rset)) {
                char buf[1024];
                int n = recv(fd, buf, sizeof(buf), 0);
                if (n <= 0) {           // 断开
                    close(fd); FD_CLR(fd, &allset);
                    clients[i] = clients[--nclients]; i--;
                } else {
                    clients[i].last = time(NULL);   // 刷新活跃时间
                    for (int j = 0; j < nclients; j++)
                        if (clients[j].fd != fd)
                            send(clients[j].fd, buf, n, 0);
                }
            }
        }

        // 周期性检查：超时踢出
        time_t now = time(NULL);
        for (int i = 0; i < nclients; i++) {
            if (now - clients[i].last > IDLE_LIMIT) {
                char msg[128];
                snprintf(msg, sizeof(msg), "client[fd=%d] 超时被踢出\n", clients[i].fd);
                for (int j = 0; j < nclients; j++)
                    if (clients[j].fd != clients[i].fd)
                        send(clients[j].fd, msg, strlen(msg), 0);
                close(clients[i].fd);
                FD_CLR(clients[i].fd, &allset);
                clients[i] = clients[--nclients];
                i--;
            }
        }
    }
    return 0;
}
```

> 🆕 精度提示：秒级 `time()` 适合"30 秒"这种粗粒度场景；若要做毫秒级心跳超时，C++ 里用 `std::chrono::steady_clock` 更合适（单调时钟，不受系统改时间影响）。

**第 2 题：文件上传（自定义协议）**

自定义协议设计：**固定 64 字节文件名（不足补 `\0`）+ 8 字节文件大小（网络字节序/大端）+ 文件内容（分批传输）**。先写一个 `recvn` 与 `send_all` 保证"收满/发完"（`recvn` 详见作业七）。

```c
// upload_client.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int send_all(int fd, const void *buf, int len) {
    const char *p = buf; int sent = 0;
    while (sent < len) {
        int n = send(fd, p + sent, len - sent, 0);
        if (n < 0) return -1;
        sent += n;
    }
    return sent;
}

int main() {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in srv;
    srv.sin_family = AF_INET; srv.sin_port = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);
    connect(sfd, (struct sockaddr *)&srv, sizeof(srv));

    const char *fname = "upload.txt";
    FILE *fp = fopen(fname, "rb");
    if (!fp) return 1;

    // 1) 发送文件名（固定 64 字节）
    char name[64] = {0};
    strncpy(name, fname, sizeof(name) - 1);
    send_all(sfd, name, 64);

    // 2) 发送文件大小（8 字节，大端）
    fseek(fp, 0, SEEK_END); long long fsize = ftell(fp); fseek(fp, 0, SEEK_SET);
    uint64_t sz = (uint64_t)fsize, be = htobe64(sz);   // 主机序转大端
    send_all(sfd, &be, 8);

    // 3) 分批发送内容
    char buf[8192]; size_t n;
    while ((n = fread(buf, 1, sizeof(buf), fp)) > 0)
        send_all(sfd, buf, (int)n);

    fclose(fp); close(sfd);
    printf("上传完成，大小 %lld 字节\n", fsize);
    return 0;
}
```

```c
// upload_server.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int recvn(int fd, void *buf, int len);   // 见作业七，确保收满 len 字节

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in addr;
    addr.sin_family = AF_INET; addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    int cfd = accept(lfd, NULL, NULL);

    // 1) 读文件名
    char name[64];
    if (recvn(cfd, name, 64) != 64) return 1;

    // 2) 读文件大小（大端 → 主机序）
    uint64_t be; recvn(cfd, &be, 8);
    long long fsize = (long long)be64toh(be);

    // 3) 分批接收并写盘
    FILE *fp = fopen(name, "wb");
    char buf[8192]; long long remaining = fsize;
    while (remaining > 0) {
        int chunk = remaining > (long long)sizeof(buf) ? sizeof(buf) : (int)remaining;
        if (recvn(cfd, buf, chunk) != chunk) break;   // 未收满则出错
        fwrite(buf, 1, chunk, fp);
        remaining -= chunk;
    }
    fclose(fp); close(cfd); close(lfd);
    printf("已保存文件 %s，大小 %lld 字节\n", name, fsize);
    return 0;
}
```

> 说明：`htobe64`/`be64toh` 是 glibc 提供的大小端转换（`<endian.h>`）；若平台没有，可手写移位转换。文件名字段若担心 64 字节不够，可改成"4 字节长度 + 名字"的可变长度协议。

**第 3 题：大文件一次性 read+send 的问题与改进**

问题：
1. **内存占用**：一次性 `read` 需要分配与文件等大的缓冲区（1MB 尚可，GB 级直接撑爆内存）；
2. **send 不保证发完**：一次 `send` 可能只发一部分，必须循环（`send_all`）；
3. **阻塞卡顿**：大块同步 I/O 会让进程/线程长期阻塞，拖垮并发；
4. **数据冗余拷贝**：数据在"用户缓冲 ↔ 内核缓冲"之间来回拷贝，浪费 CPU。

改进方法：
1. **分块流式传输**：固定大小缓冲（如 64KB）循环 `read→send`，不一次性装载整文件；
2. **非阻塞/多路复用**：配合 select/epoll，避免阻塞拖死其他连接；
3. **Linux 零拷贝 `sendfile()`**：内核直接把文件内容发给 socket，**不经过用户态**，减少一次拷贝、显著提速；
4. 也可用 `mmap` 映射文件 + 异步 I/O，或 io_uring 进一步提升。

### 通俗解释
超时踢出就像"**群里潜水太久被管理员请出群**"——服务器每 5 秒扫一眼谁的"最后发言时间"超过 30 秒，就把谁移出并广播。文件上传的"自定义协议"像"**寄快递先填面单**"：先告诉对方"包裹名叫 upload.txt、重量 1MB"，再分批把内容搬过去；接收方按面单对账，保证名字、大小、内容完全一致。大文件不能一口吞，是因为"一次 read 就是把整头牛扛起来"——会扛不动（内存）、还会手滑（send 没发完），所以要"切成小块分批搬"。

---

# 作业七（文件下载 + recvn + UDP 双向）

### 原题
> 1. 实现文件下载功能：服务器端提供两个文件（如 A.txt 和 B.txt），客户端输入文件名请求下载，服务器将对应文件内容发送给客户端，客户端保存到本地。（要求：文件名由客户端发送，服务器根据文件名打开并返回内容。）
> 2. TCP 是流式协议，存在粘包问题。请实现一个函数 int recvn(int sockfd, void *buf, int length)，该函数能确保从 socket 中读取完整的 length 字节数据（即使 recv 返回不足 length 也继续读取），不可使用 MSG_WAITALL 标志。
> 3. 基于 UDP 编写一对程序（客户端和服务端），使用 select 或 epoll 实现双向通信：客户端和服务端均可从标准输入读取并发送消息给对方，同时能接收并打印对方的消息。（提示：UDP 不需要 listen/accept，使用 sendto/recvfrom；服务端需要 bind，客户端可选 bind。）

### 解答

**第 1 题：文件下载**

复用上面的自定义协议：客户端先发"固定 64 字节文件名"，服务器 `recvn` 读名字 → `fopen` 打开 → 先发 8 字节大小 → 再分批发内容。

```c
// download_server.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int recvn(int fd, void *buf, int len);
int send_all(int fd, const void *buf, int len);

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in addr;
    addr.sin_family = AF_INET; addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    while (1) {
        int cfd = accept(lfd, NULL, NULL);
        char name[64];
        if (recvn(cfd, name, 64) != 64) { close(cfd); continue; }

        FILE *fp = fopen(name, "rb");       // 根据文件名打开
        if (!fp) {
            uint64_t zero = 0;
            send_all(cfd, &zero, 8);        // 通知客户端“文件不存在”
            close(cfd); continue;
        }
        fseek(fp, 0, SEEK_END); long long fsize = ftell(fp); fseek(fp, 0, SEEK_SET);
        uint64_t be = htobe64((uint64_t)fsize);
        send_all(cfd, &be, 8);              // 先发大小

        char buf[8192]; size_t n;
        while ((n = fread(buf, 1, sizeof(buf), fp)) > 0)
            send_all(cfd, buf, (int)n);
        fclose(fp); close(cfd);
    }
    return 0;
}
```

```c
// download_client.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdint.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int recvn(int fd, void *buf, int len);

int main() {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in srv;
    srv.sin_family = AF_INET; srv.sin_port = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);
    connect(sfd, (struct sockaddr *)&srv, sizeof(srv));

    char name[64] = {0};
    printf("请输入要下载的文件名: ");
    fgets(name, sizeof(name), stdin);
    name[strcspn(name, "\n")] = 0;          // 去掉换行
    send_all(sfd, name, 64);                // 发送文件名

    uint64_t be; recvn(sfd, &be, 8);
    long long fsize = (long long)be64toh(be);
    if (fsize == 0) { printf("文件不存在\n"); close(sfd); return 1; }

    FILE *fp = fopen(name, "wb");           // 保存到本地
    char buf[8192]; long long remaining = fsize;
    while (remaining > 0) {
        int chunk = remaining > (long long)sizeof(buf) ? sizeof(buf) : (int)remaining;
        if (recvn(sfd, buf, chunk) != chunk) break;
        fwrite(buf, 1, chunk, fp);
        remaining -= chunk;
    }
    fclose(fp); close(sfd);
    printf("下载完成: %s (%lld 字节)\n", name, fsize);
    return 0;
}
```

（其中 `recvn` 就是下面第 2 题的实现，`send_all` 见作业六。）

**第 2 题：recvn（可靠收满 length 字节）**

```c
#include <sys/socket.h>
#include <errno.h>

/*
 * 从 sockfd 可靠地接收 length 字节。
 * 返回实际收到的字节数：
 *   == length 表示完整收到；
 *   <  length 表示对端提前关闭（可能为 0）；
 *   -1       表示出错。
 */
int recvn(int sockfd, void *buf, int length) {
    char *p = (char *)buf;
    int received = 0;
    while (received < length) {
        int n = recv(sockfd, p + received, length - received, 0);
        if (n < 0) {
            if (errno == EINTR) continue;   // 被信号中断，重试
            return -1;                      // 真正的错误
        }
        if (n == 0) break;                  // 对端关闭，收不满 length
        received += n;
    }
    return received;
}
```

要点：`recv` 一次返回的字节数**可能远小于 length**（TCP 是字节流、无报文边界），所以要用 `received` 累计、偏移 `p + received`，循环到收满为止。这也正是**解决"粘包/拆包"的基础**：配合"先固定长度头、再按头中长度收 body"的自定义协议，就能精确切分每一条消息。

**第 3 题：UDP 双向通信（select + 标准输入）**

服务端（需 `bind`，用 `recvfrom/sendto`）：

```c
// udp_chat_server.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

int main() {
    int sfd = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in srv;
    srv.sin_family = AF_INET;
    srv.sin_port   = htons(8888);
    srv.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(sfd, (struct sockaddr *)&srv, sizeof(srv));

    struct sockaddr_in peer; socklen_t plen = sizeof(peer);
    int has_peer = 0;

    fd_set rset; char buf[1024];
    while (1) {
        FD_ZERO(&rset);
        FD_SET(0, &rset);       // 标准输入
        FD_SET(sfd, &rset);     // UDP socket
        int maxfd = sfd > 0 ? sfd : 0;
        select(maxfd + 1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(0, &rset)) {                       // 本地输入 → 发给对端
            if (fgets(buf, sizeof(buf), stdin) == NULL) break;
            if (has_peer)
                sendto(sfd, buf, strlen(buf), 0, (struct sockaddr *)&peer, plen);
        }
        if (FD_ISSET(sfd, &rset)) {                     // 收到对端消息
            int n = recvfrom(sfd, buf, sizeof(buf), 0,
                             (struct sockaddr *)&peer, &plen);
            has_peer = 1;                               // 记住对端地址，便于回复
            fwrite(buf, 1, n, stdout);
        }
    }
    close(sfd);
    return 0;
}
```

客户端（`connect` 可选，这里用 `sendto` 指定服务器地址，也可直接 `connect` 后 `send/recv`）：

```c
// udp_chat_client.c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

int main() {
    int sfd = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in srv;
    srv.sin_family = AF_INET;
    srv.sin_port   = htons(8888);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);

    fd_set rset; char buf[1024];
    while (1) {
        FD_ZERO(&rset);
        FD_SET(0, &rset);
        FD_SET(sfd, &rset);
        int maxfd = sfd > 0 ? sfd : 0;
        select(maxfd + 1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(0, &rset)) {
            if (fgets(buf, sizeof(buf), stdin) == NULL) break;
            sendto(sfd, buf, strlen(buf), 0, (struct sockaddr *)&srv, sizeof(srv));
        }
        if (FD_ISSET(sfd, &rset)) {
            int n = recvfrom(sfd, buf, sizeof(buf), 0, NULL, NULL);
            fwrite(buf, 1, n, stdout);
        }
    }
    close(sfd);
    return 0;
}
```

### 通俗解释
`recvn` 像是"**听写员必须凑齐 length 个字才交卷**"——recv 一次可能只听到几个字，所以要持续听（累计）直到凑够。UDP 双向聊天像"**两个人互寄明信片**"：不事先"接通电话"（无需 listen/accept），各自只留一个收件地址（bind），写了就 `sendto` 寄出、到了就 `recvfrom` 收下；服务端要提前"贴门牌号"（bind），客户端发第一张明信片时服务端顺便记住了它的地址（peer），之后就能回寄。

---

# 作业八（student 表 SQL）

### 原题
> 已有表 student (id int, name varchar(20), chinese float, english float, math float)
> 1. 添加两列：birthday（日期类型）和 native_place（字符串类型）。
> 2. 将 chinese 列的数据类型修改为 INT。
> 3. 插入至少 10 条学生记录（包括姓名、各科成绩、生日、籍贯，自拟）。
> 4. 查询所有姓"张"的学生信息。
> 5. 查询各科均不及格（<60）的学生信息。
> 6. 查询有任意一科不及格的学生姓名。
> 7. 查询至少有两科成绩在 90 分及以上的学生姓名。
> 8. 查询没有一科挂科（即所有科目 >=60）的学生姓名。

### 解答

**第 1 题：添加两列**

```sql
ALTER TABLE student
    ADD COLUMN birthday     DATE,
    ADD COLUMN native_place VARCHAR(50);
```

**第 2 题：修改列类型**

```sql
ALTER TABLE student
    MODIFY COLUMN chinese INT;
```

**第 3 题：插入 10 条记录**

```sql
INSERT INTO student (name, chinese, english, math, birthday, native_place) VALUES
('张伟', 85, 78, 92, '2002-03-12', '北京'),
('张三', 58, 45, 40, '2001-05-20', '上海'),
('李四', 95, 92, 96, '2000-11-02', '广州'),
('王五', 42, 70, 88, '2002-01-08', '深圳'),
('赵六', 88, 59, 76, '2001-09-15', '成都'),
('张敏', 99, 91, 88, '2000-07-23', '武汉'),
('刘强', 61, 62, 63, '2002-12-01', '西安'),
('陈静', 55, 88, 45, '2001-04-19', '杭州'),
('张涛', 90, 95, 92, '1999-08-30', '重庆'),
('孙丽', 70, 75, 80, '2002-06-11', '南京');
```

**第 4 题：查询姓"张"**

```sql
SELECT * FROM student WHERE name LIKE '张%';
```

**第 5 题：各科均不及格（<60）**

```sql
SELECT * FROM student
WHERE chinese < 60 AND english < 60 AND math < 60;
```

（示例数据中 `张三` 三科 58/45/40 命中。）

**第 6 题：有任意一科不及格**

```sql
SELECT name FROM student
WHERE chinese < 60 OR english < 60 OR math < 60;
```

**第 7 题：至少两科 >=90**

```sql
-- 写法1：显式枚举（可移植性好）
SELECT name FROM student
WHERE (chinese >= 90 AND english >= 90)
   OR (chinese >= 90 AND math    >= 90)
   OR (english >= 90 AND math    >= 90);

-- 写法2：MySQL 布尔相加技巧（true=1，false=0）
SELECT name FROM student
WHERE (chinese >= 90) + (english >= 90) + (math >= 90) >= 2;
```

**第 8 题：没有一科挂科（全部 >=60）**

```sql
SELECT name FROM student
WHERE chinese >= 60 AND english >= 60 AND math >= 60;
```

### 通俗解释
`ALTER TABLE ... ADD/MODIFY` 是"**在表格上增删改列**"（加两列、改某一列的类型）；`LIKE '张%'` 是"名字以'张'开头"（`%` 匹配任意多个字）；第 5、6 题就是区分"**并且（AND，全部满足）**"与"**或者（OR，任一满足）**"——前者要求三科全挂，后者挂一科就算。第 7 题"至少两科≥90"等价于"从三科里任选两科 >=90 的组合都成立"。注意：如果成绩列允许 NULL，比较结果会是"未知"，需先用 `IS NOT NULL` 过滤或保证非空。

---

# 作业九（导师/学生表 SQL）

### 原题
> 导师表 t_teacher (tid int primary key, name varchar(20), title varchar(20), research_area varchar(50))
> 学生表 t_student (sid int primary key, name varchar(20), gender enum('male','female'), tutor_id int, enroll_date date)
> 每个学生只有一个导师（外键 tutor_id 引用 t_teacher.tid）
> 1. 设计并创建上述两个表，并插入适量的数据（至少 5 个导师，每个导师带 2~3 个学生）。
> 2. 查询每个导师所带研究生的姓名（要求用 GROUP_CONCAT 或子查询）。
> 3. 查询特定导师（如"张教授"）所带研究生的姓名。
> 4. 统计每个导师所带研究生的数量。
> 5. 统计每个导师所带的男研究生数量。
> 6. 找出哪个研究方向的导师人数最多（即最多导师从事的方向）。
> 7. 统计不同职称（title）的导师人数。
> 8. 查询每个导师所带学生的平均入学年份，并按平均年份降序排列。

### 解答

**第 1 题：建表 + 插入数据**

```sql
CREATE TABLE t_teacher (
    tid           INT PRIMARY KEY,
    name          VARCHAR(20),
    title         VARCHAR(20),
    research_area VARCHAR(50)
);

CREATE TABLE t_student (
    sid        INT PRIMARY KEY,
    name       VARCHAR(20),
    gender     ENUM('male','female'),
    tutor_id   INT,
    enroll_date DATE,
    FOREIGN KEY (tutor_id) REFERENCES t_teacher(tid)
);

-- 5 个导师
INSERT INTO t_teacher (tid, name, title, research_area) VALUES
(1, '张教授', '教授',   '人工智能'),
(2, '李教授', '教授',   '计算机视觉'),
(3, '王副教授','副教授','人工智能'),
(4, '赵讲师', '讲师',   '数据库'),
(5, '刘教授', '教授',   '计算机网络');

-- 13 个学生（每位导师 2~3 人）
INSERT INTO t_student (sid, name, gender, tutor_id, enroll_date) VALUES
(101, '张三',  'male',   1, '2021-09-01'),
(102, '张敏',  'female', 1, '2022-09-01'),
(103, '李明',  'male',   1, '2023-09-01'),
(104, '王芳',  'female', 2, '2021-09-01'),
(105, '李强',  'male',   2, '2022-09-01'),
(106, '赵丽',  'female', 3, '2021-09-01'),
(107, '刘洋',  'male',   3, '2023-09-01'),
(108, '孙悦',  'female', 3, '2023-09-01'),
(109, '周杰',  'male',   4, '2022-09-01'),
(110, '吴倩',  'female', 4, '2023-09-01'),
(111, '郑爽',  'female', 4, '2023-09-01'),
(112, '钱进',  'male',   5, '2021-09-01'),
(113, '冯雪',  'female', 5, '2022-09-01');
```

**第 2 题：每个导师所带学生姓名（GROUP_CONCAT）**

```sql
SELECT t.tid, t.name AS teacher,
       GROUP_CONCAT(s.name ORDER BY s.sid) AS students
FROM t_teacher t
LEFT JOIN t_student s ON t.tid = s.tutor_id
GROUP BY t.tid, t.name;
```

（用 `LEFT JOIN` 可连"没有学生"的导师也显示出来。子查询写法等价：

```sql
SELECT t.name,
       (SELECT GROUP_CONCAT(s.name ORDER BY s.sid)
        FROM t_student s WHERE s.tutor_id = t.tid) AS students
FROM t_teacher t;
```

）

**第 3 题：特定导师（张教授）的学生**

```sql
SELECT s.name
FROM t_student s
JOIN t_teacher t ON s.tutor_id = t.tid
WHERE t.name = '张教授';
```

**第 4 题：每个导师的学生数量**

```sql
SELECT t.name, COUNT(s.sid) AS cnt
FROM t_teacher t
LEFT JOIN t_student s ON t.tid = s.tutor_id
GROUP BY t.tid, t.name;
```

**第 5 题：每个导师的男学生数量**

```sql
SELECT t.name, COUNT(s.sid) AS male_cnt
FROM t_teacher t
LEFT JOIN t_student s
       ON t.tid = s.tutor_id AND s.gender = 'male'
GROUP BY t.tid, t.name;
```

（关键点：把 `gender='male'` 写进 **JOIN 的 ON 条件**而不是 WHERE，否则没有男学生的导师会被过滤掉。）

**第 6 题：导师人数最多的研究方向**

```sql
SELECT research_area, COUNT(*) AS teacher_cnt
FROM t_teacher
GROUP BY research_area
ORDER BY teacher_cnt DESC
LIMIT 1;
```

（示例数据中"人工智能"有张教授、王副教授 2 人，结果就是它。）

**第 7 题：按职称统计导师人数**

```sql
SELECT title, COUNT(*) AS cnt
FROM t_teacher
GROUP BY title;
```

**第 8 题：每个导师所带学生的平均入学年份，降序**

```sql
SELECT t.name,
       AVG(YEAR(s.enroll_date)) AS avg_enroll_year
FROM t_teacher t
LEFT JOIN t_student s ON t.tid = s.tutor_id
GROUP BY t.tid, t.name
ORDER BY avg_enroll_year DESC;
```

### 通俗解释
两张表用 `tutor_id → tid` 的**外键**关联，就像"学生卡上写着导师工号"。`JOIN` 是"把两本花名册按工号对起来"；`GROUP BY t.name` 是按导师分组，`GROUP_CONCAT(s.name)` 把该组所有学生名拼成一列；统计男学生时，把 `gender='male'` 放进 **JOIN 条件**而不是 WHERE，是为了"没有男学生的导师也要显示 0"，而不是被整行滤掉——这是本类题的经典考点。

---

# 作业十（city 表 + 数据库设计）

### 原题
> 1. 城市表统计（参考题目）：
> - 创建城市表 city，包含字段：id, name, country, population, province。
> - 插入至少 15 条数据（覆盖中国、美国、日本等，人口数自拟）。
> - 完成以下查询：
> a) 查询所有城市名及人口。
> b) 查询所有中国的城市信息。
> c) 查询人口数小于 100 万的城市信息（单位自定）。
> d) 查询中国人口超过 500 万的城市。
> e) 查询中国或美国的城市信息。
> f) 查询人口在 100 万~200 万之间的城市。
> g) 查询中国或美国人口大于 500 万的城市。
> h) 查询城市名以"qing"开头的城市。
> i) 统计城市表的总行数。
> j) 统计每个国家的城市个数。
> k) 统计每个国家的城市总人口数。
> l) 统计中国每个省的城市个数及城市名列表（使用 GROUP_CONCAT）。
> m) 统计每个国家的城市个数，只显示超过 5 个城市的国家，并按城市数降序排列，取前三名。
>
> 2. 设计题：某高校有多个学院，每个学院有多名教师，每名教师可指导多名学生，但每个学生只能有一位导师。请设计数据库表（至少包括学院、教师、学生三个表），并写出创建表的 SQL 语句，并插入若干数据，然后查询每个学院教师所带学生总数。

### 解答

**第 1 题：建表 + 数据 + 查询**

> 说明：为配合 h) 的 `'qing%'` 查询，城市名用**拼音/英文**存储（如 Qingdao），国家名用英文（China/USA/Japan）；population 单位为**人**。

```sql
CREATE TABLE city (
    id         INT PRIMARY KEY AUTO_INCREMENT,
    name       VARCHAR(50),
    country    VARCHAR(50),
    population INT,
    province   VARCHAR(50)
);

INSERT INTO city (name, country, population, province) VALUES
('Beijing',    'China', 21540000, 'Beijing'),
('Shanghai',   'China', 24870000, 'Shanghai'),
('Guangzhou',  'China', 18680000, 'Guangdong'),
('Shenzhen',   'China', 17560000, 'Guangdong'),
('Chengdu',    'China', 20940000, 'Sichuan'),
('Chongqing',  'China', 32050000, 'Chongqing'),
('Xian',       'China', 12950000, 'Shaanxi'),
('Wuhan',      'China', 12330000, 'Hubei'),
('Nanjing',    'China',  9310000, 'Jiangsu'),
('Hangzhou',   'China', 11930000, 'Zhejiang'),
('Suzhou',     'China', 12750000, 'Jiangsu'),
('Qingdao',    'China', 10070000, 'Shandong'),
('Yichang',    'China',  1500000, 'Hubei'),
('New York',   'USA',    8419000, 'New York'),
('Los Angeles','USA',    3971000, 'California'),
('Chicago',    'USA',    2746000, 'Illinois'),
('Houston',    'USA',    2305000, 'Texas'),
('San Francisco','USA',   873000, 'California'),
('Seattle',    'USA',     737000, 'Washington'),
('San Jose',   'USA',    1013000, 'California'),
('Tokyo',      'Japan', 13960000, 'Tokyo'),
('Osaka',      'Japan',  2690000, 'Osaka'),
('Nagoya',     'Japan',  2320000, 'Aichi');
```

（共 23 条 ≥15，覆盖中/美/日。各查询如下：）

```sql
-- a) 所有城市名及人口
SELECT name, population FROM city;

-- b) 所有中国的城市信息
SELECT * FROM city WHERE country = 'China';

-- c) 人口小于 100 万（< 1000000）
SELECT * FROM city WHERE population < 1000000;

-- d) 中国人口超过 500 万
SELECT * FROM city WHERE country = 'China' AND population > 5000000;

-- e) 中国或美国
SELECT * FROM city WHERE country IN ('China', 'USA');

-- f) 人口在 100 万~200 万之间（含边界）
SELECT * FROM city WHERE population BETWEEN 1000000 AND 2000000;

-- g) 中国或美国人口大于 500 万
SELECT * FROM city WHERE country IN ('China','USA') AND population > 5000000;

-- h) 城市名以 qing 开头
SELECT * FROM city WHERE name LIKE 'qing%';       -- 命中 Qingdao

-- i) 总行数
SELECT COUNT(*) AS total_cities FROM city;

-- j) 每个国家的城市个数
SELECT country, COUNT(*) AS city_count FROM city GROUP BY country;

-- k) 每个国家的城市总人口
SELECT country, SUM(population) AS total_pop FROM city GROUP BY country;

-- l) 中国每个省的城市个数及城市名列表
SELECT province,
       COUNT(*)                AS city_count,
       GROUP_CONCAT(name SEPARATOR '、') AS city_list
FROM city
WHERE country = 'China'
GROUP BY province;

-- m) 城市数 > 5 的国家，按城市数降序，取前三
SELECT country, COUNT(*) AS city_count
FROM city
GROUP BY country
HAVING city_count > 5
ORDER BY city_count DESC
LIMIT 3;
```

**第 2 题：学院/教师/学生 数据库设计**

关系建模：**学院 1—N 教师**（一个学院多名教师）、**教师 1—N 学生**（一个导师带多名学生）。三张表：

```sql
CREATE TABLE college (
    college_id INT PRIMARY KEY,
    name       VARCHAR(50) NOT NULL
);

CREATE TABLE teacher (
    teacher_id INT PRIMARY KEY,
    name       VARCHAR(20) NOT NULL,
    college_id INT,
    FOREIGN KEY (college_id) REFERENCES college(college_id)
);

CREATE TABLE student (
    student_id INT PRIMARY KEY,
    name       VARCHAR(20) NOT NULL,
    teacher_id INT,
    FOREIGN KEY (teacher_id) REFERENCES teacher(teacher_id)
);

INSERT INTO college (college_id, name) VALUES
(1, '计算机学院'), (2, '电子信息学院'), (3, '机械学院');

INSERT INTO teacher (teacher_id, name, college_id) VALUES
(11, '张老师', 1), (12, '李老师', 1), (13, '王老师', 2), (14, '赵老师', 3);

INSERT INTO student (student_id, name, teacher_id) VALUES
(101, '张三', 11), (102, '李四', 11), (103, '王五', 12),
(104, '赵六', 13), (105, '孙七', 13), (106, '周八', 13),
(107, '吴九', 14);
```

查询"每个学院教师所带学生总数"——把三张表链起来，按学院分组统计：

```sql
SELECT c.name AS college_name,
       COUNT(s.student_id) AS total_students
FROM college c
LEFT JOIN teacher t ON c.college_id = t.college_id
LEFT JOIN student s ON t.teacher_id = s.teacher_id
GROUP BY c.college_id, c.name;
```

> 用 **LEFT JOIN** 而不是 INNER JOIN：这样"没有教师"或"教师没有学生"的学院也能显示出来（数量为 0）。若要"学院内每位老师的学生数"再套一层，可按 `c.name, t.name` 分组。实际生产中还常用 `ON DELETE` 约束（如教师离职时学生如何处理）来决定删除策略。

### 通俗解释
city 那组查询是"**给城市花名册做各种筛选和统计**"：`WHERE` 是横向筛行，`GROUP BY country` 是按国家"归堆"，`COUNT/SUM` 是数堆里的数量/求和，`GROUP_CONCAT(name)` 是把堆里的名字串成一串，`HAVING` 是"分组之后再筛组"（比如只留城市数>5 的国家），`LIMIT 3` 是"只取前三条"。设计题里"学院→教师→学生"是典型的**一对多链式关系**：学生表存导师工号（外键）、教师表存学院号（外键），查询时用两次 JOIN 把三层串起来再分组统计，这就是数据库设计的"外键 + 规范化"核心思想。

---

## 总结

这 10 道题覆盖两大主线：
- **网络编程**：HTTP 报文/抓包 → OSI/TCP-IP → 握手挥手 → socket 编程 → select/epoll → 超时踢人 → 文件上传/下载 → `recvn` 可靠接收 → UDP 双向通信，构成了从"理论"到"能跑起来的完整服务"的闭环。
- **SQL**：建表/改表 → 增删改查 → 模糊查询 → 分组聚合 → `GROUP_CONCAT` → 多表 JOIN → 外键关联统计，覆盖了面试/作业最常见的 SQL 考点。
