**解释器模式（Interpreter）是 GoF 行为型模式之一：给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。做法是文法里的每条规则对应一个类，句子被表示成由这些类的对象组成的抽象语法树（AST），对根节点调用 `interpret(context)` 就会递归地求出整句话的值。**

> 参考：GoF《设计模式》Interpreter 一章（示例是正则表达式文法 `RegularExpression` / `LiteralExpression` / `AlternationExpression` / `SequenceExpression` / `RepetitionExpression`，以及布尔表达式 `BooleanExp`）；Java 官方文档中 `java.util.regex.Pattern`。refactoring.guru 的模式目录没有收录解释器。代码沿用原笔记的 Java。

## 为什么要有解释器

有一类问题会以各种变体反复出现：按规则筛选数据、匹配字符串模式、计算用户配置的公式、判断一个技能能否释放的条件组合。如果每种变体都写死在代码里，每次规则变化都要改代码、重新发布。另一种思路是：设计一门很小的语言来描述这些规则，程序只需要实现“执行这门语言”的能力，规则本身变成数据，可以写在配置文件里、由策划或用户编辑。

GoF 的例子是正则表达式。与其为每个要搜索的模式写一个专门的匹配函数，不如定义正则表达式的文法：

```text
expression  ::= literal | alternation | sequence | repetition | '(' expression ')'
alternation ::= expression '|' expression
sequence    ::= expression '&' expression
repetition  ::= expression '*'
literal     ::= 'a' | 'b' | ... { 'a' | 'b' | ... }*
```

每条规则对应一个类：`LiteralExpression`、`AlternationExpression`、`SequenceExpression`、`RepetitionExpression`，它们都继承自 `RegularExpression`。一个具体的正则表达式（句子）就是这些类的对象组成的一棵树，匹配就是对树根调用 `interpret`，每个节点按自己那条规则的语义处理，再递归交给子节点。

这里要分清解释器模式**不负责**的部分：把字符串 `"raining & (dogs | cats)*"` 变成语法树的**解析**工作，GoF 明确说不在模式范围内，可以手写递归下降解析器，也可以用解析器生成工具，甚至由客户端直接用代码拼出语法树。解释器模式只管“语法树怎么表示、怎么求值”。

## 结构

| 参与者 | 职责 | 示例中的类 |
| --- | --- | --- |
| AbstractExpression（抽象表达式） | 声明 `interpret(context)`，所有语法树节点共享 | `Expression` |
| TerminalExpression（终结符表达式） | 文法中的终结符，语法树的叶子 | `Attribute`（如 `red`、`circle`） |
| NonterminalExpression（非终结符表达式） | 每条组合规则一个类，持有子表达式，递归解释 | `And`、`Or`、`Not` |
| Context（上下文） | 解释时需要的全局信息，如变量取值 | `Set<String>`：被判断形状拥有的属性 |
| Client（客户端） | 构建（或解析出）语法树，调用解释 | `InterpreterDemo`（含一个小解析器） |

原笔记示例的类图：

![[解释器模式-1.png]]

从类图就能看出，非终结符持有子表达式、自身又是表达式，这正是组合模式。解释器模式可以理解为：**把组合模式用在一门语言的语法树上，并约定树上有一个求值操作**。

## Java 实现：判断形状的小规则语言

原示例用 `context.contains(data)` 做终结符判断，于是 `"RedCircle"` 因为同时包含子串 `Red` 和 `Circle` 被判为“红色的圆”；但同样的规则也会把 `"RedCircleOutline"`、甚至 `"BoredCircle"`（`Bored` 里含有子串 `red`）判成红色的圆，子串匹配作为语义太脆弱。下面改成：上下文是形状拥有的属性集合，终结符判断“是否有这个属性”；再加一个递归下降解析器，让规则可以写成字符串。

文法：

```text
expr   ::= term { "or" term }
term   ::= factor { "and" factor }
factor ::= "not" factor | "(" expr ")" | IDENT
```

