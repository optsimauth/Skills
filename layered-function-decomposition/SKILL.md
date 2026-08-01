---
name: layered-function-decomposition
description: 强制将代码组织为编排层（只调用函数）与叶子层（单一语义、简单到一眼看懂），禁止两者混在同一函数。基于 SLAP/Composed Method/Step-down Rule。用户要求编写、生成、重构任何函数/模块/脚本/业务逻辑代码时必须使用，无需用户明确提及"分层"。
---

# 分层函数分解规范（强制）

## 铁律

写任何函数前，必须先判定其类型，禁止中间态：

1. **编排函数**：函数体只能是"调用其他函数 + 简单顺序/条件分支/循环（用于决定调用谁）"。禁止出现任何数学运算、字符串/数据结构处理、裸业务条件表达式（如 `a and b or c`）、直接的 DB/HTTP/文件操作。分支判断必须封装为返回布尔值的语义函数（如 `isEligibleUser(user)`），不允许原始表达式出现在编排函数里。
2. **叶子函数**：只能承担单一语义单元（一次计算/一次校验/一次转换/一次副作用）。长度必须控制在 3～15 行。若需要在心里默念多步才能理解，必须继续拆分。

## 强制规则（逐条不可违反）

- 一个函数体内**禁止**同时出现"调用子函数"与"具体实现代码"。
- 编排函数长度 ≤ 调用数量，一般不超过 10～15 行。
- 函数名必须是动词短语，且必须自解释，禁止 `process/handle/doStuff/helper` 等模糊命名。
- 子函数定义必须紧跟在其调用者之后（Step-down Rule），文件必须能从上到下读成一篇叙事。
- 递归拆分直至每个叶子满足"一眼识别"标准，无一例外。
- 若函数体超过 15~20 行且包含多个不同目的的代码块 → 必须拆分，不允许找理由跳过。
- 嵌套 if/for 超过 2 层 → 必须把内层提取为命名函数。
- 函数名之后仍需注释才能说明其作用 → 命名或拆分不合格，必须重做。

## 唯一允许不拆的例外（仅限以下情形）

- 纯数据搬运/变量重命名
- 单行日志打印
- 只用一次、完全自解释的单行表达式（如 `total = len(items)`）

除以上三种情形外，一律拆分，不接受任何"为了简洁"的例外理由。

## 示例（仅作格式参照，不代表可降低标准）

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
```

## 自查清单（每次生成代码后必须逐条核对，任何一项不满足禁止交付）

- [ ] 每个函数只属于编排或叶子二者之一，无混合
- [ ] 编排函数内无具体计算/字符串/数据结构操作
- [ ] 编排函数内无裸业务条件表达式
- [ ] 所有函数名为自解释动词短语
- [ ] 调用者定义在被调用者之前，顺序即阅读顺序
- [ ] 叶子函数长度 3～15 行，编排函数长度 ≈ 调用数量
- [ ] 仅在三种允许例外下才存在未拆分的代码
