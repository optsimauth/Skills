---
name: layered-function-decomposition
description: 强制将代码组织为清晰的"编排层"（函数体只调用其他函数）与"实现层"（单一、简单到可一眼看懂的具体逻辑），禁止两者混在同一个函数里。基于软件工程中已验证的 SLAP（单一抽象层次原则）/ Composed Method（组合方法模式）/ Step-down Rule（递降规则）。当用户要求编写、生成、重构任何函数、模块、脚本、业务逻辑代码时都应主动使用此规范 —— 尤其是涉及多步骤流程、数据处理管道、条件判断链、包含计算+校验+副作用混合逻辑的场景。也适用于代码审查（review）时判断某个函数是否需要拆分。即使用户没有明确提到"分层"或"可读性"，只要是"写代码/写函数/实现某功能"的请求，都应默认遵循此规范。
---

# 分层函数分解规范（Layered Function Decomposition）

## 为什么这样做

人在读代码时，真正消耗认知资源的不是"代码总量"，而是**在同一段代码里被迫在不同抽象层次之间来回切换**——一会儿在想"这一步的业务目的是什么"，一会儿又要盯着某个具体的字符串拼接或数值换算。这种切换会打断"模式识别"（pattern recognition），迫使读者用逐行推理去代替一眼扫过就能理解的直觉判断。

解决方法不是新发明，而是软件工程里三个互相印证的经典原则：

- **SLAP（Single Level of Abstraction Principle）**：同一个函数内的所有代码必须处于同一抽象层次。
- **Composed Method（Kent Beck）**：公共方法应该读起来像一份"步骤大纲"，具体怎么做交给私有方法。
- **Step-down Rule（Robert C. Martin, *Clean Code*）**：代码应该像自上而下的叙事，调用者永远在被调用者上方，往下读一层，抽象层次刚好降一级。

本 skill 把这三者转化为写代码时可以直接执行的强制规则。

---

## 核心规则：函数只能是两种类型之一

写任何函数之前，先决定它属于哪一类，不允许中间态。

### 类型 A：编排函数（Orchestrator）

- 函数体**只包含对其他函数的调用**（可以有简单的顺序、条件分支、循环，用来决定调用哪个/是否调用），但不包含具体的计算、字符串/数据结构操作、业务判断细节。
- 读这个函数就像读一份目录 / 伪代码，不需要理解任何实现细节就能明白"这一步在做什么、下一步是什么"。
- 函数名必须是**动词短语**，描述"做什么"（What），而不是"怎么做"（How）。

### 类型 B：叶子函数（Leaf）

- 函数体是**不可再分的单一语义单元**：一次计算、一次校验、一次数据转换、一次副作用（比如一次网络调用、一次写文件）。
- 简单到读者不需要下钻就能"模式识别"出它在做什么——如果还需要在心里默念好几步才能懂，说明它还不是叶子，需要继续拆。
- 叶子函数内部允许有正常的语言特性（循环、条件、局部变量），只要它们服务于**同一个语义目的**。

**判断标准（写完一个函数后自问）**：
> 这个函数里，有没有哪一行和其他行"感觉不是一回事"？如果有，说明混入了不同抽象层次，需要把那部分抽出去成为子函数。

---

## 强制规则

1. **禁止混合层次**：一个函数体内不能既有"调用子函数"又有"具体实现代码"（除了极简单的胶水代码，见下方"允许的例外"）。
2. **禁止编排层做实际工作**：编排函数里不写 if 判断的具体业务条件表达式（比如 `if user.age >= 18 and user.status == 'active'`），把这种判断本身也封装成一个语义清晰的函数（比如 `isEligibleUser(user)`）。
3. **调用顺序即阅读顺序（Step-down Rule）**：一个函数定义完之后，紧跟着定义它调用的子函数，让代码文件本身可以从上到下读成一篇叙事，不需要来回跳转查找。
4. **递归应用**：子函数如果还混合了抽象层次，继续拆，直到每一片叶子都满足"一眼识别"标准。
5. **命名必须是语义化的动词短语**，让调用链本身就是文档。反例：`process()`、`handle()`、`doStuff()`、`helper()`；正例：`validateOrderPayment()`、`calculateShippingFee()`、`extractUserEmailFromHeader()`。

---

## 拆分终止条件（避免过度拆分 / 避免拆不够）

拆分不是拆得越细越好，过度拆分会导致"跳转疲劳"——读者要在几十个单行函数之间反复跳转，反而比读一个稍长但连贯的函数更累。用以下量化标准判断"够了"：

| 信号 | 处理方式 |
|---|---|
| 函数体超过约 15-20 行**且**明显能看出几个不同目的的代码块 | 继续拆 |
| 函数内出现嵌套超过 2 层的 if/for | 把内层逻辑提取成命名函数 |
| 函数名之后需要加注释才能说明它在干什么 | 说明命名/拆分粒度不对，重新拆或重新命名 |
| 一个函数只有 1-3 行，且只是简单转发调用、没有独立语义价值 | 停止拆分，考虑内联合并回调用者 |
| 拆出来的子函数只会被调用一次，且逻辑极简单（比如一行数学表达式且含义自解释） | 可以不拆，除非该行明显是不同抽象层次 |

