---
name: decompose-and-test
description: 编写高准确率AI生成代码的统一流程——先用分层函数分解（编排层/叶子层拆分，SLAP/Composed Method/Step-down Rule）把逻辑拆成大量小而语义清晰的函数，再为每个叶子函数配套 pytest 输入输出对测试，让不看实现的人也能靠"跑测试"判断代码对错。当用户要求"写代码""实现某功能""重构""帮我写个脚本/模块/工具"，尤其是涉及多步骤业务流程、数据处理管道、条件判断链、计算+校验+副作用混合逻辑时，必须主动触发本skill，即使用户没有提到"分解""测试""pytest""质量"等字眼。只要任务是"产出可运行的Python代码"且逻辑复杂度超过一两行，就应默认走这个流程，而不是分别单独触发 layered-function-decomposition 或 pytest-io-testing 中的一个。
---

# 分解 + 测试：统一代码质量流程

## 这个 skill 解决什么问题

两个已有的 skill 各管一段：

- `layered-function-decomposition` 负责**写的时候**怎么组织代码——拆成编排层（只调用）和叶子层（单一语义、简单到一眼看懂）。
- `pytest-io-testing` 负责**写完之后**怎么验证——用输入输出对做 parametrize 测试，让人不看代码也能靠 PASS/FAIL 判断对错。

单独用其中一个都不够：只拆分不测试，代码好读但对不对靠"看起来像对的"；只测试不拆分，测试再多也测的是一个大黑箱函数，一旦 FAILED 用户根本定位不到是哪一步错了。

**核心洞察**：分层分解产生的"叶子函数"天然就是 pytest-io-testing 想测的最小单元——一次计算、一次校验、一次转换。把两者串起来，就得到"拆得越细 → 每个叶子越容易测 → 出错定位越精确 → 整体正确率越高"的正循环。

## 统一工作流程

对任意一个代码生成/重构任务，按下面 5 步顺序执行，不要跳步：

### 第 1 步：按分层分解规范写代码

严格应用 `layered-function-decomposition` 的规则：
- 先决定顶层是编排函数还是叶子函数
- 编排函数只做调用编排，不写具体计算/字符串处理/裸业务条件表达式
- 递归拆分，直到每个叶子函数满足"3~15行、一眼可识别语义"的标准
- 文件内 Step-down 排列：调用者在上，被调用者紧随其后

产出：`src/xxx.py`，包含 1 个（或少数几个）编排函数 + 若干叶子函数。

### 第 2 步：列出"待测清单"

把第1步产出的所有**叶子函数**列成一张表（函数名、输入参数、预期职责），这张表就是后续测试要覆盖的范围。编排函数一般不需要单独做 parametrize 测试（它没有独立的计算逻辑），但如果编排函数包含"决定调用A还是B"的分支逻辑，也要为分支路径写一个行为型测试用例。

### 第 3 步：为每个叶子函数生成输入输出对

应用 `pytest-io-testing` 的规范：
- 每个叶子函数对应 `cases` 列表里的若干组 `(输入..., 期望输出)`，覆盖正常值、边界值（0/空/极值）、异常值
- 纯计算/转换类叶子函数 → `@pytest.mark.parametrize` 数值型测试
- 涉及状态变化或副作用的叶子函数（比如 `charge_user_for_order`、`deduct_inventory_for_item`）→ 用 mock 隔离外部依赖（HTTP/DB），再对"调用参数是否正确""异常是否正确抛出"做行为型断言，测试函数名用中文描述期望行为

### 第 4 步：为编排函数写集成级测试（可选但推荐）

如果编排函数有实际业务价值判断分支，额外写 1~2 个端到端用例，验证"叶子函数被正确串起来了"，而不仅仅是"每个叶子单独正确"。这类测试可以对内部的叶子函数做 mock/spy，验证调用顺序和参数传递，而不重复断言叶子函数内部的计算逻辑（那已经在第3步测过了）。

### 第 5 步：跑测试并汇报结果

按 `tests/` 目录约定（`tests/test_xxx.py` 对应 `src/xxx.py`）跑：

```bash
pytest -v
```

把结果原样展示给用户：
- 全 PASS → 明确告诉用户"N 个叶子函数、M 条测试全部通过"，并附上拆分出的函数调用关系（一份编排函数目录），让用户不用看实现也能确认逻辑合理
- 有 FAILED → 先自己判断是"测试数据写错"还是"代码逻辑错"，给出结论和定位到具体哪个叶子函数，而不是把报错原文甩给用户

## 完整示例（浓缩版）

以 `layered-function-decomposition` 里的 `process_order` 为例，拆分后产出 7 个函数（1 个编排 + 6 个叶子）。第 3 步针对每个叶子写测试：

```python
# tests/test_order.py
import pytest
from unittest.mock import patch
from order import (
    calculate_membership_discount,
    calculate_final_price,
    validate_order_is_eligible,
)

# 纯计算叶子：parametrize 数值对
discount_cases = [
    (100, "gold", 15.0),
    (100, "silver", 8.0),
    (100, "bronze", 0.0),   # 边界：未知等级
    (0, "gold", 0.0),       # 边界：金额为0
]

@pytest.mark.parametrize("total,membership,expected", discount_cases)
def test_calculate_membership_discount(total, membership, expected):
    actual = calculate_membership_discount(total, membership)
    assert actual == expected, f"输入(total={total}, membership={membership})，期望{expected}，实际{actual}"

# 校验类叶子：行为型测试
def test_订单金额为0时应拒绝():
    with pytest.raises(ValueError):
        validate_order_is_eligible(order=make_order(total=0), user=make_active_user())

def test_信用分不足时应拒绝():
    with pytest.raises(ValueError):
        validate_order_is_eligible(order=make_order(total=100), user=make_user(credit_score=500))
```

编排函数 `process_order` 则单独用 mock 验证"调用链是否完整、顺序是否正确"，不重复算折扣的具体数值。

## 什么时候可以简化流程

- 任务只是"写一个 10 行以内的一次性小脚本，没有分支/校验/副作用" → 分层分解可以跳过（本来就没什么可拆），但仍建议至少给核心函数配 2~3 组输入输出测试
- 用户明确说"先别写测试，我先看逻辑对不对" → 先完成第1~2步展示函数拆分和调用关系，等用户确认后再做第3~5步
- 纯配置/纯数据文件（没有函数逻辑）→ 本 skill 不适用

## 自查清单（完成后过一遍）

- [ ] 编排函数和叶子函数职责清晰分离，没有混层
- [ ] 每个叶子函数都能在待测清单里找到对应的测试用例
- [ ] 测试用例覆盖正常值、边界值、异常值三类
- [ ] 每个 assert / raises 断言都能让用户不看代码就懂"错在哪"
- [ ] 已实际执行 `pytest -v` 并把结果（尤其 FAILED 详情）展示给用户，而不是空口说"写好了"
