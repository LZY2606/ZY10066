# PDDL 库解析—对象—校验链路分析

分析对象：仓库中的 `pddl` 包（PDDL 3.1 子集：STRIPS、typing、负/析取/量词前置条件、派生谓词、数值流、条件/非确定效果、action-costs）。
所有结论均通过直接运行代码验证（Python 3.13、`lark 1.1.x`、仓库 2275 个既有测试 + 本次新增 1 个回归测试全部通过）。文中每条主干链路给出 `文件:行号`。

---

## 1. 总体架构：三条入口共享什么

三个公开 parser 入口：

| 入口 | 文件 | start 规则 | Transformer | 产物 |
|---|---|---|---|---|
| `pddl.parse_domain(fn)` → `DomainParser()(text)` | `pddl/__init__.py:31`, `pddl/parser/domain.py:555` | `domain` (`grammar.lark:3`) | `DomainTransformer` (`domain.py:64`) | `pddl.core.Domain` |
| `pddl.parse_problem(fn)` → `ProblemParser()(text)` | `pddl/__init__.py:39`, `pddl/parser/problem.py:220` | `problem` (`grammar.lark:107`) | `ProblemTransformer` (`problem.py:42`) | `pddl.core.Problem` |
| `pddl.parse_plan(fn)` → `PlanParser()(text)` | `pddl/__init__.py:47`, `pddl/parser/plan.py:47` | `plan` (`grammar.lark:148`) | `PlanTransformer` (`plan.py:24`) | `pddl.core.Plan` |

共享关系（这是三条入口最关键的耦合点）：

1. **同一份 Lark grammar**：`pddl/parser/grammar.lark`，路径常量在 `pddl/parser/__init__.py:17-18`。三条入口在 `BaseParser.__init__` 中各自用 `Lark(GRAMMAR_FILE.read_text(), parser="lalr", import_paths=[PARSERS_DIRECTORY], start=..., transformer=...)` 构造 **独立** 的 LALR parser（`pddl/parser/base.py:36-42`）。grammar 中 domain / problem / plan 三块规则同时存在于同一个文件，靠不同 `start` 符号进入；逻辑节点规则（`gd`、`f_exp`、`typed_list_*`、`atomic_formula_term`）因此被 domain 与 problem 复用。
2. **problem 复用 domain 的 transformer 逻辑（对象级委托，而非继承）**：`ProblemTransformer.__init__` 内部 `self._domain_transformer = DomainTransformer()`（`problem.py:49`）。`typed_list_name`（`problem.py:94`）、`gd`（`problem.py:146`）、`num_literal`（`problem.py:169`）、`f_exp`（`problem.py:173`）、`f_head`（`problem.py:177`）、`atomic_formula_term`（`problem.py:212`）全部直接转发给这个**没有解析过任何 domain、requirements 集合为空、常量表为空**的内部 `DomainTransformer`。这一委托是第 6 节多个风险点的根源。
3. **typed list 解析共享**：domain 与 problem 的 `typed_list_name` 都调用 `TypedListParser.parse_typed_list`（`pddl/parser/typed_list_parser.py:75`；调用点 `domain.py:480`、`problem.py:94`）。变量表走 `get_typed_list_of_variables`（`typed_list_parser.py:70`），名字/函数表走 `get_typed_list_of_names`（`typed_list_parser.py:60`）。
4. **逻辑节点对象共享**：`pddl/logic/base.py`（`And/Or/Not/Imply/OneOf/ForallCondition/ExistsCondition`）、`pddl/logic/predicates.py`（`Predicate/EqualTo/DerivedPredicate`）、`pddl/logic/functions.py`（`NumericFunction/NumericValue/Plus/Times/Minus/UnaryMinus/Divide/比较算子/赋值算子/Metric`）、`pddl/logic/effects.py`（`When/Forall`）、`pddl/logic/terms.py`（`Constant/Variable`）。formatter 只有一份：`pddl/formatter.py`，被 `Domain.__str__`（`core.py:161`）与 `Problem.__str__`（`core.py:377`）调用；`Plan.__str__`（`core.py:454`）不经过 formatter。
5. **后置校验共享 `pddl/_validation.py`**：`Types`（类型字典合法性、`:typing`、`object` 不可有父类、环检测，`_validation.py:91` 与 `_validation.py:126`）、`TypeChecker`（对 term/谓词/函数/公式/action 做单分派类型标注检查，`_validation.py:195`）、`Functions`（`total-cost` 与 `:action-costs`/`:numeric-fluents` 的一致性，`_validation.py:322`、`_validation.py:356`）。

### 缓存与复用发生在哪里

- **Lark parser 与 transformer 的生命周期是“每个 parser 实例一个”**：`BaseParser.__init__`（`base.py:32-42`）在构造时一次性建好 `self._parser` 与 `self._transformer`，多次调用 `parser(text)` 复用同一对象（`base.py:44-46`）。但每次 `DomainParser()`/`ProblemParser()` 都会重新读文件、重新构建 LALR 表；**不存在模块级 parser 缓存**。`tests/conftest.py:92-101` 用 session 级 fixture 规避了重复构造，生产代码没有。
- **Transformer 内部状态随实例复用并在一次 domain 解析结束时部分重置**：`DomainTransformer` 持有 `_constants_by_name`、`_predicates_by_name`、`_functions_by_name`、`_current_parameters_by_name`、`_requirements`、`_extended_requirements`、`_types`（`domain.py:71-77`）。`domain()` 结尾只把 `self._types = None`（`domain.py:105`），其余表项不重置——所以同一个 `DomainParser` 实例连续解析两个 domain 时，上一个 domain 的常量/谓词表会残留在 transformer 中。
- **`hash` 缓存**：`@cache_hash`（`pddl/helpers/cache_hash.py:98`）装饰 `Term`（`terms.py:31`）、`Formula`、`NumericFunction`、`Predicate`、`DerivedPredicate`、`When/Forall` 等，首次 `__hash__` 后把结果缓存在实例属性 `__hash`（`cache_hash.py:32-35`），并定制 pickle 的 getstate/setstate 防止跨进程脏缓存（`cache_hash.py:40-96`）。
- **逻辑构造期的“代数化简”缓存式复用**：`And/Or` 使用元类 `BinaryLogicOpMetaclass`，构造时做去重与同算子提升（`base.py:152-160`、`base.py:300-321`），`And(p)` 直接返回 `p`，`And(And(a,b),c)` 被压平。这不是缓存但改变了树结构，分析 round-trip 时必须计入。
- **量词/forall-effect 的变量被收进集合**：`QuantifiedCondition.__init__` 的 `ensure_set(variables)`（`base.py:209`）和 `effects.Forall` 的 `ensure_set(variables)`（`effects.py:87`）。顺序信息在此丢失（见风险点 4）。
---

