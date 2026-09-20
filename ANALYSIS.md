# PDDL 库解析链路分析（domain / problem / plan）

本文基于当前仓库的真实代码与测试（Lark 1.1.9 / `pddl` 0.4.10，Python 3.10+）。所有结论均在本地用脚本实际跑过；无法证实的内容只写为“观察/限制”，不写成缺陷。行号对应当前工作区文件。

- 文法文件：`pddl/parser/grammar.lark`
- 公共入口：`pddl/__init__.py:31`（`parse_domain`）、`pddl/__init__.py:39`（`parse_problem`）、`pddl/__init__.py:47`（`parse_plan`），以及 CLI `pddl/__main__.py:37`
- 三个 parser 类：`pddl/parser/domain.py:555`、`pddl/parser/problem.py:220`、`pddl/parser/plan.py:47`

---

## 1. 总体结构：三条入口共享什么

三条入口共享同一个文法文件和同一套对象模型，但 transformer 并不在入口间共享：

1. **共享文法**：`BaseParser.__init__` 每次实例化都读一次 `GRAMMAR_FILE`（路径常量在 `pddl/parser/__init__.py:17-18`），构造一个 LALR parser，`start` 分别是 `domain`/`problem`/`plan`（`pddl/parser/base.py:32-42`）。解析与规约同时进行——Lark 构造时直接传入 `transformer=self._transformer`，即 `parse()` 返回时树已经被规约成领域对象，没有单独的 transform 阶段。
2. **共享 typed list 解析**：`TypedListParser.parse_typed_list`（`pddl/parser/typed_list_parser.py:74-147`）被 `DomainTransformer.typed_list_name`（`pddl/parser/domain.py:477-483`）、变量 typed list（`pddl/parser/domain.py:485-498`）、函数 typed list（`pddl/parser/domain.py:500-506`）以及 problem 侧经复用调用（`pddl/parser/problem.py:85-94`）共用。
3. **共享逻辑节点构造**：problem parser 在其 transformer 内部 new 了一个 `DomainTransformer`（`pddl/parser/problem.py:49`），把 goal 的 `gd`、数值表达式 `f_exp`/`f_head`/`num_literal`、原子公式 `atomic_formula_term` 直接**委托**给 domain transformer 的同名方法（`pddl/parser/problem.py:144-147,167-177,210-212`）。注意：这不是共享实例，每次 `ProblemParser()` 都有一个独立的内部 `DomainTransformer()`。
4. **共享对象模型与后置校验**：`Domain`/`Problem` 构造函数在对象装配完成后立即跑一致性校验（`pddl/core.py:87`、`pddl/core.py:238`），校验逻辑集中在 `pddl/_validation.py`（`Types`、`Functions`、`TypeChecker`）。
5. **plan 不共享**：`PlanTransformer` 完全独立（`pddl/parser/plan.py:24-44`），不经过 domain/problem 的任何类型或要求校验；`Plan.check(domain, problem)` 是事后手动调用（`pddl/core.py:423-441`，CLI 在 `pddl/__main__.py:51` 调用）。

“缓存/复用”实际只发生在以下位置：

- **无 parser 缓存**：每次 `DomainParser()`/`ProblemParser()` 都重新读文法并建 LALR 表；`pddl.parse_*` 三个 helper 每次调用都 new parser（`pddl/__init__.py:31-52`）。测试里靠 session 级 fixture 复用（`tests/conftest.py:92-101`）。
- **同实例 transformer 状态复用**：一个 parser 实例的 transformer 在多次 `__call__` 之间不重置（见风险 R2）。
- **字符串驻留式复用**：`RegexConstrainedString.__new__` 对同类型同值直接返回原对象（`pddl/helpers/base.py:151-157`）。
- **hash 缓存**：`@cache_hash` 把首算的 `__hash__` 缓存在实例属性 `__hash` 上（`pddl/helpers/cache_hash.py:19-37`），装饰在 `Term`、`Predicate`、`NumericFunction`、公式、effects 等类上。
- **集合化存储**：domain 的 constants/predicates/actions/derived_predicates 都经 `ensure_set` 变 `frozenset`（`pddl/core.py:81-84`，`pddl/helpers/base.py:63-72`）；量词变量也是 frozenset（`pddl/logic/base.py:209`）。这决定了 formatter 输出顺序（见第 5、6 节）。

---

## 2. Domain 主干调用链

以 `DomainParser()(text)`（或 `parse_domain(path)` → `pddl/__init__.py:36`）为例：

