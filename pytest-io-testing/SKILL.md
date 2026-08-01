---
name: pytest-io-testing
description: 用输入输出对为AI生成的Python代码写pytest测试，让不看实现的人靠跑测试判断代码对错。用户要求写代码、写测试、验证代码、提到pytest/单元测试时必须使用，无需用户明确提及"skill"。每写完/改完一个源文件都必须主动配套测试。
---

# Pytest 输入输出测试（强制）

## 铁律

- 每个 `src/xxx.py` 必须对应且仅对应一个 `tests/test_xxx.py`，禁止把测试和源码写在同一文件。
- 用户只看 `pytest -v` 的 PASSED/FAILED 判断对错，测试必须让用户不看实现也能定位错误。
- case 的取舍**必须**由被测函数实现里是否存在对应分支决定，禁止套用"正常/边界/异常"模板凑数；实现里没有的分支不允许写 case。

## 强制格式（不允许偏离）

```python
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))
from xxx import 被测函数

cases = [
    (输入1, 输入2, 期望输出, "说明"),
]

import pytest

@pytest.mark.parametrize("参数名1,参数名2,expected,note", cases)
def test_被测函数(参数名1, 参数名2, expected, note):
    实际 = 被测函数(参数名1, 参数名2)
    assert 实际 == expected, f"[{note}] 输入({参数名1},{参数名2})，期望{expected}，实际{实际}"
```

- `parametrize` 字符串里的参数名必须与函数签名参数名**逐字一致**、数量一致，否则视为不合格。
- 每个 assert 必须携带 `f"[{note}] 输入...期望...实际..."` 格式的失败信息，禁止裸 assert。
- **一个文件内所有函数的所有 cases 列表必须集中放在文件最顶部**（import 之后、第一个 `def test_...` 之前），禁止分散定义、禁止与测试函数交替书写、禁止在测试函数内部临时定义 case。
- **每条 case 的 `note` 字段必须写明该 case 在验证被测函数的哪个具体分支/规则/场景**（如"gold会员8折""total为0应抛异常"），禁止使用"正常情况""测试1""case1"等无信息量占位说明。
- 涉及状态变化/行为（锁定、计数器等）的测试，函数名必须用中文描述期望行为（如 `test_连续失败5次后应锁定`），禁止用 parametrize 数值硬套。

## 完成后强制动作（缺一不可）

1. 必须实际执行 `pytest -v`，禁止只口头声称"写好了"。
2. 结果必须原样展示给用户。
3. 出现 FAILED，必须先判定是测试数据错误还是代码逻辑错误，明确给出结论并定位到具体函数，禁止把报错原文甩给用户了事。
4. 禁止引入额外测试框架/数据库/CI 配置，除非用户明确要求。