## 2. Domain 主干调用链

```
parse_domain(fn)                         pddl/__init__.py:31
  -> open/read
  -> DomainParser()(dtext)
       BaseParser.__init__               pddl/parser/base.py:32   (读 grammar.lark、建 LALR、装 DomainTransformer)
       BaseParser.__call__               base.py:44
       _call_parser -> Lark.parse        base.py:49-64           (临时把 sys.tracebacklimit=0)
         grammar 规则 domain             pddl/parser/grammar.lark:3
         transformer 回调（按 Lark 归约顺序，大致自底向上）：
           requirements()                domain.py:112   -> 记录 _requirements 并计算 _extended_requirements
           types()                       domain.py:119   -> :typing 校验 + "object" 父类归一为 None + 记录 self._types
           typed_list_name()             domain.py:477   -> TypedListParser.parse_typed_list(:75)
           constants()                   domain.py:137   -> 建 _constants_by_name: Dict[str, Constant]
           atomic_formula_skeleton()     domain.py:431   -> Predicate(name, *Variable(...))
           predicates()                  domain.py:144   -> 建 _predicates_by_name
           atomic_function_skeleton()    domain.py:437   -> NumericFunction（total-cost 特判 :439）
           functions()                   domain.py:150
           action_parameters()           domain.py:223   -> 建 _current_parameters_by_name
           gd / f_exp / atomic_formula_term / constant ... domain.py:402-475
           action_def()                  domain.py:156   -> Action(...)；从 Tree.children 成对取 :precondition/:effect
           derived_predicates()          domain.py:168   -> 必须在 _predicates_by_name 中、arity 一致、子类型检查
           domain()                      domain.py:90    -> Domain(**kwargs)；重置 self._types
  -> Domain.__init__                      pddl/core.py:51
       Types(types, requirements)        core.py:80 -> _validation.py:94
       Functions(functions, requirements) core.py:85 -> _validation.py:325
       _check_consistency()              core.py:89
         TypeChecker.check_type(constants/predicates/actions)  core.py:91-94, _validation.py:195
         _check_types_in_has_terms_objects(actions, all_types) core.py:95 -> _validation.py:73
         _check_types_in_derived_predicates()                  core.py:98-105
```

要点：

- **requirements 在 `requirements()` 回调时就被解析**（`domain.py:114-115`），随后所有构造节点（or/imply/quantifier/equality/oneof）按“当前已读 requirements”做门槛检查。因为 grammar 里 `[requirements]` 是 domain 第一个可选段，这在正常 PDDL 文本里总是先生效。扩展规则（`:adl` 展开、`:quantified-preconditions` 展开、`:fluents` 展开）在 `pddl/requirements.py:87-100`。
- **negative-preconditions 门槛被故意关掉了**：`gd_not`（`domain.py:238-247`）里检查逻辑被注释，留有 `# TODO temporary change; remove`（`domain.py:244-245`），即没有任何 requirement 时 `(not ...)` 也能过。这是已知的、源码中明确标注的临时行为，不应当成新缺陷。
- **两处“恒真”检查（死代码门槛）**：`gd_comparison` 的 `if not bool({Requirements.NUMERIC_FLUENTS, Requirements.FLUENTS}):`（`domain.py:296`）集合非空，永远不抛异常；`atomic_function_skeleton` 对 `total-cost` 的 `if not bool({Requirements.ACTION_COSTS}):`（`domain.py:439-441`）同理。真正的 requirement 约束由对象构造后的 `Functions._check_total_cost`（`_validation.py:356-385`）兜底。
- **常量解析依赖“先 constants 后 body”的归约顺序**：body 里的裸 `NAME` 走 grammar `constant: NAME`（`grammar.lark:62`），回调 `DomainTransformer.constant`（`domain.py:415-421`）在 `_constants_by_name` 中查表，查不到直接 `raise ParseError("Constant '...' not defined.")`。变量则经 `?NAME`（`grammar.lark:96`）与 `_constant_or_variable`（`domain.py:395-400`）：不是常量且不在当前 action 参数表中的名字，会被**静默**构造成无类型 `Variable(str(t), {})`（`domain.py:399`），不报错。
- **action 参数对象在参数表与 precondition/effect 间共享**：`_constant_or_variable` 命中 `_current_parameters_by_name` 时返回同一个 `Variable` 实例（`domain.py:400`）。因此 `action.parameters[0] is action.precondition 中的 ?x`（已实测为 `True`），类型标注只在参数表上给一次即传播到整个 action。
- **action_def 对缺失可选段不安全**：见风险点 3。
- **derived predicate 的处理**（`domain.py:168-221`）：(i) 谓词名必须已在 `:predicates` 中声明（`:176-177`）；(ii) arity 必须匹配（`:179-183`）；(iii) 当派生头里变量无类型时，从谓词定义拷贝类型；有类型时用 `_check_subtypes`（`:534-552`）沿 `self.types_hierarchy` 逐级向上找父类型；(iv) `update_type_tags`（`pddl/parser/_update_type_tags.py:32`）用 singledispatch 重建整棵条件公式树，把类型标注传播到内部变量出现处，量词节点会用内层变量遮蔽外层映射（`_update_type_tags.py:67-76`）。

## 3. Problem 主干调用链

```
parse_problem(fn)                        pddl/__init__.py:39
  -> ProblemParser()(ptext)
       grammar 规则 problem               grammar.lark:107
         problem_def / problem_domain     -> ("name", x) / ("domain_name", x)   problem.py:65-71
         requirements()                   problem.py:73   仅构造成 set 返回，不写入内部 DomainTransformer
         objects()/typed_list_name()      problem.py:77/85,94
         init()/init_el()/literal_name()/basic_function_term()/atomic_formula_name()
                                          problem.py:101-138,149
         goal() -> gd() -> 转发内部 DomainTransformer.gd   problem.py:140-147
         metric_spec()/metric_f_exp()     problem.py:179-208
         problem()                        problem.py:56 -> Problem(**dict(...))
  -> Problem.__init__                     pddl/core.py:197
       _parse_domain_and_domain_name      core.py:240  (parser 只传 domain_name，故 self._domain is None)
       _parse_requirements                core.py:263
       validate(init 全为 literal)        core.py:233-236
       _check_consistency                 core.py:287  (domain is None => 什么都不做)
```