1. `BaseParser.__call__` → `_call_parser` → `Lark.parse`（`pddl/parser/base.py:44-69`，同时临时把 `sys.tracebacklimit` 置 0）。
2. 文法起点 `domain`（`grammar.lark:3`）。规约顺序即文法小节顺序：`domain_def` → 可选 `requirements` → `types` → `constants` → `predicates` → `functions` → 若干 `structure_def`（action/derived）。
3. 各回调（均在 `pddl/parser/domain.py`）：
   - `domain_def:108-110` → `{"name": ...}`；
   - `requirements:112-117`：`Requirements(r[1:])` 去冒号转枚举，并立即用 `_extend_domain_requirements`（`pddl/requirements.py:87-100`）展开 `:adl`/`:quantified-preconditions`/`:fluents` 的隐含要求，缓存到 transformer 的 `self._extended_requirements`，供后续所有 `gd_or/gd_imply/gd_quantifiers/...` 用；
   - `types:119-135`：有父类型却无 `:typing` 立即抛 `PDDLMissingRequirementError`；把父类型 `"object"` 规范成 `None`；并把类型映射**缓存到 `self._types`**，供 derived predicate 的子类型检查使用（`types_hierarchy` 属性在 `pddl/parser/domain.py:80-84`）；
   - `constants:137-142`：调用 `TypedListParser` 得到 `{name: parent}`，实例化 `Constant` 并缓存进 `self._constants_by_name`，使 action/precondition 里的同名裸名能回查到同一个 `Constant` 实例（`constant:415-421`，未知裸名抛 lark `ParseError`）；
   - `predicates:144-148`：`atomic_formula_skeleton:431-435` 经 `_formula_skeleton:423-429` 产出 `Predicate(Variable...)`，缓存到 `self._predicates_by_name`；
   - `functions:150-154`：`f_typed_list_atomic_function_skeleton:500-506` 把 `NumericFunction -> "number"/None` 收进 dict；`atomic_function_skeleton:437-445` 特判 `total-cost`；
   - `action_def:156-166`：`action_parameters:223-228` 先把参数实例化并缓存到 `self._current_parameters_by_name`；`action_body_def` 是一棵 Tree，靠“`:precondition` 偶数位、内容奇数位”的下标配对取出 precondition/effect，构造 `Action`（`pddl/action.py:27-45`）；
   - 公式里的变量/常量统一经 `_constant_or_variable:395-400`：裸常量查常量缓存，变量名查当前 action 参数缓存，**查不到就 `Variable(name, {})` 当作自由变量**（见 R4）；
   - `derived_predicates:168-221`：见第 4 节。
4. 顶层 `domain:90-106`：把 dict 小节 `kwargs.update` 合并，把 `Action`/`DerivedPredicate` 分桶，`self._types = None`（只清类型表，见 R2），最后 `return Domain(**kwargs)`。
5. **对象构造 + 后置校验**：`Domain.__init__`（`pddl/core.py:51-87`）里依次：
   - `parse_name`（`pddl/custom_types.py:48-62`，正则 + 保留字）；
   - `Types(types, requirements)`（`pddl/_validation.py:94-106`）→ `to_types` 规范化键值 → `_check_types_dictionary:126-192`：无 `:typing` 却有父类型、`object` 有父类型、层级有环（`find_cycle`，`pddl/helpers/base.py:173-193`）三类错误；
   - `Functions(...)`（`pddl/_validation.py:325-336`）→ `_check_total_cost:356-385` 校验 `total-cost`/`:action-costs` 与 `:numeric-fluents`；
   - `_check_consistency:89-96`：`TypeChecker.check_type` 递归检查 constants、predicates（`pddl/_validation.py:238-319` 的 singledispatch 注册表）、actions 的参数/前提/效果中每个 term 的 type tag 是否落在 `Types.all_types` 里（`_find_inconsistencies_in_typed_terms:54-70`），再检查 derived predicate 头部（`pddl/core.py:98-105`）。

## 3. Problem 与 Plan 主干调用链

**Problem**（文法 `problem`，`grammar.lark:107`；小节顺序固定：`problem_def`、`problem_domain`、可选 `requirements`、可选 `objects`、`init`、`goal`、可选 `metric_spec`）：