```java
// InterpreterDemo.java
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

// 抽象表达式
interface Expression {
    boolean interpret(Set<String> context);
}

// 终结符：属性名
class Attribute implements Expression {
    private final String name;
    Attribute(String name) { this.name = name; }
    public boolean interpret(Set<String> ctx) { return ctx.contains(name); }
    public String toString() { return name; }
}

// 非终结符：每条组合规则一个类
class And implements Expression {
    private final Expression left, right;
    And(Expression l, Expression r) { left = l; right = r; }
    public boolean interpret(Set<String> ctx) { return left.interpret(ctx) && right.interpret(ctx); }
    public String toString() { return "(" + left + " AND " + right + ")"; }
}

class Or implements Expression {
    private final Expression left, right;
    Or(Expression l, Expression r) { left = l; right = r; }
    public boolean interpret(Set<String> ctx) { return left.interpret(ctx) || right.interpret(ctx); }
    public String toString() { return "(" + left + " OR " + right + ")"; }
}

class Not implements Expression {
    private final Expression inner;
    Not(Expression e) { inner = e; }
    public boolean interpret(Set<String> ctx) { return !inner.interpret(ctx); }
    public String toString() { return "NOT " + inner; }
}

// 解析器：不属于解释器模式本身，负责把字符串变成语法树
class Parser {
    private final List<String> tokens = new ArrayList<>();
    private int pos = 0;

    Parser(String src) {
        for (String t : src.replace("(", " ( ").replace(")", " ) ").trim().split("\\s+")) {
            tokens.add(t.toLowerCase());
        }
    }

    Expression parse() {
        Expression e = expr();
        if (pos != tokens.size()) throw new IllegalArgumentException("多余的符号: " + tokens.get(pos));
        return e;
    }

    private Expression expr() {
        Expression e = term();
        while (peek("or")) { pos++; e = new Or(e, term()); }
        return e;
    }

    private Expression term() {
        Expression e = factor();
        while (peek("and")) { pos++; e = new And(e, factor()); }
        return e;
    }

    private Expression factor() {
        if (pos >= tokens.size()) throw new IllegalArgumentException("表达式不完整");
        String t = tokens.get(pos++);
        if (t.equals("not")) return new Not(factor());
        if (t.equals("(")) {
            Expression e = expr();
            if (!peek(")")) throw new IllegalArgumentException("缺少右括号");
            pos++;
            return e;
        }
        return new Attribute(t);
    }

    private boolean peek(String s) {
        return pos < tokens.size() && tokens.get(pos).equals(s);
    }
}

public class InterpreterDemo {
    public static void main(String[] args) {
        Expression rule = new Parser("red and (circle or square) and not outline").parse();
        System.out.println("语法树: " + rule);

        List<Set<String>> shapes = List.of(
                Set.of("red", "circle"),
                Set.of("red", "square", "outline"),
                Set.of("blue", "circle"),
                Set.of("red", "rectangle"));
        for (Set<String> s : shapes) {
            System.out.println(s + " -> " + rule.interpret(s));
        }
    }
}
```

`Parser` 里 `expr` 调 `term`、`term` 调 `factor`，层级对应文法中的优先级：`not` 最紧，`and` 次之，`or` 最松，所以不加括号时 `a or b and c` 会被解析成 `a or (b and c)`。解析得到的树可以缓存起来反复解释，解析只做一次。

## 解释器和相近模式的关系

| 模式 | 关系 |
| --- | --- |
| 组合模式 | 抽象语法树就是一棵组合树，非终结符是容器，终结符是叶子 |
| 访问者 | 表达式节点种类稳定、但要做求值、打印、优化、类型检查等多种操作时，把这些操作移到访问者里，比在每个节点类里加方法更好 |
| 享元 | 语法树中大量重复的终结符（同一个变量名）可以共享 |
| 迭代器 | 可以用迭代器遍历语法树 |
| 过滤器 / 规格 | 可组合的筛选条件本质上是一门只有与、或、非的布尔小语言 |

## 真实例子

`java.util.regex.Pattern.compile` 把正则表达式字符串编译成内部的节点结构（在 OpenJDK 的实现中是 `Pattern` 的一系列内部 `Node` 子类），`Matcher` 匹配时沿这些节点执行，结构上和 GoF 的正则表达式示例一脉相承。SQL 的 `WHERE` 条件、Spring 的 SpEL、模板引擎里的表达式，都是“小语言 + 解释执行”。

游戏里，策划配置的条件和公式（技能释放条件、伤害公式、任务触发条件）很适合做成小语言。可视化脚本系统本质上也是在编辑语法树，只不过节点是拖出来的而不是解析出来的。

## 容易踩的坑

**文法变复杂后类爆炸。** 每条规则一个类，文法稍大就有几十上百个类，维护困难。GoF 也指出这个模式只适合简单文法；复杂文法应当用解析器生成工具（如 ANTLR）加专门的求值器。原笔记提到在 Java 中可以考虑 expression4J 之类的现成库，这个具体库的现状未核实，选型时以现有维护状况为准。

**效率问题。** 直接解释语法树比编译执行慢，对性能敏感的场景，通常先把树转换成更紧凑的形式（字节码、状态机）再执行。

**深层递归。** 很长的表达式（例如上千个 `or` 串起来）会生成很深的树，递归解释可能栈溢出。

**用子串匹配代替真正的词法分析。** 原示例的 `contains` 就是这个问题：没有把句子切分成记号，导致匹配语义含糊。

**把解析错误推迟到解释时。** 括号不配对、未知关键字应该在解析阶段就报错，并给出位置信息，而不是解释时得到一个莫名其妙的结果。

## 相关

[[ACAEG-组合模式（Composite Pattern）]] [[ACAFD-访问者模式（Visitor Pattern）]] [[ACAEE-享元模式（Flyweight Pattern）]] [[ACAFC-迭代器模式（Iterator Pattern）]] [[ACAEB-过滤器模式（Filter Pattern）]]