关键点：**`ProblemParser` 产出的 `Problem` 不携带 `Domain` 对象**（构造时只有 `domain_name`），因此 `_check_consistency` 是空操作（`core.py:289-290`）。problem 对 domain 的所有引用一致性校验只能事后由调用方显式执行 `problem.check(domain)`（`core.py:292-306`）——CLI 在 `pddl/__main__.py:46` 这么做，`tests/test_parser/test_parametrized.py:35` 这么做，但单独调 `ProblemParser()(...)` 的用户不会得到任何跨文件校验。

`Problem.check(domain)` 实际检查：
1. 名字匹配，且**只此处大小写不敏感**：`self.domain_name.lower() == domain.name.lower()`（`core.py:295`）；
2. problem requirements 是 domain requirements 的子集（`core.py:298-301`）；注意条件是 `self._requirements is None or ...`，若 problem 没写 `:requirements` 则整段跳过；
3. 用 domain 的类型表新建 `Types(..., skip_checks=True)` + `TypeChecker`，递归检查 objects / init / goal（`core.py:302-306`）。

它**不**检查：谓词/函数是否声明、谓词 arity、谓词参数类型与论域类型是否对应（`TypeChecker` 只看 term 上的类型标注是否“存在于类型表”，不与谓词签名比对；见 `_validation.py:238-252`）。

## 4. Plan 主干调用链

```
parse_plan(fn)                           pddl/__init__.py:47
  -> PlanParser()(text)
       grammar 规则 plan/ground_action    grammar.lark:148-149
       PlanTransformer.ground_action      plan.py:31-39
         action_name = args[1]；其余 NAME 包成 Constant(str)（无类型、无校验）
       plan()                             plan.py:41 -> Plan(actions=[(name, [Constant...]), ...])
```

Plan 解析**没有任何后置校验**：不查 action 是否存在、参数个数、参数是否是 object/constant。校验只存在于 `Plan.check(domain, problem)`（`core.py:423-441`：动作名、参数个数、参数 ∈ problem.objects ∪ domain.constants）和 `Plan.instantiate(domain)`（`core.py:411-421`，注意实例化按 `zip(parameters)` 对位替换，不做类型检查）。`Plan.check` 自身也不检查参数类型与 action 签名相容。
---

## 5. 各类“校验时机”一览

| 被验证的内容 | 何时验证 | 位置 | 不通过时 |
|---|---|---|---|
| 名字词法（`NAME` 正则 `[a-zA-Z][a-zA-Z0-9-_]*`） | Lark 词法扫描 | `grammar.lark:157`；对象层 `name.REGEX` 在 `custom_types.py:30` | `UnexpectedCharacters` |
| 保留字不能做名字/类型 | typed list 落对象时（`parse_name`/`parse_type`），即 `Domain`/`Action`/`Predicate`/`Term` 构造时 | `custom_types.py:48-79,117-132`；`typed_list_parser.py:170-173` | `PDDLValidationError("... it is a keyword")` |
| typed list 重复名字/重复类型标注 | 解析 typed list 时（constants/types/objects 默认不允许重复；变量表允许重复） | `typed_list_parser.py:54-58,176-214`；`allow_duplicates` 传参点 `domain.py:495` | `PDDLParsingError`（`domain.py:508-514` 包装） |
| 类型层级（`:typing` 与父类型共存、`object` 无父、无环） | `Domain` 构造（parser 路径）以及任何手工 `Domain(...)` | `domain.py:119-135`（parser 期 `:typing` 前置检查、`object -> None` 归一）；`_validation.py:126-192`（对象期完整检查） | `PDDLMissingRequirementError` / `PDDLValidationError` |
| 常量是否声明 | domain body 归约到 `constant` 时 | `domain.py:415-421` | lark `ParseError` |
| 谓词变量（skeleton）类型是否在 all_types 中 | `Domain._check_consistency` | `core.py:93` -> `_validation.py:73-88,244-247` | `PDDLValidationError` |
| 同一谓词中同名变量类型一致 | 构造 `Predicate`/`EqualTo` 时 | `predicates.py:26-42,54,130` | `ValueError` |
| action 参数类型可用、precondition/effect 类型标注可用 | `Domain._check_consistency` | `core.py:94-95`；`_validation.py:314-319` | `PDDLValidationError` |
| action 内自由变量 | **不验证**；`_constant_or_variable` 静默造无类型变量 | `domain.py:395-400` | —（正文第 6 节末“附带观察”记录，未列为缺陷） |
| derived predicate：谓词已声明、arity、头参数是父类型子类型、条件变量类型传播 | 解析 `:derived` 段时立即做 | `domain.py:168-221`、`domain.py:534-552`、`_update_type_tags.py` | lark `ParseError` / `PDDLParsingError` |
| derived condition 变量类型在 all_types 中 | `Domain._check_consistency`（对 dp.predicate 的 terms 检查） | `core.py:98-105` | `PDDLValidationError` |
| problem 引用 domain 的名字/requirements/objects/init/goal 类型 | **仅** `problem.check(domain)` 显式调用时 | `core.py:292-306` | `PDDLValidationError` |
| problem goal 中结构/requirement 门槛 | 解析期，但委托给一个“空状态”的 `DomainTransformer` | `problem.py:144-147` -> `domain.py:313-328` | 见风险点 1（大量误报） |
| init 必须全为 literal | `Problem.__init__` | `core.py:233-236`（`is_literal` 定义 `base.py:282-297`） | `PDDLValidationError` |
| functions 与 `:numeric-fluents`、`total-cost` 与 `:action-costs` | `Domain` 构造 | `_validation.py:356-385` | `PDDLValidationError` |
| plan 动作名/参数个数/参数是已知对象 | `plan.check(domain, problem)` 显式调用时 | `core.py:423-441` | `PDDLValidationError` |

