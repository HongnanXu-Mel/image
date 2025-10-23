# Task 3: 基于模糊测试的漏洞发现

## 3.1 概述

在Task 3中，我们通过大规模模糊测试在SQLite 3.31版本中成功发现了**2个空指针解引用漏洞（Null Pointer Dereference, CWE-476）**。基于Task 2的经验，我们采用迭代优化策略，设计了针对性的种子文件和智能变异算子，将漏洞触发效率提升了500倍（从500万次降至约1万次）。

---

## 3.2 实现方法与决策理由

### 3.2.1 漏洞发现的三个阶段

**阶段1：初期探索（失败）**
- 使用Task 2的36个通用种子和3个自定义算子
- 运行超过500万次测试，持续3天
- **结果**：未发现任何漏洞
- **失败原因**：缺少针对特定漏洞模式的测试（如自引用生成列、窗口函数、复杂VIEW）

**阶段2：针对性改进（成功）**
- 添加2个针对性种子（task3_seed1.dat, task3_seed2.dat）
- 添加2个智能算子（mutate_view_based_patterns, mutate_window_aggregate_patterns）
- 运行超过500万次测试
- **结果**：成功发现2个不同的bug

**阶段3：优化验证（高效）**
- 优化种子和算子配置
- 仅使用Task 3种子和新算子
- **结果**：约1万次执行即可触发bug（效率提升500倍）

**决策理由：**
1. **为什么添加新种子**：分析已知CVE后发现缺少VIEW+生成列、窗口函数+嵌套查询的测试
2. **为什么设计新算子**：字节级变异无法生成复杂的SQL结构组合（如COALESCE+窗口函数+嵌套子查询）
3. **为什么选择这两类模式**：SQLite 3.31的已知bug集中在查询优化器和表达式生成器，这两类模式能够触发相关代码路径

### 3.2.2 新增种子设计

#### 3.2.2.1 task3_seed1.dat - VIEW与生成列种子

**1. Description**

该种子文件包含29行SQL语句，专门设计用于测试SQLite的VIEW、生成列（generated columns）和COALESCE函数的交互。核心内容包括：

- **自引用生成列表**：`CREATE TABLE products(prod_id, prod_name AS(prod_name) UNIQUE)`
- **多个带UNIQUE约束的表**：inventory和categories表
- **基础VIEW定义**：`CREATE VIEW inv_view AS SELECT item_id, item_name FROM inventory`
- **基础查询操作**：简单的SELECT和JOIN查询
- **插入操作**：为表提供基础数据

种子结构简洁清晰，每个表只包含必要的列，避免了不相关的SQL语句干扰。

**2. 为什么这样设计**

**设计动机1：触发VIEW与生成列的交互bug**
SQLite 3.31在处理包含生成列的VIEW时，查询优化器可能出现错误。自引用生成列（`col AS(col)`）创建了特殊的表达式树结构，当与VIEW的COALESCE函数组合时，容易触发WHERE表达式分析器的空指针问题。

**设计动机2：为变异算子提供材料**
- 提供多个表：让算子可以生成复杂的JOIN
- 包含UNIQUE约束：为算子创建复杂约束环境提供基础
- 包含VIEW：让算子可以在此基础上生成更复杂的VIEW
- 保持简洁：减少噪音，让变异聚焦于关键模式

**设计动机3：模拟真实场景**
使用产品目录、库存、分类等真实业务场景，使生成的SQL更接近实际应用，提高测试的实用性。

**设计动机4：配合mutate_view_based_patterns算子**
种子提供的表结构和VIEW定义，正是该算子需要的"原材料"。算子可以提取表名和列名，生成包含COALESCE和复杂WHERE条件的新VIEW。

#### 3.2.2.2 task3_seed2.dat - 窗口函数与聚合种子

**1. Description**

该种子文件包含30行SQL语句，专门设计用于测试SQLite的窗口函数、自连接和嵌套聚合。核心内容包括：

- **极简表结构**：`CREATE TABLE items(id UNIQUE)`仅包含必要的列
- **自连接查询模板**：`SELECT items.id FROM items JOIN items i2 ON items.id = i2.id`
- **聚合函数示例**：SUM、COUNT等基础聚合
- **子查询模式**：IN子查询和标量子查询
- **交叉表连接**：items与refs的JOIN操作

种子采用极简设计，每个表只有1-2个列，所有查询都是基础模式的简单组合。

**2. 为什么这样设计**

