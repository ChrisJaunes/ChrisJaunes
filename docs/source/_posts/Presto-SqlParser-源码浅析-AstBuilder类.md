---
title: Presto-SqlParser-源码浅析-AstBuilder类
date: 2022-01-24 15:17:20
categories:
- [Presto, SqlParser]
tags:
- Presto
- Sql解析
excerpt: 介绍了Presto中的SqlParser模块下的AstBuilder类
---

# AstBuilder 浅析

语法树是一个复杂的结构，为了对这个复杂对象结构中的全部元素执行某些操作，可以使用[访问者模式](https://refactoringguru.cn/design-patterns/visitor)

Antlr为了支持访问者模式，需要让语法树中的每一个节点都支持accept方法, 因此在ParseTree接口中要求继承类实现accept方法

{% asset_img SqlBaseParser.svg [SqlBaseParser] %}

RuleContext 和 TerminalNodeImpl 提供了默认的实现， 默认时，Antlr生成的类重写了accept方法以便于找到访问者中正确的方法

{% spoiler "accept 实现相关源码" %}
```java
public class RuleContext implements RuleNode {
	...
	@Override
	public <T> T accept(ParseTreeVisitor<? extends T> visitor) { 
		return visitor.visitChildren(this); 
	}
	...
}
public class TerminalNodeImpl implements TerminalNode {
	...
	@Override
	public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
		return visitor.visitTerminal(this);
	}
	...
}
public static class NonReservedContext extends ParserRuleContext {
	@Override
	public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
		if ( visitor instanceof SqlBaseVisitor ) return ((SqlBaseVisitor<? extends T>)visitor).visitNonReserved(this);
		else return visitor.visitChildren(this);
	}
}
```
{% endspoiler %}

---

antlr 默认生成的 SqlBaseBaseVisitor 里 visitXXX 实际上调用了 visitChildren

{% spoiler "visitXXX 相关源码" %}
```java
public class SqlBaseBaseVisitor<T> extends AbstractParseTreeVisitor<T> implements SqlBaseVisitor<T> {
	@Override public T visitSingleStatement(SqlBaseParser.SingleStatementContext ctx) { 
		return visitChildren(ctx); 
	}
	...
}
public abstract class AbstractParseTreeVisitor<T> implements ParseTreeVisitor<T> {
	@Override
	public T visitChildren(RuleNode node) {
		T result = defaultResult();
		int n = node.getChildCount();
		for (int i=0; i<n; i++) {
			if (!shouldVisitNextChild(node, result)) {
				break;
			}
			ParseTree c = node.getChild(i);
			T childResult = c.accept(this);
			result = aggregateResult(result, childResult);
		}
		return result;
	}
	...
}
```
{% endspoiler %}

---

我们需要将Antlr产生的语法树进行语义分析，然后转换成Presto下的语法树，因此 AstBuilder 重写了 visitXXX 方法

{% asset_img AstBuilder.svg [AstBuilder] %}

SqlParser 调用 new AstBuilder(parsingOptions).visit(tree)
```java
public class SqlParser{
	private Node invokeParser(String name, String sql, Function<SqlBaseParser, ParserRuleContext> parseFunction, ParsingOptions parsingOptions) {
		...
		return new AstBuilder(parsingOptions).visit(tree);
		...
	}
	...
}
```
以后进入 AbstractParseTreeVisitor 中的 visit 方法
```java
public abstract class AbstractParseTreeVisitor<T> implements ParseTreeVisitor<T> {
	public T visit(ParseTree tree) {
		return tree.accept(this);
	}
	...
}
```

此处根据tree的类型，调用对应的accept

例如tree是SingleStatementContext的话, 就会调用 AstBuilder 中的 visitSingleStatement

{% spoiler "SingleStatementContext 相关源码" %}
```java
public class SqlBaseParser extends Parser {
	public static class SingleStatementContext extends ParserRuleContext {
		public StatementContext statement() {
			return getRuleContext(StatementContext.class,0);
		}
		public TerminalNode EOF() { return getToken(SqlBaseParser.EOF, 0); }
		public SingleStatementContext(ParserRuleContext parent, int invokingState) {
			super(parent, invokingState);
		}
		@Override 
		public int getRuleIndex() { return RULE_singleStatement; }
		@Override
		public void enterRule(ParseTreeListener listener) {
			if ( listener instanceof SqlBaseListener ) ((SqlBaseListener)listener).enterSingleStatement(this);
		}
		@Override
		public void exitRule(ParseTreeListener listener) {
			if ( listener instanceof SqlBaseListener ) ((SqlBaseListener)listener).exitSingleStatement(this);
		}
		@Override
		public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
			if ( visitor instanceof SqlBaseVisitor ) return ((SqlBaseVisitor<? extends T>)visitor).visitSingleStatement(this);
			else return visitor.visitChildren(this);
		}
	}
	...
}
class AstBuilder extends SqlBaseBaseVisitor<Node> {
	@Override
	public Node visitSingleStatement(SqlBaseParser.SingleStatementContext context) {
		return visit(context.statement());
	}
	...
}

```
{% endspoiler %}

例如tree是Query， 就会调用 AstBuilder 中的 visitQuery，然后构造成Presto的结构

{% spoiler "Query 相关源码" %}
```java
public class SqlBaseParser extends Parser {
	public static class QueryContext extends ParserRuleContext {
		public QueryNoWithContext queryNoWith() {
			return getRuleContext(QueryNoWithContext.class,0);
		}
		public WithContext with() {
			return getRuleContext(WithContext.class,0);
		}
		public QueryContext(ParserRuleContext parent, int invokingState) {
			super(parent, invokingState);
		}
		@Override public int getRuleIndex() { return RULE_query; }
		@Override
		public void enterRule(ParseTreeListener listener) {
			if ( listener instanceof SqlBaseListener ) ((SqlBaseListener)listener).enterQuery(this);
		}
		@Override
		public void exitRule(ParseTreeListener listener) {
			if ( listener instanceof SqlBaseListener ) ((SqlBaseListener)listener).exitQuery(this);
		}
		@Override
		public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
			if ( visitor instanceof SqlBaseVisitor ) return ((SqlBaseVisitor<? extends T>)visitor).visitQuery(this);
			else return visitor.visitChildren(this);
		}
	}
	...
}
class AstBuilder extends SqlBaseBaseVisitor<Node> {
	@Override
	public Node visitQuery(SqlBaseParser.QueryContext context) {
		Query body = (Query) visit(context.queryNoWith());
		return new Query(
			getLocation(context),
			visitIfPresent(context.with(), With.class),
			body.getQueryBody(),
			body.getOrderBy(),
			body.getOffset(),
			body.getLimit());
	}
	...
}

```
{% endspoiler %}