1. `ProblemTransformer.problem:56-63` 断言首尾是 `(define`/`)`，把小节作为 kwargs 构造 `Problem`。
2. `problem_domain:69-71` 只返回字符串 `domain_name`——**解析期不引用 domain 对象**。
3. `objects:77-83`：委托内部 domain transformer 的 typed-list 逻辑后，实例化 `Constant(name, type_tag)`，缓存到 `self._objects_by_name`（独立于内部 domain transformer 的常量表）。
4. `init:101-108` 拍平列表；`init_el:110-121` 处理 `(= (f args) num)`；`literal_name:123-130` 处理 `(not ...)`；`basic_function_term:132-138` 对函数参数一律新建**无类型** `Constant(x)`；`atomic_formula_name:149-165`：非等号分支里，对象名能在 `_objects_by_name` 查到就复用带类型的实例，否则新建无类型 `Constant`。
5. goal/metric：`gd`/`f_exp`/`f_head`/`num_literal`/`atomic_formula_term` 全部委托内部 `DomainTransformer`（`pddl/parser/problem.py:144-212`）；metric 单独走 `metric_spec:179-186` 与 `metric_f_exp:188-208`（复制了 domain 的数值算子分派）。
6. **对象构造 + 后置校验**：`Problem.__init__`（`pddl/core.py:197-238`）解析 domain/domain_name（`_parse_domain_and_domain_name:240-261`）、requirements（`_parse_requirements:263-285`：problem 必须是 domain 要求的子集）、init 必须全是 literal（`is_literal`，`pddl/logic/base.py:282-297`）。**只有构造时拿到了 `Domain` 实例才会调用 `check`**（`pddl/core.py:287-306`）：domain 名匹配（小写比较）、requirements 子集、用 domain 类型重建 `Types(skip_checks=True)` 后对 objects/init/goal 跑 `TypeChecker`。
7. 也可事后 `problem.check(domain)`（`pddl/core.py:292-306`）或经 `problem.domain = domain` setter（`pddl/core.py:319-324`）触发。CLI 在 `pddl/__main__.py:46` 这样做。

**Plan**（文法 `plan`，`grammar.lark:148-149`）：

`ground_action`（`pddl/parser/plan.py:31-39`）→ `(action_name, [Constant...])`；`plan:41-44` → `Plan(actions=...)`。动作名保持为 lark **`Token('NAME', ...)`**（`str` 子类，未走 `parse_name`/保留字检查）；参数都是无类型 `Constant`。`Plan` 自身不校验任何东西；`Plan.check`（`pddl/core.py:423-441`）事后只查三件事：动作名在 domain 中、参数个数与 schema 一致、参数 ∈ problem.objects ∪ domain.constants——**不查参数类型**。

---

## 4. 各类校验在何时发生

| 校验内容 | 发生时机 | 位置 |
| --- | --- | --- |
| 名字词法（`[A-Za-z][-A-Za-z0-9_]*`） | 两层：Lark `NAME`（`grammar.lark:157`）在词法层；对象构造时 `name` 再做一次 fullmatch | `pddl/custom_types.py:23-30`，`pddl/helpers/base.py:140-170` |
| 保留字不能作名字/类型 | 仅对象模型层：`parse_name`/`parse_type` → `_check_not_a_keyword`（`object` 对类型豁免、`total-cost` 对函数名豁免） | `pddl/custom_types.py:48-96,117-132`；保留字表 `pddl/parser/symbols.py:22-74` |
| 类型层级：父类型需 `:typing`、`object` 无父、无环 | domain `Domain` 构造时（解析期 `types` 回调还做一次早期 `:typing` 检查） | `pddl/_validation.py:126-192`；`pddl/parser/domain.py:119-135` |
| typed list 重名/冲突类型 | **解析期**（常量、类型、objects 不允许重名；变量允许重名但冲突 tag 报错） | `pddl/parser/typed_list_parser.py:176-214`；变量例外在 `pddl/parser/domain.py:495` 传 `allow_duplicates=True` |
| 常量必须先声明 | 解析期：公式中出现裸名时查 `_constants_by_name` | `pddl/parser/domain.py:415-421` |
| 谓词变量类型存在 | domain 构造后：`TypeChecker` 递归 predicates/actions/derived | `pddl/core.py:89-105`；`pddl/_validation.py:238-319` |
| action 参数类型 | 同上（参数本身 + 前提/效果里同名变量经 `_constant_or_variable` 复用参数实例，tag 随参数走） | `pddl/parser/domain.py:223-228,395-400` |
| derived predicate：谓词已声明、元数一致、变量 tag 是头部 tag 的子类型 | 解析期（transformer 内，先于 `Domain` 构造） | `pddl/parser/domain.py:168-221` + `_check_subtypes:534-552` |
| requirements（`:or`/`:imply`/量词/`=`/`:oneof` 等） | 解析期，按构造节点即时查 `_extended_requirements` | `pddl/parser/domain.py:238-362,402-409`；展开逻辑 `pddl/requirements.py:87-100` |
| problem 引用 domain（名字、requirements 子集、objects/init/goal 类型） | **不在解析期**；仅在持有 `Domain` 实例时由 `Problem.check`/setter/构造触发 | `pddl/core.py:287-306` |
| init 只能是 literal | `Problem` 构造时（无 domain 也查） | `pddl/core.py:233-236` |
| plan 的动作名/参数个数/参数归属 | 事后手动 `Plan.check`，且不校验参数类型 | `pddl/core.py:423-441` |