**设计动机1：触发窗口函数与嵌套聚合的bug**
SQLite 3.31在处理窗口函数（如lead() OVER()）与聚合函数（如SUM）的嵌套组合时，表达式代码生成器可能出现空指针问题。该种子提供了自连接和聚合的基础模式，供算子生成复杂嵌套。

**设计动机2：极简schema策略**
- 最少的列数：减少变异空间，聚焦核心模式
- 最简单的类型：避免类型相关的干扰
- 没有复杂约束：让变异聚焦于查询结构而非约束处理

**设计动机3：提供自连接模板**
自连接是触发Bug #002的关键。种子明确提供了自连接示例，让变异算子理解这种模式，并在此基础上生成更复杂的多层自连接和NATURAL JOIN组合。

**设计动机4：配合mutate_window_aggregate_patterns算子**
种子中的聚合函数（SUM、COUNT）和自连接模式，正是该算子需要的基础。算子会提取表名和列名，生成包含lead() OVER()、嵌套子查询和相关引用的复杂查询。

### 3.2.3 新增变异算子设计

#### 3.2.3.1 mutate_view_based_patterns算子

**1. Description**

该算子专门针对VIEW、生成列和COALESCE函数设计，实现了6种变异策略：

**策略1 - 创建自引用生成列表**：生成`CREATE TABLE tbl{random}(col, col AS(col) UNIQUE)`形式的表，其中列名与生成列表达式相同，形成自引用。

**策略2 - 创建多UNIQUE约束表**：生成包含多个UNIQUE约束的表，如`CREATE TABLE tbl{random}(col1 UNIQUE, col2 UNIQUE)`。

**策略3 - 创建含COALESCE的VIEW**：从现有表中提取列名，生成如`CREATE VIEW v{random}(c{random}) AS SELECT coalesce(col1, col2) FROM table WHERE col IN('VALUE')`的VIEW。

**策略4 - 生成VIEW与表的复杂JOIN**：结合现有VIEW和表，生成包含复杂WHERE条件的JOIN查询，如`SELECT * FROM view JOIN table WHERE c1 IS NULL AND view_col OR c2 = 'test'`。

**策略5 - IFNULL到COALESCE的转换**：将输入中的IFNULL函数替换为等价的COALESCE函数。

**策略6 - 向VIEW的SELECT添加COALESCE**：识别现有VIEW定义，自动在SELECT子句中包装列为COALESCE函数。

**2. 为什么这样设计**

**针对已知bug模式**：通过分析SQLite 3.31的已知CVE，我们发现WHERE表达式分析器在处理以下组合时存在潜在问题：
- 生成列的自引用会创建特殊的表达式树
- VIEW中使用COALESCE引用生成列时，查询展开变得复杂
- WHERE条件包含`IS NULL AND ... OR ...`模式时，优化器需要特殊处理

**组合多个高风险特征**：单一特征通常不会触发bug，需要多个特征组合。该算子能够系统性地生成VIEW+生成列+COALESCE+复杂WHERE的组合。

**保持语法正确性**：算子从现有表中提取真实的表名和列名，确保生成的SQL语法正确，能够深入执行阶段而不是在解析阶段失败。

**渐进式复杂度**：6种策略从简单到复杂，确保既能触发简单场景的bug，也能触发复杂组合的bug。

#### 3.2.3.2 mutate_window_aggregate_patterns算子

**1. Description**

该算子专门针对窗口函数、自连接和嵌套聚合设计，实现了5种变异策略：

**策略1 - 生成自连接查询**：创建表与自身的JOIN，如`SELECT table.col FROM table JOIN table alias ON value = table.col`，使用常量作为JOIN条件。

**策略2 - 生成NATURAL JOIN**：生成表的自然连接，如`SELECT * FROM table NATURAL JOIN table alias`，自动匹配同名列。

**策略3 - 添加窗口函数**：在查询中插入lead()窗口函数，如`SELECT lead(value) OVER() FROM table`。

**策略4 - 嵌套子查询与COALESCE+SUM组合**：生成三层嵌套结构，如`SELECT (SELECT coalesce(lead(n) OVER(), SUM(col))) FROM table alias`。

**策略5 - 复杂多重自连接与嵌套聚合WHERE**：组合所有高风险模式，生成包含显式JOIN、NATURAL JOIN、WHERE IN嵌套子查询、窗口函数和聚合函数的复杂查询。

**2. 为什么这样设计**