**经验法则**：叶子函数理想长度在 3～15 行之间；编排函数理想长度等于它调用的子函数数量（每行一个调用，或每个分支一个调用），一般不超过 10～15 行。

---

## 编排层允许的内容 vs 不允许的内容

**允许（属于"编排"，不算"实现"）**：
- 简单的顺序调用
- 简单的 if/else 来决定调用 A 还是 B（但判断条件本身应该是一个语义函数返回的布尔值，而不是原始表达式）
- 简单的 for/while 循环来遍历调用某个函数
- 组装/传递参数给子函数（不做计算，只做数据搬运）
- 捕获异常后决定调用哪个恢复函数（异常处理的具体逻辑本身还是要下沉）

**不允许（属于"实现"，必须下沉到叶子函数）**：
- 任何数学运算、字符串处理、数据结构变换
- 具体的业务规则表达式（多个条件用 and/or 拼接的判断）
- 直接操作数据库/文件/网络的具体调用细节（应该封装成语义函数，比如 `saveOrderToDatabase(order)` 而不是裸的 SQL/ORM 调用散落在编排函数里）
- 循环体内如果做的不只是"调用同一个函数"，而是包含具体逻辑，要把循环体也提取成函数

**允许的例外（不必强行封装成函数）**：
- 纯粹的数据搬运/重命名（比如把子函数返回值赋给一个更易读的变量名）
- 单行日志打印
- 极简单、只用一次、含义完全自解释的表达式（比如 `const total = items.length`）

拆分的目的是可读性，不是"函数数量最大化"，遇到这些例外不必较真。

---

## 完整示例

### ❌ 拆分前（一个函数里混了多个抽象层次）

```python
def process_order(order, user):
    # 校验逻辑
    if order.total <= 0 or not user.is_active or user.credit_score < 600:
        raise ValueError("invalid order")

    # 计算折扣
    discount = 0
    if user.membership == "gold":
        discount = order.total * 0.15
    elif user.membership == "silver":
        discount = order.total * 0.08

    final_price = order.total - discount

    # 调用支付网关
    payload = {"amount": final_price, "currency": "USD", "user_id": user.id}
    response = requests.post("https://payment.api/charge", json=payload)
    if response.status_code != 200:
        raise RuntimeError("payment failed")

    # 更新库存
    for item in order.items:
        db.execute("UPDATE inventory SET stock = stock - ? WHERE sku = ?",
                   (item.qty, item.sku))

    # 发通知
    send_email(user.email, f"Order confirmed, charged {final_price}")
```

这个函数把"校验规则的具体条件"、"折扣的具体计算"、"HTTP 请求细节"、"SQL 细节"全都摊在同一层，读者必须逐行推理才能拼出"这个函数到底在做什么"。

### ✅ 拆分后（编排层 + 叶子层）

```python
def process_order(order, user):
    validate_order_is_eligible(order, user)
    final_price = calculate_final_price(order, user)
    charge_user_for_order(user, final_price)
    deduct_inventory_for_order(order)
    notify_user_order_confirmed(user, final_price)


def validate_order_is_eligible(order, user):
    if order.total <= 0 or not user.is_active or user.credit_score < 600:
        raise ValueError("invalid order")


def calculate_final_price(order, user):
    discount = calculate_membership_discount(order.total, user.membership)
    return order.total - discount


def calculate_membership_discount(total, membership):
    rates = {"gold": 0.15, "silver": 0.08}
    return total * rates.get(membership, 0)


def charge_user_for_order(user, amount):
    response = call_payment_gateway(user.id, amount)
    if response.status_code != 200:
        raise RuntimeError("payment failed")


def call_payment_gateway(user_id, amount):
    payload = {"amount": amount, "currency": "USD", "user_id": user_id}
    return requests.post("https://payment.api/charge", json=payload)


def deduct_inventory_for_order(order):
    for item in order.items:
        deduct_inventory_for_item(item)


def deduct_inventory_for_item(item):
    db.execute("UPDATE inventory SET stock = stock - ? WHERE sku = ?",
               (item.qty, item.sku))


def notify_user_order_confirmed(user, final_price):
    send_email(user.email, f"Order confirmed, charged {final_price}")
```

现在 `process_order` 本身就是一份可以直接讲给非技术人员听的流程说明：校验 → 算价 → 收款 → 扣库存 → 通知。想核实"折扣算得对不对"，直接跳进 `calculate_membership_discount`，不用扫全文。

---

## 写代码 / 审查代码时的自查清单

生成或修改代码后，逐条检查：

- [ ] 每个函数只属于"编排"或"叶子"其中一种，没有混合
- [ ] 编排函数里没有具体的数学/字符串/数据结构操作
- [ ] 编排函数里没有拼接多个条件的裸判断表达式（已封装成语义函数）
- [ ] 每个函数名是动词短语，看名字就知道做什么，不需要看实现
- [ ] 一个函数定义之后紧跟着它调用的子函数定义（Step-down Rule）
- [ ] 没有为了拆而拆——那些只调用一次、极简单、自解释的表达式没有被强行封装
- [ ] 叶子函数长度大致在 3～15 行，编排函数长度大致等于它的调用数量
- [ ] 如果把这个模块的所有函数名和调用关系列出来（不看函数体），一个不熟悉代码的人也能大致理解整个流程

如果某一条不满足，回去继续拆分或合并，直到满足为止。