### 大小写与保留字的处理（实测）

- grammar 终端全部按**小写字面量**定义（`DEFINE: "define"`、`AND: "and"` …，`grammar.lark:177-239`），Lark 词法默认大小写敏感；`NAME` 正则 `[a-zA-Z][a-zA-Z0-9-_]*`（`grammar.lark:157`）也能匹配大写单词。实测：
  - `(AND p p)` 中大写 `AND` 不匹配 `AND` 终端，被当作 `NAME`，于是整个式子被解析为**名为 `AND` 的谓词** `Predicate(AND, p, p)`（需 `p` 是常量才能完成词法归约；若 `p` 未声明为常量则会在 `constant` 处报 “Constant 'p' not defined.”——报错信息与真正原因无关）。
  - 对象层保留字检查对大小写敏感：`(:predicates (and))` 抛 `invalid name 'and': it is a keyword`，而 `(:predicates (AND))` 可被对象层接受（但 grammar 里 `(AND)` 作为 skeleton 时参数为空可以解析）。
- `ALL_SYMBOLS`（`symbols.py:74`）是保留字全集；`parse_name` 一律拒绝，`parse_type` 放行 `object`（`custom_types.py:78`），`parse_function` 放行 `total-cost`（`custom_types.py:95`）。`:requirements` 形式的关键字（`:strips` 等）以冒号开头，不匹配 `NAME`，不会撞名。
- 唯一的大小写不敏感点是 problem/domain 名字匹配（`core.py:295`，双侧 `.lower()`）。其余所有名字（谓词名、对象名、类型名）比较均大小写敏感。PDDL 规范传统上视名字大小写不敏感，这个库并未实现该约定。
- 连字符 `-` 同时是 `TYPE_SEP`（`grammar.lark:239`）和 `MINUS`（`grammar.lark:204`）：在 typed list 位置由 `typed_list_*` 规则消歧，在 `f_exp` 位置由数值规则消歧（`grammar.lark:78-83`，一元/二元减法靠括号内 f_exp 个数区分，见 `domain.py:454-456`）。名字内部允许连字符（如 `scale-up`、`total-cost`），靠“完整 token 优先匹配字面终端”工作；这也是 `NAME` 允许 `-` 却没有把类型分隔符吞掉的原因（Lark LALR 按上下文选终端）。
---

## 6. 已确认风险点（均有最小输入与实测结果）

下列每一项都在当前环境实跑确认；未跑过的猜测没有写入本节。

### 风险点 1：problem 单独解析时，goal 里的 `=`/`or`/`forall`/`exists` 必然误报缺 requirement，即使 problem 自己声明了对应 requirement

- **机理**：`ProblemTransformer.gd`（`problem.py:144-147`）把目标公式原样转发给 `self._domain_transformer`（`problem.py:49` 构造的独立 `DomainTransformer`），其 `_requirements`/`_extended_requirements` 永远是空集（构造函数 `domain.py:75-76`）；problem 自己的 `:requirements` 只在 `problem.py:73-75` 变成返回字典里的一个值，从不写回该内部 transformer。于是 `atomic_formula_term` 的 equality 门槛（`domain.py:404-406`）、`gd_or`（`domain.py:256-260`）、`gd_quantifiers`（`domain.py:285-289`）都按“无 requirement”判定。
- **最小输入**：
  ```
  (define (problem p) (:domain d) (:objects a b)
    (:init (p a)) (:goal (= a b)))
  ```
  （或 `(:goal (or (p a) (p b)))`、`(:goal (forall (?x) (p ?x)))`、`(:goal (exists (?x) (p ?x)))`；在 problem 里补 `:requirements :equality ...` 结果相同。）
- **当前结果**：`PDDLMissingRequirementError: Missing PDDL requirement, :equality not found.`（or/forall/exists 同理报各自的 requirement）。
- **合理预期**：problem 自带的 `:requirements` 应传播给委托 transformer（或要求传入 domain），至少显式声明了 requirement 时不应报错；这也与 `test_numeric_function_comparison_in_goal`（`tests/test_parser/test_problem.py:142`）能通过形成对比——数值比较走的是 `gd_comparison` 中那条恒真死分支（`domain.py:296`），恰好绕过了门槛。
- **测试为何没覆盖**：`tests/test_parser/test_problem.py` 的 goal 用例只覆盖数值比较（`:119-195`）和 metric；量词 goal 的解析测试全部在 domain parser 侧（`tests/test_parser/test_domain.py:273-540`）。端到端 fixture 测试 `test_problem_parser`（`tests/test_parser/test_parametrized.py:29-35`）是先 `problem_parser(...)` **再** `problem.check(domain)`——但 `check` 在解析之后才执行，救不回解析期的误报；被收集的 647 个 problem（`tests/conftest.py:73-80` 按 `DOMAIN_NAMES` 白名单 `:43-71` rglob）goal 中没有对象等式/析构/量词构造。反证就在仓库里：`tests/fixtures/pddl_files/block-grouping/p*.pddl` 的 goal 大量使用 `(or (not (= (x b1) (x b5))) ...)`，但 `block-grouping` 不在 `DOMAIN_NAMES` 白名单中（实测收集数为 0），直接 `ProblemParser()(p01.pddl)` 立即复现本缺陷（`PDDLMissingRequirementError: :disjunctive-preconditions not found`），整套测试因此从未碰到它。

### 风险点 2：`NUMBER` 词法允许多个小数点（如 `1.2.3`），解析期抛裸 `ValueError` 而非 PDDL 异常

- **机理**：`NUMBER: /[0-9]+(\.[0-9]+)*/`（`grammar.lark:217`）的 `*` 允许重复小数段；`num_literal` 回调 `float(n) if "." in n else int(n)`（`domain.py:330-333`，problem 经 `problem.py:167-169` 转发同一方法）对 `1.2.3` 调 `float()` 直接抛 Python 内置 `ValueError: could not convert string to float: Token('NUMBER', '1.2.3')`。
- **最小输入**：domain 的 `:precondition (>= (f) 1.2.3)`（或 problem 的 metric/goal/init 数值位置）。对照：`1.` 在词法层被拒（`1` 后 `.` 无法匹配任何终端），而 `1.2.3`、`00.1.2` 能被词法接受。
- **当前结果**：内置 `ValueError`（不是 `PDDLParsingError`/`ParseError`），消息直接泄露 Lark Token。
- **合理预期**：收紧为 `[0-9]+(\.[0-9]+)?` 或在回调中抛 `PDDLParsingError`；PDDL 3.1 的 number 语法只有整数与一段小数。
- **测试为何没覆盖**：数值相关测试（`tests/test_parser/test_domain.py:617-693` 的 `test_number_parsing` 等）只覆盖 `10/42/2.5/0.5` 与一/二元负号；没有任何用例构造畸形数字（`rg "1\\.2\\.3" tests` 无命中）。