**针对表达式代码生成器**：SQLite的表达式代码生成器是最复杂的组件之一。窗口函数需要特殊的上下文处理，嵌套子查询需要正确的变量绑定，这些复杂性容易导致空指针问题。

**窗口函数的特殊性**：lead() OVER()访问下一行数据，在空结果集或特定上下文中可能导致未初始化的指针访问。与聚合函数SUM嵌套使用时，求值顺序和上下文管理更加复杂。

**多层自连接的复杂性**：表与自身多次连接会创建复杂的列绑定环境。当NATURAL JOIN与显式JOIN混用时，列名解析变得更加复杂，优化器容易出错。

**相关子查询的挑战**：内层子查询引用外层查询的列（如`WHERE items.id IN(... FROM items i9 WHERE items.id)`）需要特殊的变量传递机制，这是bug的高发区域。

**渐进式复杂度策略**：策略1-3生成相对简单的模式，策略4-5组合多个特征。这种设计确保了既能触发简单场景的bug，也能触发需要多个特征组合的复杂bug。

---

## 3.3 发现的漏洞

### 3.3.1 Bug #001: WHERE表达式分析器空指针解引用

**漏洞信息：**
- 类型：CWE-476 (Null Pointer Dereference)
- 崩溃位置：sqlite3.c:142576 `isAuxiliaryVtabOperator`
- Bug签名：cba0df1f110d863563528a26d5d3fca6

**触发模式：**
```sql
CREATE TABLE products(prod_id, prod_name AS(prod_name) UNIQUE);
CREATE VIEW v36(vc3) AS SELECT coalesce(prod_id, prod_name) 
FROM products WHERE prod_name IN('VALUE');
CREATE TABLE t617(c8 UNIQUE, c9 UNIQUE);
SELECT * FROM v36 JOIN t617 WHERE c8 IS NULL AND vc3 OR c9 = 'test';
```

**关键特征**：自引用生成列 + COALESCE的VIEW + 复杂WHERE条件（IS NULL AND ... OR ...）

**根本原因**：查询优化器在分析包含VIEW列的OR表达式时，未正确处理生成列引用，导致访问空指针。

### 3.3.2 Bug #002: 表达式代码生成器空指针解引用

**漏洞信息：**
- 类型：CWE-476 (Null Pointer Dereference)
- 崩溃位置：sqlite3.c:102733 `sqlite3ExprCodeTarget`
- Bug签名：7aeb314cb45a096e7441d6fac5eb993e

**触发模式：**
```sql
CREATE TABLE items(id UNIQUE);
SELECT items.id FROM items 
JOIN items i3 ON 5 = items.id 
NATURAL JOIN items 
WHERE items.id IN((SELECT(SELECT coalesce(lead(2) OVER(), SUM(id))) 
FROM items i9 WHERE items.id));
```

**关键特征**：多层自连接 + NATURAL JOIN + 窗口函数与聚合函数嵌套 + 相关子查询

**根本原因**：表达式代码生成器在处理嵌套子查询的窗口函数时，由于复杂的列绑定环境，表达式树节点未正确初始化，导致空指针解引用。

---

## 3.4 漏洞复现说明

### 3.4.1 环境准备

```bash
cd system
make clean
make sqlite3-asan  # 编译带AddressSanitizer的SQLite
```

### 3.4.2 手动触发漏洞

**Bug #001：**
```bash
./sqlite3-asan empty.db < ../results/bugs/bug_001_POC.dat
```

**Bug #002：**
```bash
./sqlite3-asan empty.db < ../results/bugs/bug_002_POC.dat
```

**预期结果**：AddressSanitizer报告SEGV错误并显示详细栈追踪。

### 3.4.3 通过模糊测试自动发现

**使用所有种子：**
```bash
python3 run_experiment.py \
    --fuzzer_type mutation_based \
    --feedback_enabled \
    --corpus seed-corpus \
    --runs 50000
```

**使用仅Task 3种子（更高效）：**
```bash
mkdir -p task3-corpus
cp seed-corpus/task3_seed*.dat task3-corpus/
python3 run_experiment.py \
    --fuzzer_type mutation_based \
    --feedback_enabled \
    --corpus task3-corpus \
    --runs 20000
```

**预期结果**：通常在1-2万次执行内触发bug，框架自动保存触发输入到bugs目录。

---

## 3.5 实验反思与经验总结

### 3.5.1 什么有效

**✓ 针对性种子设计**
- Task 3的简洁针对性种子优于Task 2的全面通用种子（在漏洞发现方面）
- 种子应该为算子提供"正确的建筑材料"