**大小写处理**（实测）：

- 文法对关键字和 `NAME` 都是**大小写敏感**的字面匹配（Lark 默认无 `i` 标志）：`:REQUIREMENTS`、`:Action`、大写 `AND` 作小节关键字都会 `UnexpectedCharacters`；但 `AND`、`Move`、`MOVE` 作谓词名/动作名完全合法（它们走 `NAME` 分支）。实测见第 8 节风险外的说明性输入。
- 唯一的大小写宽松点是 **problem↔domain 名字比较**：`self.domain_name.lower() == domain.name.lower()`（`pddl/core.py:295`），有专门测试 `tests/test_problem.py:164-169`。其余所有比较（谓词名、对象名、plan 动作名查表）都区分大小写。
- 名字正则同时允许大小写（`grammar.lark:157`），类型/对象名按原样保留（如 `Truck` 保持 `Truck`）。

**保留字处理小结**：保留字由两层共同拦截。词法层让 `and`/`not`/`or`/`exists`/... 在文法位置上优先成为关键字终端（contextual lexer 按文法位置选择）；名字层 `parse_name` 再拦一遍，保证即使手工 `Domain(...)`/`Predicate("and")` 构造对象也会抛 `PDDLValidationError: it is a keyword`（测试：`tests/test_parser/test_domain.py:153-184`、`tests/test_types.py`）。`:strips` 这类以冒号开头的要求永远匹配不了 `NAME`，天然安全。

---

## 5. 样本追踪：嵌套数值运算 + 量词 + typed object 的 parse→format

domain 样本（实测可完整 parse-format-parse，除类型规范化外，见第 7 节）：

```lisp
(define (domain sample)
  (:requirements :strips :typing :numeric-fluents :universal-preconditions)
  (:types truck car - vehicle)
  (:predicates (at ?v - vehicle ?l))
  (:functions (fuel ?v - vehicle) (dist))
  (:action drive
     :parameters (?v - truck ?l)
     :precondition (and (at ?v ?l) (>= (fuel ?v) (* 2 (+ (dist) 3))))
     :effect (decrease (fuel ?v) (+ 1 (* 2 (dist)))))
  (:action refuel
     :parameters (?v - car)
     :precondition (forall (?x - truck) (>= (fuel ?x) 0))
     :effect (increase (fuel ?v) 1)))
```

**解析后的对象树**（类与 `pddl/logic/functions.py`、`pddl/logic/base.py` 对应）：

- `Domain.types = {name('truck'): name('vehicle'), name('car'): name('vehicle')}`——注意 `vehicle` 不在 dict 里（隐式根类型，见 R5）。
- `drive.parameters = (Variable('v',{'truck'}), Variable('l',frozenset()))`。
- precondition 是 `And`（flatten 后 2 个 operands）：`Predicate('at', v, l)` 与 `GreaterEqualThan(NumericFunction('fuel', v), Times(NumericValue(2), Plus(NumericFunction('dist'), NumericValue(3))))`。
  - 数值节点来自 `f_exp` 回调（`pddl/parser/domain.py:447-467`）；`*` 走 grammar 的多目规则 `LPAR multi_op f_exp f_exp+ RPAR`（`grammar.lark:80`），`+` 同理，`>=` 走 `gd` 的 binary_comp 分支（`grammar.lark:38`，回调 `gd_comparison:294-311`）。
  - `And`/`Or` 经 `BinaryLogicOpMetaclass` 做同算子 flatten + 去重（`pddl/logic/base.py:152-160,300-322`）；数值 `Plus/Times/Minus/Divide` 也带同一个 metaclass（`functions.py:387-440`），所以 `(+ (+ a b) c)` 会被压平成一个 `Plus(a,b,c)`——**结构会变，语义不变**。
- refuel 的 precondition 是 `ForallCondition(condition=GreaterEqualThan(...), variables=frozenset({Variable('x',{'truck'})}))`（`gd_quantifiers:273-292`）。

**formatter 写回路径**（`Domain.__str__`，`pddl/core.py:161-191`；实现全在 `pddl/formatter.py`）：