### 风险点 3：action 缺少 `:precondition` 或 `:effect`（grammar 明确允许两者皆可选）时，transformer 抛 `TypeError: 'NoneType' object is not subscriptable`

- **机理**：grammar `?action_body_def: [PRECONDITION emptyor_pregd] [EFFECT emptyor_effect]`（`grammar.lark:25`）允许缺省，缺省子树在 Tree.children 中是两个连续的 `None`。`action_def` 无条件按成对切片：`{_children[i][1:]: _children[i+1] for i in range(0, len(_children), 2)}`（`domain.py:162-165`），对 `None[1:]` 取下标即崩。实测三种形态（只有 effect、只有 precondition、两者皆无）全部以同一 `TypeError` 失败。空括号形态 `:precondition ()` 则走 `emptyor_pregd`（`grammar.lark:29`）返回空 `Or()`（`domain.py:230-236`），可以正常解析。
- **最小输入**：
  ```
  (define (domain test) (:requirements :strips) (:predicates (p))
    (:action a :parameters () :effect (p)))
  ```
- **当前结果**：`TypeError: 'NoneType' object is not subscriptable`（经 `BaseParser._call_parser` 把 `tracebacklimit=0` 后，用户连栈都看不到，`base.py:61-68`）。
- **合理预期**：缺省段应等价于空条件（与 `emptyor_pregd` 的空 `Or()`、`Action(..., precondition=None, effect=None)` 对象模型一致——对象模型本身允许 `None`，见 `action.py:31-32,63-69`，且 `tests/test_action.py:34-36` 直接断言手工构造的 action `precondition is None`）。
- **测试为何没覆盖**：parser 侧所有 action 文本用例都同时给出 `:precondition` 与 `:effect`；对象层允许 `None`（`tests/test_action.py`），但没有任何测试把“缺省段文本 → parser”连起来。

### 风险点 4：多变量量词（及 forall-effect）的变量顺序随 `PYTHONHASHSEED` 非确定

- **机理**：parser 侧量词变量按文本顺序构造成 list（`domain.py:290`、`domain.py:355`），但 `ForallCondition/ExistsCondition` 构造时 `self._variables = ensure_set(variables)` 把它变成 `frozenset`（`base.py:209`）；`Forall` effect 同理（`effects.py:87`）。`__str__` 直接 `" ".join(self.variables)`（`base.py:233`、`effects.py:101` 经 `_typed_parameters`），因此输出顺序是集合迭代序。`Variable.__hash__` 只用名字（`terms.py:143-145`），不同名字的哈希序受随机哈希种子影响。实测同一进程模型在 `PYTHONHASHSEED=0/1/4` 下分别输出 `?beta ?gamma ?alpha` 与 `?alpha ?gamma ?beta`。
- **最小输入/脚本**：
  ```python
  ForallCondition(cond=Predicate("p", va, vb, vc), variables=[va, vb, vc])
  ```
  或解析含 `(forall (?a - ta ?b - tb ?c - tc) ...)` 的 domain 后 `str(...)`。
- **当前结果**：formatter 输出的量词变量块顺序跨进程不稳定（集合序）。语义上 round-trip 后对象仍相等（集合比较），所以 `test_domain_formatter`（`tests/test_formatter.py:38-46`）在 fixture 上不会失败——fixtures 中量词恰好都是单变量。
- **合理预期**：量词变量是有序序列（PDDL 语义里与条件中出现次序对应），应保留解析顺序（tuple/list），保证 formatter 输出稳定、可复现。
- **测试为何没覆盖**：`test_formatter.py:145-174` 的 forall 用例只有一个 `?neighbor`；domain parser 的多变量量词测试（`test_domain.py:460-540`）断言的是类型标注传播，从不比较 `str()`，也没有跨 hash-seed 的稳定性测试。

### 风险点 5：对内建类型 `object` 的显式标注在 term 位置一律被拒（parser 与 `Problem.check` 都受影响）

- **机理**：parser 的 `types()` 回调把所有以 `object` 为父的类型改写成 `None`（`domain.py:126-128`），因此 `Types.all_types = keys ∪ values - None`（`_validation.py:118-124`）不含字符串 `"object"`。随后 `TypeChecker` 对每个 term 调 `_check_types_are_available`（`_validation.py:218-225`），凡 term 显式带 `object` 标注就报“not in available types”。`:types` 段里写 `t - object` 没问题（走父类型归一），但 **term 位置**（谓词参数、action 参数、常量、派生谓词头）写 `?x - object` / `c - object` 全部失败；甚至无 `:typing` 时先在 `_check_typing_requirement`（`_validation.py:211-216`）报“typing requirement is not specified, but the following types were used: frozenset({'object'})”。Problem 侧 `(:objects x - object)` 能被 parser 接受（无 domain、不交叉校验），但随后 `problem.check(domain)` 在 `core.py:304` 用同一 `TypeChecker`，同样报 `types ['object'] ... are not in available types set()`（已实测，即使 domain 声明了 `:typing`）。
- **最小输入**：
  ```
  (define (domain test) (:requirements :strips :typing)
    (:predicates (p ?x - object))
    (:action a :parameters (?x - object) :precondition (p ?x) :effect (p ?x)))
  ```
- **当前结果**：`PDDLValidationError: types ['object'] of term Variable(x) are not in available types set()`。
- **合理预期**：`object` 是 PDDL 的内建根类型，term 显式标注 `- object` 与不标注等价，应被接受（`_check_types_are_available` 特判 `object`，或在 `all_types` 中保留它）。
- **测试为何没覆盖**：keyword 测试只验证 `object` 可作为**类型名**出现（`tests/test_types.py` 与 `test_domain.py:153-184`），全部 fixture 的 `object` 只出现在 `:types ... - object` 段（如 `barman/domain.pddl:3`），没有任何谓词/参数/常量写成 `- object`（对 `tests/fixtures` 做正则 `\?... - object` 零命中）。