**✓ 智能的语义感知变异**
- 两个新算子能够生成极其复杂的SQL模式组合
- 比字节级变异更有效地触发深层bug

**✓ 迭代优化方法**
- 从500万次失败到1万次成功，关键是持续分析和改进
- 失败提供了改进方向

**✓ AddressSanitizer的使用**
- 将隐藏的内存错误转化为明确崩溃
- 提供详细栈追踪，便于分析

### 3.5.2 什么无效

**✗ 缺乏针对性的盲目测试**
- 初期500万次测试未发现漏洞
- 仅靠测试次数和运气是低效的

**✗ 过度通用的种子**
- Task 2的36个种子覆盖率高但无法发现漏洞
- 太多无关内容稀释了关键模式

**✗ 忽视已知漏洞模式**
- 应该先研究目标软件的已知CVE
- 了解高风险代码区域

### 3.5.3 模糊测试的优势

1. **高度自动化**：无需人工干预，可运行数百万次测试
2. **覆盖率引导**：自动探索新代码路径，提高效率
3. **发现复杂bug**：能够发现需要极其复杂模式组合的bug
4. **可扩展性**：易于添加新算子和种子

### 3.5.4 模糊测试的局限性

1. **依赖先验知识**：Task 3的成功依赖于对已知CVE的分析，对零日漏洞发现可能不够高效
2. **资源消耗大**：需要数万次测试，覆盖率收集降低速度
3. **覆盖率瓶颈**：在46%左右达到瓶颈，难以进一步提升
4. **只检测崩溃**：依赖ASAN，无法检测逻辑错误
5. **针对性与通用性矛盾**：针对性种子能发现特定bug但覆盖范围有限

### 3.5.5 改进建议

**立即可行：**
1. 结合语法生成和变异fuzzing
2. 实现上下文感知的变异
3. 集成更多sanitizers（UBSan、MSan）
4. 优化算子选择策略

**长期方向：**
1. 引入符号执行混合技术
2. 使用机器学习优化策略
3. 实现差分测试检测逻辑错误
4. 开发漏洞模式自动学习机制

---

## 3.6 总结

**主要成果：**
- 发现2个CWE-476漏洞，均可导致DoS
- 开发2个针对性种子和2个智能算子
- 漏洞触发效率提升500倍

**最高覆盖率：**
- Task 2最高覆盖率：sqlite3.c 46.2%（使用Interesting Values算子）

**技术贡献：**
- mutate_view_based_patterns：针对VIEW+生成列+COALESCE模式
- mutate_window_aggregate_patterns：针对窗口函数+自连接+嵌套聚合

**关键经验：**
1. 代码覆盖率≠漏洞发现能力，需要针对性策略
2. 智能变异远优于盲目测试
3. 迭代优化是关键：从失败中学习并持续改进
4. 语义完整性很重要：探索性变异优于破坏性变异

通过Task 2和Task 3，我们建立了从广泛探索到定向测试的系统化方法论，这对未来的安全测试工作具有重要指导意义。

---

## 附录：详细复现步骤

**环境准备：**
```bash
# 安装依赖
sudo apt-get install python3 python3-pip gcc make
pip3 install fuzzingbook matplotlib gcovr
```

**验证已发现的bug：**
```bash
cd system
make sqlite3-asan
./sqlite3-asan empty.db < ../results/bugs/bug_001_POC.dat  # Bug #001
./sqlite3-asan empty.db < ../results/bugs/bug_002_POC.dat  # Bug #002
```

**自动发现bug：**
```bash
# 使用所有种子
python3 run_experiment.py --fuzzer_type mutation_based \
    --feedback_enabled --corpus seed-corpus --runs 50000

# 使用仅Task 3种子（更高效）
mkdir task3-corpus
cp seed-corpus/task3_seed*.dat task3-corpus/
python3 run_experiment.py --fuzzer_type mutation_based \
    --feedback_enabled --corpus task3-corpus --runs 20000
```

**预期结果**：
- 手动验证应稳定触发ASAN错误
- 自动fuzzing通常在1-2万次内发现bug
- Bug保存在bugs/目录

**注意事项：**
- 必须使用sqlite3-asan（带AddressSanitizer）
- 使用空数据库文件（empty.db）
- ARM64使用python3，Intel使用python3.10
- 随机性可能导致发现时间有波动，但应在合理范围内（<10万次）