1. 每个集合先 `sorted(map(str, ...))`（`formatter.py:52-53`）：requirements、actions、derived predicates、init 元素都按**字符串**排序，因此两个 action 输出按名字排成 `drive`、`refuel`，与输入顺序无关。
2. `:types` 由 `print_types_or_functions_with_parents:59-71` 反查 dict（child→parent）聚成 parent→[children]，再由 `print_typed_lists:125-213` 做拓扑：先把所有根类型（含无 parent 和 parent 为 `object` 的）合成一行 `... - object`，其余 parent 按“子类型先于父类型被定义”的依赖顺序逐行输出。这就是为什么 `truck car - vehicle` 被写成两行：`vehicle - object` 在前、`car truck - vehicle` 在后。
3. constants/objects 经 `print_constants:74-83` 按 type tag 分组，组按类型名排序、组内按名字排序，无类型的放最后（`formatter.py:187-211`）。
4. 谓词骨架 `print_predicates_with_types:86-105`：谓词按 `Predicate.__lt__`（名字+terms）排序；每个变量无论原文是否写了 `- type`，formatter 都**强制重写**成 `?v - vehicle` 的带 tag 形式（tag 来自对象模型，不是原文），多 tag 写成 `(either ...)`。
5. 公式自身的括号是**语义括号**：每个节点的 `__str__` 都给自己加一对括号（`BinaryOp.__str__` `base.py:73-75`、`UnaryOp:113-114`、`BinaryFunction:175-177`、`Predicate:88-93`、`QuantifiedCondition:225-234`）。因此 `(* 2 (+ (dist) 3))` 里三层括号分别属于 `Times`、`Plus`、`NumericFunction('dist')`；删任一层都会改变 AST。`fuel ?v` 外层 `(fuel ?v)` 属于 f_head 语义，`?v` 没有括号因为 term 不是节点。
6. **来自集合/映射的顺序**：`:types` 行顺序（拓扑）、行内类型名/子类型名（sorted）、`:functions` 键（dict，按 `NumericFunction.__lt__`，`functions.py:107-111`）、predicates/actions/requirements/init（sorted 字符串）、`either` 内 tag（sorted）都不是原文顺序。**保持原文顺序的只有序列**：谓词参数 terms（tuple，`predicates.py:53`）、二元/多元算子 operands（tuple，顺序敏感的 `-`、`/` 依赖它）、plan 的 action 列表（list，`core.py:404`）。
7. 量词变量块的顺序来自 **frozenset 迭代**（`base.py:233` 对 `self.variables` 直接 join）。单变量稳定；多变量时顺序随 `PYTHONHASHSEED` 变化（实测同文件五个 seed 输出五种排列）。这是“顺序来自集合”最危险的一处：输出不确定，但 parse 回来对象仍相等。

problem 侧同理：`(:objects t1 - truck c1 - car)` 输出按类型分组重排为 `c1 - car t1 - truck`；init 各元素按字符串排序；metric 经 `Metric.__str__`（`functions.py:231-233`）。problem 的嵌套数值运算与 domain 走同一批类（`metric_f_exp` 除外，它复制了一份分派，`problem.py:188-208`）。

---

## 6. 已确认的风险点（均经实际运行复现）

下列每条都给了最小输入、当前结果、合理预期与测试盲区。R1/R2/R3 是行为错误（R3 还会挂死），R4 是静默接受，R5/R6 是语义保真问题，R7/R8 是错误处理/覆盖缺口。

### R1. Problem goal 的 requirements 永远按空集合判定（已确认）

- 最小输入：
  ```lisp
  (define (problem p) (:domain d)
    (:requirements :disjunctive-preconditions)
    (:init ) (:goal (or (p) (q))))
  ```
- 当前结果：`PDDLMissingRequirementError: ... :disjunctive-preconditions not found.`；`(forall ...)`、`(= a b)` 在 problem 里显式声明对应 requirement 后同样必挂。
- 合理预期：problem 声明了（或绑定的 domain 含）该 requirement 时应接受。
- 根因：`ProblemTransformer` 持有的内部 `DomainTransformer`（`pddl/parser/problem.py:49`）从未接收 problem 自己的 `requirements` 回调（`requirements:73-75` 只构造返回 tuple，没有更新内部 transformer），故 `gd_or/gd_quantifiers/atomic_formula_term(=)` 读到的 `_extended_requirements` 恒为空（`domain.py:255-289,404-406`）。
- 测试盲区：fixture 的 goal 基本只用 `and`/原子；带 `(or)`/量词 goal 的 `block-grouping` 域被 `DOMAIN_NAMES` 显式排除（`tests/conftest.py:43-71`），磁盘上 `tests/fixtures/pddl_files/block-grouping/p11.pddl` 实测在当前 parser 上直接抛同样异常。