### 风险点 6：problem goal 中引用的对象丢失其在 `:objects` 中声明的类型；未声明对象被静默造为无类型常量，交叉校验因此被架空

- **机理**：`:objects` 段构造的 `Constant(name, type_tag)` 存在 `_objects_by_name`（`problem.py:80-82`）。init 中的原子公式走 `atomic_formula_name`（`problem.py:149-165`），会查该表并复用带类型的实例（实测 identity 共享）；但 goal 经 `gd -> 内部 DomainTransformer.atomic_formula_term -> constant` 路径，problem 侧自己的 `constant` 回调（`problem.py:214-217`）**无条件** `return Constant(args[0])`——既不查 `_objects_by_name`，类型标签恒为空；内部 DomainTransformer 的 `_constants_by_name` 又是空表。后果：
  - goal 里 `t1`（声明为 `truck`）解析出的是无类型 `Constant(t1)`，与 `:objects` 里的实例不是同一对象（实测 `is False`）。
  - goal 里写一个根本不存在的对象 `ghost`，同样静默产生无类型 `Constant`，`problem.check` 的 `_check_types_are_available(∅, ...)` 对空标签集合直接放行（`_validation.py:66-69,218-225`）；`Plan.check` 才会查名字集合，`Problem.check` 不查。
- **最小输入**：
  ```
  (define (problem p) (:domain d) (:requirements :typing)
    (:objects t1 - truck) (:init) (:goal (p t1)))
  ```
  以及未声明对象：`(:init (p ghost))` 或 goal 中出现 `ghost`。
- **当前结果**：`problem.goal.terms` 中对象 `type_tags == frozenset()`（与声明不符）；未声明名字无任何告警/异常。
- **合理预期**：goal/init 中的名字应统一从 `:objects`（及 domain constants）解析，未知名字在交叉校验时报错；这样 `TypeChecker` 才能真正按声明类型核对 goal。
- **测试为何没覆盖**：problem parser 测试只断言数值公式与顶层 requirement（`tests/test_parser/test_problem.py`），从未检查 goal term 上的 type_tags；端到端 `test_problem_parser` 对 fixtures 做 `check`，但因为 goal term 标签为空，类型检查被空标签短路，而 fixtures 又都恰好类型“碰巧不错”。

### 附带观察（不计入“缺陷”，仅记录）

- action 前置/效果中出现既不是常量也不是参数的裸名（自由变量）会被静默构造成无类型 `Variable`（`domain.py:399`）；derived condition 同样如此。是否报错属于设计取舍，源码注释称“Case where the term is a free variable (bug) or comes from a parent quantifier”，未作定论，故仅记录。
- `ProblemTransformer.atomic_formula_name` 的 equality 分支（`problem.py:151-154`）用 `args[1]/args[2]` 取对象，而该分支的 `args[1]` 是 token `"="`、`args[2]` 才是第一个 term，属于不可达/错误死代码（grammar 的 init 等式走 `init_el` 的 `LPAR EQUAL_OP basic_function_term num_literal`，`grammar.lark:116`，不会产生 `atomic_formula_name` 的 `=` 形态）。
- `ProblemTransformer.domain__type_def`（`problem.py:96-99`）在当前 grammar 中没有对应规则名（grammar 内规则为 `type_def`，且只服务 domain 侧变量表），为不可达方法。
- 同一个 `DomainParser` 实例连续解析两个 domain 时，transformer 的 `_constants_by_name`/`_predicates_by_name` 不重置（仅 `_types` 在 `domain.py:105` 重置）。常规“一个 parser 解析一个文件”的用法不受影响，仅在复用时值得留意。

## 7. 综合样本追踪：嵌套数值运算 + 量词 + typed object 的 parse → format → parse

样本（实际可解析、可往返）：

```lisp
(define (domain sample)
  (:requirements :strips :typing :numeric-fluents :existential-preconditions :derived-predicates :action-costs)
  (:types truck car - vehicle
          location package vehicle - object)
  (:constants depot1 depot2 - location)
  (:predicates (at ?l - location ?o)
               (in ?p - package ?v - vehicle)
               (movable ?v - vehicle))
  (:functions (fuel ?v - vehicle) (total-cost))
  (:action drive
    :parameters (?v - vehicle ?from - location ?to - location)
    :precondition (and (at ?v ?from)
                       (>= (fuel ?v) (+ 1 (* 2 3) (/ 10 2))))
    :effect (and (not (at ?v ?from)) (at ?v ?to)
                 (decrease (fuel ?v) (* 2 (+ 3 1)))))
  (:derived (movable ?v - vehicle)
    (exists (?p - package) (in ?p ?v))))

(define (problem sample-p1)
  (:domain sample)
  (:requirements :strips :typing :numeric-fluents)
  (:objects t1 - truck p1 p2 - package)
  (:init (at depot1 t1) (= (fuel t1) 10))
  (:goal (>= (fuel t1) (+ 1 2)))
  (:metric minimize (* (total-cost) 2)))
```

（problem 目标刻意只用数值比较：量词/对象等式 goal 会触发风险点 1；这正说明该缺陷如何限制了样本选择。）

### 7.1 解析后的对象结构

`>=` 前置条件经 `gd`（`domain.py:313-328`）→ `gd_comparison`（`domain.py:294-311`）→ `GreaterEqualThan(left, right)`；右值 `(+ 1 (* 2 3) (/ 10 2))` 经 `f_exp`（`grammar.lark:78-83` → `domain.py:447-467`）：

```
GreaterEqualThan(
  NumericFunction("fuel", Variable("v", {"vehicle"})),   # f_head: domain.py:469-475
  Plus(
    NumericValue(1),                                      # num_literal: domain.py:330-333
    Times(NumericValue(2), NumericValue(3)),
    Divide(NumericValue(10), NumericValue(2))))
```

效果 `(decrease (fuel ?v) (* 2 (+ 3 1)))` 经 `num_effect`（`domain.py:380-393`）→ `Decrease(NumericFunction(...), Times(NumericValue(2), Plus(NumericValue(3), NumericValue(1))))`。
量词 `(exists (?p - package) (in ?p ?v))` 经 `gd_quantifiers`（`domain.py:273-292`）→ `ExistsCondition(cond=Predicate("in", ?p, ?v), variables=frozenset({Variable("p", {"package"})}))`；derived 头 `(movable ?v - vehicle)` 先在 `derived_predicates`（`domain.py:168-221`）与 `:predicates` 里的 `movable` 签名核对，再用 `update_type_tags` 把 `vehicle` 传播进 `(in ?p ?v)` 中的 `?v`。

类型层级被解析成字典（`types()` + `get_typed_list_of_names`）：
`{truck: vehicle, car: vehicle, location: None, package: None, vehicle: None}`（`- object` 归一为 `None`，`domain.py:126-128`）。
problem 侧 `t1 - truck, p1/p2 - package` 由 `objects()`（`problem.py:77-83`）建成带类型 `Constant`；init 里 `(at depot1 t1)` 经 `atomic_formula_name` 复用带类型实例，`(= (fuel t1) 10)` 经 `init_el`（`problem.py:110-121`）→ `EqualTo(NumericFunction("fuel", Constant("t1","truck")), NumericValue(10))`。

### 7.2 formatter 如何写回（哪些括号是语义，哪些顺序来自集合/映射）

`str(domain)`（`core.py:161-191`）/ `str(problem)`（`core.py:377-392`）的输出要点：

- **语义性括号**（由节点 `__str__` 产生，不可省）：`(define ...)`、每段 `(:requirements ...)` 等、原子公式 `(name term...)`（`predicates.py:88-93`、`functions.py:84-89`）、一元/二元逻辑与数值运算的前缀括号（`base.py:73-75,112-114`、`functions.py:175-177`、`functions.py:370-373`）、量词括号两层——外层 `(forall (vars...) cond)`（`base.py:225-234`）及变量表自身的括号、typed list 中 `(either t1 t2)` 的括号（`formatter.py:96`）、plan 每行 `(action args)`（`core.py:443-448`）。
- **来自集合/映射的顺序（非语义、不稳定或被排序稳定化）**：
  - requirements：`Domain._requirements`/problem requirements 是 `frozenset`（`core.py:79`），formatter 用 `sorted(...)` 稳定输出（`formatter.py:52-53`），故样本输出为字典序 `:action-costs :derived-predicates :existential-preconditions ...`，与输入顺序不同。
  - actions / derived / predicates：`Domain` 存的是 `frozenset`（`core.py:82-84`），输出统一经 `sort_and_print_collection` 按 `str` 排序（`formatter.py:29-56`）；predicates 内部还单独 `sorted(predicates)`（`formatter.py:89`）。
  - types：dict 按“父类型 → 子类型列表”反向分组（`formatter.py:59-71`），`print_typed_lists`（`formatter.py:125-213`）先输出无父类型并统一追加 ` - object`（`:140-155`），其余父类型按“子类型先定义”的拓扑顺序 + 名字排序输出（`:157-185`），遇环抛 `ValueError("Cycle in types")`（`:163`）。因此输入 `truck car - vehicle, location package vehicle - object` 被重排为 `location package vehicle - object` 然后 `car truck - vehicle`。
  - constants / objects：按类型 tag 分桶、tag 字典序排序、桶内名字排序，无类型者最后（`formatter.py:74-83,187-209`）。样本 problem 对象输出为 `p1 p2 - package t1 - truck`（而非输入的 `t1 ... p1 p2 ...`）。
  - init 是 `frozenset`（`core.py:230`），输出按字符串排序（`formatter.py:52-53`）：`(= (fuel t1) 10)` 排在 `(at depot1 t1)` 前。
  - functions：`Domain.functions` 是 dict（`core.py:85,128-130`），formatter 先按父类型（恒为 `number`/`None`）分桶再排序 skeleton（`formatter.py:59-71,108-122`）。
  - **量词变量块**：`QuantifiedCondition._variables` 是集合，`__str__` 不排序（`base.py:233`）——这是唯一没有被 formatter 排序稳定化的集合顺序（风险点 4）。
- typed list 的“成组”语义在写回时被重排：PDDL typed list 中 `- type` 作用于其**左侧**紧邻的名字组（grammar `typed_list_name: (NAME+ TYPE_SEP primitive_type)+ NAME*`，`grammar.lark:98-99`；解析实现 `typed_list_parser.py:124-145`）。formatter 不保留输入分组，只按类型重新分桶，因此等价但分组不同的文本（`a - t1 b - t2` vs 显式重写）会规范化为同一种输出。
实测 formatter 对样本的真实输出（原样）：

```text
(define (domain sample)
    (:requirements :action-costs :derived-predicates :existential-preconditions :numeric-fluents :strips :typing)
    (:types
        location package vehicle - object
        car truck - vehicle
    )
    (:constants depot1 depot2 - location)
    (:predicates (at ?l - location ?o)  (in ?p - package ?v - vehicle)  (movable ?v - vehicle))
    (:functions (fuel ?v - vehicle) (total-cost))
    (:derived (movable ?v - vehicle) (exists (?p - package) (in ?p ?v)))
    (:action drive
        :parameters (?v - vehicle ?from - location ?to - location)
        :precondition (and (at ?v ?from) (>= (fuel ?v) (+ 1 (* 2 3) (/ 10 2))))
        :effect (and (not (at ?v ?from)) (at ?v ?to) (decrease (fuel ?v) (* 2 (+ 3 1))))
    )
)
(define (problem sample-p1)
    (:domain sample)
    (:requirements :numeric-fluents :strips :typing)
    (:objects p1 p2 - package t1 - truck)
    (:init (= (fuel t1) 10) (at depot1 t1))
    (:goal (>= (fuel t1) (+ 1 2)))
    (:metric minimize (* (total-cost) 2))
```

（`:predicates` 行的双空格来自 `print_predicates_with_types` 每个 skeleton 后追加的分隔空白，`pddl/formatter.py:103-104`，不影响再解析。）

对该文本再次解析：`DomainParser()(str(d)) == d` 为 `True`，`ProblemParser()(str(p)) == p` 为 `True`（实测）。

## 8. domain 与 problem 的校验依赖顺序