### R2. parser 实例跨文件复用会泄漏上一份文件的常量与 requirements（已确认）

- 最小输入：同一个 `DomainParser()` 先解析含 `(:constants c1)` 的 d1，再解析没有声明 `c1` 却在动作里用了 `(p c1)` 的 d2。
- 当前结果：d2 被接受，`c1` 被解析成 d1 遗留的 `Constant(c1)`；全新 parser 解析 d2 则正确抛 `ParseError: Constant 'c1' not defined.`。requirements 同理：先解析一个 `:adl` 域后，再解析无任何 requirements 却用 `(or ...)` 的域也被放行。
- 合理预期：每次 `__call__` 之间解析上下文相互独立（或明确文档化“不可跨输入复用”）。
- 根因：`DomainTransformer.__init__` 的状态（`domain.py:71-77`）只在实例化时初始化；顶层 `domain` 回调只重置 `self._types = None`（`domain.py:105`），`_constants_by_name`、`_predicates_by_name`、`_requirements`、`_extended_requirements`、`_current_parameters_by_name` 均残留。
- 测试盲区：session 级 fixture（`tests/conftest.py:92-95`）鼓励单实例复用，但所有用例恰好互不冲突；没有“同一实例连续解析两份相互矛盾输入”的测试。

### R3. derived predicate 子类型检查在不匹配链上死循环（已确认）

- 最小输入：
  ```lisp
  (define (domain d)
    (:requirements :typing)
    (:types root c - object b1 - c)
    (:predicates (P ?v - root))
    (:derived (P ?v - b1) (P ?v)))
  ```
- 当前结果：解析**永久挂起**（5 秒 alarm 实测 Timeout），不是报错。
- 合理预期：抛 `PDDLParsingError`（类型不兼容）。
- 根因：`_check_subtypes`（`pddl/parser/domain.py:543-552`）循环里更新的是 `parent_type`，但下一次查询仍用不变的 `current_type`：`parent_type = self.types_hierarchy.get(current_type, None)`——当 `current_type` 的父不在右集合且父不是 None 时，每次都取到同一个父值，死循环。
- 测试盲区：`test_variables_types_propagated_in_derived_predicate_complex_hierarchy`（`tests/test_parser/test_domain.py:545-573`）只覆盖合法子类型（child1→root 命中、child3→root 命中），没有“链长 ≥2 且最终不命中”的用例。

### R4. action 前提/效果里未绑定的 `?变量` 被静默当成自由变量（已确认）

- 最小输入：参数声明 `?x`，前提写成 `(p ?y)`。
- 当前结果：解析成功，`?y` 成为 `Variable('y', frozenset())`，无任何警告。
- 合理预期：至少给出未声明变量错误（PDDL 动作里不存在自由变量）。
- 根因：`_constant_or_variable`（`pddl/parser/domain.py:395-400`）的 fallback 分支注释自认“free variable (bug)”，无条件造 `Variable(str(t), {})`。
- 测试盲区：现有测试（如 `tests/test_parser/test_domain.py:287-325`）只验证参数 tag 正确传播，没有拼写错误变量的负例。

### R5. 隐式根类型破坏 domain 的 parse-format-parse 对象相等性（已确认，已加回归测试）

- 最小输入：`(:types truck car - vehicle)`（`vehicle` 不显式写 `- object`）。
- 当前结果：首次 parse 得 `{'truck':'vehicle','car':'vehicle'}`（无 `vehicle` 键）；formatter 输出补成 `vehicle - object`（`formatter.py:145-155` 把无 parent 键统一挂 object）；再次 parse 得 `{'vehicle':None,'truck':'vehicle','car':'vehicle'}`，两次对象不相等，但两次解析都成功。
- 合理预期：要么首次 parse 就把隐式根规范化为 `vehicle -> None`，使 round-trip 相等；要么文档化该规范化。
- 根因：typed list 只记录被命名的 child（`typed_list_parser.py:60-68`），父名仅作为 value 出现，不会单独建键；而 `DomainTransformer.types` 只把字面量 `"object"` 转 `None`（`domain.py:126-128`）。
- 测试盲区：所有 fixture 都显式写了 `- object`（如 `barman/.../domain.pddl:3`、`triangle-tireworld/domain.pddl:3-5`），`test_domain_formatter`（`tests/test_formatter.py:38-46`）因此全绿；手工构造域的 `test_typed_constants_formatting_in_domain` 又显式传了完整 dict。