```
解析期（transformer 内，随 Lark 归约自底向上、自前向后）
  grammar: domain_def -> requirements -> types -> constants -> predicates/functions -> (action | derived)*
                                                                          (grammar.lark:3)
  requirements 先落地 (_requirements/_extended_requirements, domain.py:114)
  types 落地并缓存 self._types (domain.py:133) ──┐
  constants 建表 _constants_by_name (domain.py:139)
  predicates 建表 _predicates_by_name (domain.py:147)
                                                 │
  action 参数建表 _current_parameters_by_name (domain.py:225)
    └─ body 内 term 解析依赖：constants 表 / 当前 action 参数表 / 已读 requirements / self._types
  derived 解析依赖：_predicates_by_name + 谓词签名 + self._types 层级 (domain.py:176-220)
                                                 ┘
Domain 对象构造期 (core.py:51-87)
  Types.__init__      : ->dict 转换 (custom_types.to_types, custom_types.py:109)
                       -> all_types = keys∪values−None (_validation.py:118)
                       -> :typing / object 无父 / 环 (_validation.py:126)
  Functions.__init__  -> total-cost/action-costs、numeric-fluents (_validation.py:336)
  _check_consistency  -> TypeChecker 扫 constants/predicates/actions/derived
                       (_validation.py:195；core.py:89-105)

problem 解析期 (problem.py)
  自身只做：typed-list 重复检查(委托 typed_list_name)、init literal 形状、metric 关键字
  gd/f_exp 全部委托空状态 DomainTransformer (problem.py:146,173,177,212)  ← 风险点 1/6
Problem 对象构造期 (core.py:197-238)
  init 必须全 literal (core.py:233)
  _check_consistency: 仅当持有 Domain 对象才执行 (core.py:289) —— parser 路径恒为 no-op
跨文件校验 (必须显式)
  problem.check(domain) (core.py:292):
    1) 名字 .lower() 相等
    2) problem.requirements ⊆ domain.requirements（problem 未声明则跳过）
    3) 用 domain.types 建 TypeChecker，扫 problem.objects/init/goal 的类型标签
  plan.check(domain, problem) (core.py:423):
    动作名存在 / 参数 arity / 参数 ∈ objects ∪ constants
```

依赖方向上的注意事项：

- `self._types` 必须在 constants/predicates 之前就绪，derived 的子类型检查（`domain.py:546-547`）才能沿父链查找；grammar 把 `[types]` 固定在 `[constants] [predicates]` 之前保证了这一点。若手工调整段落顺序（例如把 `:predicates` 放到 `:types` 前，grammar 不允许），整条假设失效。
- problem 端**没有**对应于 `self._types`/`_constants_by_name` 的 domain 上下文，这是设计上的单向依赖：problem parser 独立可用，但代价是第 6 节的两个 goal 相关缺陷；真正的语义校验被推迟到 `check(domain)`。
- requirement 门槛分布在两个层级：transformer 层（`gd_or/gd_imply/gd_quantifiers/atomic_formula_term(=)/c_effect(oneof)/types`）与对象层（`Types._check_types_dictionary`、`Functions._check_total_cost`）。两层依据的 requirement 状态不同源（transformer 用解析流中的 `_extended_requirements`；对象层用传入的集合），而 problem 委托链上第一层读到的是空集。

## 9. parse → format → parse 能保证与不能保证什么

**能保证（在当前实现 + 已测范围内）：**

- 对仓库 26 个 fixture domain 与全部 fixture problem，`parse(format(parse(x))) == parse(x)`（`tests/test_formatter.py:38-55`，本机 2276 测试全绿）。相等性基于对象模型 `__eq__`：`Domain.__eq__`（`core.py:147-159`）、`Problem.__eq__`（`core.py:363-375`），集合类字段按集合语义比较，因此**段落顺序、typed-list 分组、requirements/init/actions 的书写顺序**被规范化后不影响相等判定。
- 嵌套数值表达式的树形状（含一元/二元减号区分、`Plus/Times/Divide` 的多操作数与嵌套）在往返中保留——这正是近期专门补过的测试（`test_nested_minus_in_precondition_preserved`、`test_unary_minus_domain_round_trip`，`tests/test_parser/test_domain.py:792-840`；problem metric 对应 `tests/test_parser/test_problem.py:236-255`）。
- 类型层级（父子映射）、谓词/函数签名的变量类型标签、derived 谓词传播后的变量标签在往返中保留（formatter 对谓词按 term 逐个写 `- tag` / `(either ...)`，`formatter.py:86-105`）。
- 文本一定仍是**语法合法**的 PDDL（formatter 输出再解析失败会被同一个参数化测试捕获）。

**不能保证：**

- **文本级稳定/幂等的字节输出**：formatter 会重排 requirements、types、constants/objects、predicates、actions、init 的顺序；两次 format 相同对象虽然输出一致（排序确定），但与原始输入通常不同。它不是“保真打印”，是“规范化打印”。
- **量词变量块的跨进程稳定性**：多变量量词的变量顺序来自 frozenset 迭代序（风险点 4）；同一对象在不同 `PYTHONHASHSEED` 进程里 format 结果可不同。对象相等性不受影响，文本 diff/快照测试会受影响。
- **被 parser 拒绝的输入无法谈往返**：风险点 1（problem 的 `=`/`or`/量词 goal）、风险点 3（缺省 action 段直接 `TypeError`）、风险点 5（term 上的 `- object`）意味着这些合法 PDDL 形态根本走不到第一次 parse 成功。
- **语义正确性未被验证**：往返保持对象相等，但对象模型本身不校验谓词是否声明（body 内）、arity 一致性、参数类型与谓词签名相容、problem 谓词/函数是否属于 domain、自由变量、数字词法（`1.2.3` 在到达对象模型前就以非 PDDL 异常崩溃）。`problem.check(domain)` 也只覆盖类型标签存在性，不覆盖签名/arity。
- **大小写规范化不存在**：由于名字比较大小写敏感而 domain 名字匹配大小写不敏感（`core.py:295`），混用大小写的输入可能通过往返但在不同名字比较点表现不一致；大写关键字（`AND`）会被当作普通 NAME 谓词。
- **plan 的往返更弱**：`PlanParser` 产出的 `Constant` 无类型，`Plan` 无对象级校验，`str(plan)` 只保证重新解析得到同名同参数串的动作列表，不校验其对 domain/problem 的合法性。