### R6. 空 precondition/effect 被构造成 `Or()`，格式化成非法文本 `(or )`（已确认）

- 最小输入：`:precondition ()`。
- 当前结果：`emptyor_pregd` 在 2 个 token（即空括号）时返回 `Or()`（`pddl/parser/domain.py:230-236`），`str` 为 `(or )`；`(or )` 因 grammar 里 `gd*` 后紧跟 RPAR 无法重新 parse。空 effect 同样（`emptyor_effect:335-340`）。
- 合理预期：空前提语义为真，应是空 `And()`（项目其他位置如 `Problem.goal` 默认值就是 `And()`，`pddl/core.py:231`），且能 round-trip。
- 测试盲区：没有任何测试用 `(:precondition ())`；formatter round-trip 测试只覆盖 fixture，fixture 里无此写法。

### R7. 谓词元数/签名在 domain 内与 problem.check 中都不校验（已确认）

- 最小输入：声明 `(p ?x)`，动作前提写 `(p ?y ?y)`；或 problem init/goal 引用从未声明的谓词 `(undeclared)`。
- 当前结果：domain 解析成功（声明元数 1、使用元数 2 并存）；`Problem.check(Domain('d'))` 对完全未声明谓词也通过。
- 合理预期：按 PDDL 静态语义拒绝元数不一致/谓词未声明。
- 根因：`TypeChecker` 只检查 term 的 **type tag 是否存在于类型表**（`_validation.py:238-247`），从不比对 `self._predicates_by_name` 中声明的元数/签名；problem 侧根本拿不到谓词签名（`check` 只用了 `domain.types`，`core.py:302-306`）。
- 测试盲区：测试覆盖“变量类型不存在”（`test_variables_typed_with_not_available_types`），但没有元数不一致或未声明谓词的负例。

### R8. 畸形数字 `1.2.3` 抛出未包装的 `ValueError`（已确认）

- 最小输入：`:precondition (>= (f) 1.2.3)`（`:numeric-fluents` 已声明）。
- 当前结果：grammar 的 `NUMBER: /[0-9]+(\.[0-9]+)*/`（`grammar.lark:217`）接受 `1.2.3`，`num_literal` 里 `float(n)`（`domain.py:330-333`）抛裸 `ValueError: could not convert string to float: Token('NUMBER', '1.2.3')`，既不是 `PDDLParsingError` 也不是 lark 解析错误。
- 合理预期：词法拒绝或转成 `PDDLParsingError`（其它 typed-list 错误都做了包装，`domain.py:508-514`）。
- 测试盲区：`test_number_parsing`（`tests/test_parser/test_domain.py:650-700`）只测 `int`/单小数点 `float`。

### 其他实测观察（非确定缺陷，仅记录）

- problem 的 `:objects` 与 domain 的 `:constants` 不支持 `(either t1 t2)` 联合类型（grammar 的 `typed_list_name` 只接 `primitive_type`，`grammar.lark:98-99,102`）；只有谓词/动作变量的 `type_def` 支持 either（`:100-101`）。这是 PDDL 合法子集缺口而非实现 bug。
- domain/problem 小节顺序被 grammar 写死（`:functions` 必须在 `:predicates` 之后等），调换顺序即解析失败。
- init 数值不接受负数（`init_el` 只接 `num_literal`，`grammar.lark:116`），而 goal/metric 接受 unary minus。
- `gd_comparison` 的要求检查写成 `if not bool({Requirements.NUMERIC_FLUENTS, Requirements.FLUENTS})`（`domain.py:296`）——恒为 False 的死条件；真正兜底来自 `Functions._check_total_cost` 对 `:functions` 小节的检查。同理 `atomic_function_skeleton` 的 `if not bool({Requirements.ACTION_COSTS})`（`:440`）恒不触发。
- plan 动作名是未校验的 lark `Token`，`(and a b)` 可被解析成名为 `and` 的动作；`Plan.check` 也不查参数类型。
- domain 名匹配只对 problem↔domain 名称做 `lower()`，其它名字一律大小写敏感。

---

## 7. Domain 与 problem 校验的依赖顺序

**Domain 解析期（transformer 规约，顺序由 grammar 小节顺序决定）**：

```
requirements 读入并展开 (_extend)
   └─> types（:typing 早期检查；object->None；缓存 self._types）
         └─> constants（TypedList 重名检查；建 _constants_by_name）
               └─> predicates（变量 typed list；建 _predicates_by_name）
                     └─> functions（建 NumericFunction->type）
                           └─> action/derived 逐条：
                                 action_parameters 建参数表
                                   -> gd/effect 节点构造时即时查 requirements
                                   -> term 经 _constant_or_variable 绑定常量/参数
                                 derived：谓词存在性 -> 元数 -> _check_subtypes（用 self._types）
```

**Domain 对象构造后的后置校验（`pddl/core.py:87`）**：

```
parse_name/parse_type（正则+保留字）
  -> Types：:typing 一致性 -> object 无父 -> find_cycle 无环
  -> Functions：total-cost/:action-costs -> :numeric-fluents
  -> TypeChecker：constants -> predicates -> actions(参数, 前提, 效果)
  -> derived_predicates 头部 terms 的类型
```

**Problem**：解析期只有 typed-list 重名检查与 init literal 检查；**对 domain 的全部依赖都推迟到 `check(domain)`**（构造时传入 domain、setter 或 CLI `pddl/__main__.py:46`）：

```
domain_name.lower() 匹配
  -> problem.requirements ⊆ domain.requirements
  -> Types(domain.types, skip_checks=True)
  -> TypeChecker：objects -> init -> goal（只查 type tag 是否在 domain 类型表）
```

注意 domain 的层级/环检查在 problem 侧以 `skip_checks=True` 跳过（`core.py:302`）——domain 已被假定为合法。谓词签名、元数、函数声明、action 参数绑定在 problem 检查链中**完全不存在**（R7）。

## 8. parse-format-parse 能保证与不能保证什么

以 `D2 = DomainParser()(str(D1))` 为例（测试形态见 `tests/test_formatter.py:38-46`）：

**能保证**：

- 对 grammar 支持且 fixture 覆盖的构造，`D1 == D2`（对象相等比较见 `core.py:147-159`）：名称、requirements 集合、类型 child→parent 映射、常量/谓词/函数/动作/axiom 的逻辑结构相等；序列（terms、operands、plan 动作）顺序保持。
- formatter 输出一定是可被同一 parser 再次解析的合法文本（括号都来自节点自身的 `__str__`）。
- 数值表达式的嵌套结构与算子顺序保持，包括一元/二元 `-`（`tests/test_parser/test_domain.py:834-848`）与 metric（`tests/test_parser/test_problem.py:236-255`）。

**不能保证**：

- **文本一致**：集合全部排序重排（requirements、predicates、actions、init、objects/constants 分组），types 拓扑重排，谓词变量被强制补全 `- type`；注释、空白、大小写风格全部丢失。
- **隐式根类型信息无损**：R5——`a - b`（b 隐式根）经一轮后 b 被显式挂到 object，对象 dict 变化。
- **flatten 前的嵌套形状**：连续同算子（`And/Or/Plus/Times/...`）在第一次 parse 时就已被 metaclass 压平，无法靠 round-trip 恢复括号嵌套（语义等价、结构不等价于原文树）。
- **多变量量词块的输出确定性**：变量存 frozenset，跨 `PYTHONHASHSEED` 文本顺序会变（对象仍相等）。
- **语义正确性**：round-trip 通过不代表输入符合 PDDL 规范——未声明谓词/元数不符（R7）、自由变量（R4）、空前提 `Or()`（R6）都可能完成 round-trip（`(or )` 除外，它根本无法二遍 parse）。
- **跨域引用正确**：problem 单独 round-trip 只验证字符串层面，`domain_name` 不会被解析成 `Domain`，类型/谓词一致性要另行 `check(domain)`；而 goal 中的 or/量词/等号在当前版本下甚至无法单独 parse（R1）。

## 9. 新增的回归测试

新增一条小测试 `test_domain_implicit_root_type_normalized_on_roundtrip`（`tests/test_formatter.py`），锁定 R5 这个已确认不变量：首次 parse 的 types dict 不含隐式根 `vehicle`，formatter 输出含 `vehicle - object`，二遍 parse 后 dict 多了 `vehicle: None` 且两对象不相等。它断言的是 parser/formatter 内部的规范化行为（typed-list 只建 child 键、formatter 补 object 父），不是任何公开接口的正常用法，因此可证明分析深入到了 transformer 与 formatter 的交互而非仅看公开 API。本次只加测试，未改动 `pddl/` 下任何生产代码。

## 10. 复现实验入口

- 安装：`python3 -m venv .venv && . .venv/bin/activate && pip install -e . pytest`
- 全量测试：`. .venv/bin/activate && pytest -q`（基线 2275 passed，新增 1 条后 2276 passed）
- 本文 R1–R8 均可由第 6 节给出的最小输入直接喂给对应 parser 复现。
