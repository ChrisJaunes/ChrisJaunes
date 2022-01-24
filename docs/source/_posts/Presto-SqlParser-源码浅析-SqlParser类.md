---
title: Presto-SqlParser源码浅析-SqlParser类
date: 2022-01-23 11:53:57
categories:
- [Presto, SqlParser]
tags:
- Presto
- Sql解析
excerpt: 介绍了Presto中的SqlParser模块下的SqlParser类
---

# SqlParser 浅析

从编译的角度来看, 编译需要以下步骤
<img src="https://blog.ginshio.org/blog/CompilerPrinciple/compiler-steps.svg" alt="编译" style="background: white;">

解析SQL和上述略有不同， 产生语法树以后接着生成逻辑执行计划和物理执行计划，但之前的词法分析和语法分析步骤是一致的

而前面这几部分的由SqlParser管理，其中主要需要了解的方法是 invokeParser

---

首先 将要解析的 sql 从 String 转成 CharStreams

考虑到 SQL 大小写不敏感，因此将字符流转成不区分大小写的字符流。 CaseInsensitiveStream 采用了代理模式，并在 LA方法中添加了 将所有的字符变成了大写 的逻辑。
{% spoiler "CaseInsensitiveStream 源码" %}
```java
public class CaseInsensitiveStream implements CharStream {
    private final CharStream stream;
    public CaseInsensitiveStream(CharStream stream) { this.stream = stream; }

    @Override
    public String getText(Interval interval) { return stream.getText(interval); }

    @Override
    public void consume() { stream.consume(); }

    @Override
    public int LA(int i) {
        int result = stream.LA(i);
        switch (result) {
            case 0:
            case IntStream.EOF:
                return result;
            default:
                return Character.toUpperCase(result);
        }
    }

    @Override
    public int mark() { return stream.mark(); }

    @Override
    public void release(int marker) { stream.release(marker); }

    @Override
    public int index() { return stream.index(); }

    @Override
    public void seek(int index) { stream.seek(index); }

    @Override
    public int size() { return stream.size(); }

    @Override
    public String getSourceName() { return stream.getSourceName(); }
}
```
{% endspoiler %}

接着 将字符流送入词法分析器

SqlBaseLexer是Antlr根据[sqlbase.g4](https://github.com/prestodb/presto/blob/master/presto-parser/src/main/antlr4/com/facebook/presto/sql/parser/SqlBase.g4)生成的词法解析器，继承[Lexer](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/Lexer.java)，间接继承了[Recognizer<Integer, LexerATNSimulator>](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/Recognizer.java) 实现了[TokenSource](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/TokenSource.java)

---

经过词法分析器以后，产生符号流

接着 将符号流送入语法分析器

SqlBaseParser是Antlr根据[sqlbase.g4](https://github.com/prestodb/presto/blob/master/presto-parser/src/main/antlr4/com/facebook/presto/sql/parser/SqlBase.g4)生成的语法解析器，继承[Parser](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/Parser.java)，间接继承了[Recognizer<Integer, LexerATNSimulator>](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/Recognizer.java)

---

为了允许用户对于词法分析器和语法分析器做额外的定制，initializer接受词法分析器对象和语法分析器对象并对其操作，默认情况下是不进行任何操作

{% spoiler "SqlParser initializer 相关源码" %}
```java
public class SqlParser {
    private static final BiConsumer<SqlBaseLexer, SqlBaseParser> DEFAULT_PARSER_INITIALIZER = (SqlBaseLexer lexer, SqlBaseParser parser) -> {};
    
    private final BiConsumer<SqlBaseLexer, SqlBaseParser> initializer;
    
    @Inject
    public SqlParser(SqlParserOptions options) { this(options, DEFAULT_PARSER_INITIALIZER); }

    public SqlParser(SqlParserOptions options, BiConsumer<SqlBaseLexer, SqlBaseParser> initializer) {
        this.initializer = requireNonNull(initializer, "initializer is null");
        ...
    }

    private Node invokeParser(String name, String sql, Function<SqlBaseParser, ParserRuleContext> parseFunction, ParsingOptions parsingOptions) {
        ...
        initializer.accept(lexer, parser);
        ...
    }
    ...
}
```
{% endspoiler %}

---

Parser注册了 PostProcessor 对象 作为其 Listener, PostProcessor 继承于 SqlBaseBaseListener(由Antlr生成)，SqlBaseBaseListener 实现了 SqlBaseListener 接口

SqlBaseBaseListener 对语法树节点都提供了 enterXXX 和 exitXXX，不过这些默认都是空方法，也就是SqlBaseBaseListener其实什么都没有干

PostProcessor 重写了exitUnquotedIdentifier、exitBackQuotedIdentifier、exitDigitIdentifier、exitNonReserved 四个方法

1. exitUnquotedIdentifier 方法处理不带引号的标识符，当标识符存在不合法符号的时候抛出异常
    - 不合法符号由EnumSet.complementOf(allowedIdentifierSymbols)确定

    - allowedIdentifierSymbols 在 构造函数 中初始化，并且使用final仅禁止改变该变量的引用

    - 例如 select * from foo@bar 存在不合法符号 @，由此抛出“line 1:15: identifiers must not contain '@'”

2. exitBackQuotedIdentifier 方法处理反引号包裹的标识符，当存在反引号的时候抛出异常

    - 标准的Presto是不能使用反引号的

    - 例如 select * from \`foo\` 存在反引号，由此抛出“line 1:15: backquoted identifiers are not supported; use double quotes to quote identifiers”

3. exitDigitIdentifier 方法处理数字开头的标识符

    - 例如 select 1x from dual 存在不合法的1X， 由此抛出“line 1:8: identifiers must not start with a digit; surround the identifier with double quotes”

4. exitNonReserved 处理 NonReserved

    - 要求是叶子节点(TerminalNode)

    - 用 IDENT 标记替换 nonReserved 标记
    
    - 例如 select if(1=1,1,0) from foo 中的 if ，原本if的type类型是 SqlBaseLexer.IF (数值85)， 现改为 SqlBaseLexer.IDENTIFIER (数值231)

{% spoiler "PostProcessor 源码" %}
```java
public class SqlParser {
    private final EnumSet<IdentifierSymbol> allowedIdentifierSymbols;

    public SqlParser(SqlParserOptions options, BiConsumer<SqlBaseLexer, SqlBaseParser> initializer) {
        allowedIdentifierSymbols = EnumSet.copyOf(options.getAllowedIdentifierSymbols());
        ...
    }

    private class PostProcessor extends SqlBaseBaseListener {
        private final List<String> ruleNames;
        private final Consumer<ParsingWarning> warningConsumer;

        public PostProcessor(List<String> ruleNames, Consumer<ParsingWarning> warningConsumer) {
            this.ruleNames = ruleNames;
            this.warningConsumer = requireNonNull(warningConsumer, "warningConsumer is null");
        }

        @Override
        public void exitUnquotedIdentifier(SqlBaseParser.UnquotedIdentifierContext context) {
            String identifier = context.IDENTIFIER().getText();
            for (IdentifierSymbol identifierSymbol : EnumSet.complementOf(allowedIdentifierSymbols)) {
                char symbol = identifierSymbol.getSymbol();
                if (identifier.indexOf(symbol) >= 0) {
                    throw new ParsingException("identifiers must not contain '" + identifierSymbol.getSymbol() + "'", null, context.IDENTIFIER().getSymbol().getLine(), context.IDENTIFIER().getSymbol().getCharPositionInLine());
                }
            }
        }

        @Override
        public void exitBackQuotedIdentifier(SqlBaseParser.BackQuotedIdentifierContext context) {
            Token token = context.BACKQUOTED_IDENTIFIER().getSymbol();
            throw new ParsingException(
                    "backquoted identifiers are not supported; use double quotes to quote identifiers",
                    null,
                    token.getLine(),
                    token.getCharPositionInLine());
        }

        @Override
        public void exitDigitIdentifier(SqlBaseParser.DigitIdentifierContext context) {
            Token token = context.DIGIT_IDENTIFIER().getSymbol();
            throw new ParsingException(
                    "identifiers must not start with a digit; surround the identifier with double quotes",
                    null,
                    token.getLine(),
                    token.getCharPositionInLine());
        }

        @Override
        public void exitNonReserved(SqlBaseParser.NonReservedContext context) {
            // we can't modify the tree during rule enter/exit event handling unless we're dealing with a terminal.
            // Otherwise, ANTLR gets confused an fires spurious notifications.
            if (!(context.getChild(0) instanceof TerminalNode)) {
                int rule = ((ParserRuleContext) context.getChild(0)).getRuleIndex();
                throw new AssertionError("nonReserved can only contain tokens. Found nested rule: " + ruleNames.get(rule));
            }

            // replace nonReserved words with IDENT tokens
            context.getParent().removeLastChild();

            Token token = (Token) context.getChild(0).getPayload();
            if (RESERVED_WORDS_WARNING.contains(token.getText().toUpperCase())) {
                warningConsumer.accept(new ParsingWarning(
                        format("%s should be a reserved word, please use double quote (\"%s\"). This will be made a reserved word in future release.", token.getText(), token.getText()),
                        token.getLine(),
                        token.getCharPositionInLine()));
            }

            context.getParent().addChild(new CommonToken(
                    new Pair<>(token.getTokenSource(), token.getInputStream()),
                    SqlBaseLexer.IDENTIFIER,
                    token.getChannel(),
                    token.getStartIndex(),
                    token.getStopIndex()));
        }
    }
    ...
}
```
{% endspoiler %}

---

当然 还为词法分析器和语法分析器添加了ErrorHandler，也提供了选项让用户选择是否启用增强ErrorHandler

{% spoiler "SqlParser ErrorHandler 相关源码" %}
```java
public class SqlParser {
    private static final BaseErrorListener LEXER_ERROR_LISTENER = new BaseErrorListener() {
        @Override
        public void syntaxError(Recognizer<?, ?> recognizer, Object offendingSymbol, int line, int charPositionInLine, String message, RecognitionException e) {
            throw new ParsingException(message, e, line, charPositionInLine);
        }
    };
    
    private static final ErrorHandler PARSER_ERROR_HANDLER = ErrorHandler.builder()
            .specialRule(SqlBaseParser.RULE_expression, "<expression>")
            .specialRule(SqlBaseParser.RULE_booleanExpression, "<expression>")
            .specialRule(SqlBaseParser.RULE_valueExpression, "<expression>")
            .specialRule(SqlBaseParser.RULE_primaryExpression, "<expression>")
            .specialRule(SqlBaseParser.RULE_identifier, "<identifier>")
            .specialRule(SqlBaseParser.RULE_string, "<string>")
            .specialRule(SqlBaseParser.RULE_query, "<query>")
            .specialRule(SqlBaseParser.RULE_type, "<type>")
            .specialToken(SqlBaseLexer.INTEGER_VALUE, "<integer>")
            .ignoredRule(SqlBaseParser.RULE_nonReserved)
            .build();
    
    private boolean enhancedErrorHandlerEnabled;
    
    public SqlParser(SqlParserOptions options, BiConsumer<SqlBaseLexer, SqlBaseParser> initializer) {
        ...
        enhancedErrorHandlerEnabled = options.isEnhancedErrorHandlerEnabled();
    }
    
    private Node invokeParser(String name, String sql, Function<SqlBaseParser, ParserRuleContext> parseFunction, ParsingOptions parsingOptions) {
        ...
        // Override the default error strategy to not attempt inserting or deleting a token.
        // Otherwise, it messes up error reporting
        parser.setErrorHandler(new DefaultErrorStrategy() {
            @Override
            public Token recoverInline(Parser recognizer) throws RecognitionException{
                if (nextTokensContext == null) {
                    throw new InputMismatchException(recognizer);
                } else {
                    throw new InputMismatchException(recognizer, nextTokensState, nextTokensContext);
                }
            }
        });
        lexer.removeErrorListeners();
        lexer.addErrorListener(LEXER_ERROR_LISTENER);
        parser.removeErrorListeners();
        if (enhancedErrorHandlerEnabled) {
            parser.addErrorListener(PARSER_ERROR_HANDLER);
        } else {
            parser.addErrorListener(LEXER_ERROR_LISTENER);
        }
        ...
    }
    ...
}
```
{% endspoiler %}

---

设置 parser 的解析方法，首先尝试使用更快的SLL，如果失败采用LL

parseFunction 接受 parser，并返回语法树

关注到parseFunction的类型为Function<SqlBaseParser, ParserRuleContext> 

主要是指定了文法规则入口

{% spoiler "SqlParser parseFunction 相关源码" %}
```java
public class SqlParser{
    public Statement createStatement(String sql, ParsingOptions parsingOptions) {
        return (Statement) invokeParser("statement", sql, SqlBaseParser::singleStatement, parsingOptions);
    }

    public Expression createExpression(String expression, ParsingOptions parsingOptions) {
        return (Expression) invokeParser("expression", expression, SqlBaseParser::standaloneExpression, parsingOptions);
    }

    public Return createReturn(String routineBody, ParsingOptions parsingOptions) {
        return (Return) invokeParser("return", routineBody, SqlBaseParser::standaloneRoutineBody, parsingOptions);
    }

    ...
}
```
{% endspoiler %}

---

最后需要对于这颗语法树的所有元素进行某些处理，考虑到语法树是一个复杂的对象结构，Antlr允许使用[访问者模式](https://refactoringguru.cn/design-patterns/visitor)。

使用AstBuilder对象对这颗语法树进行访问

---

最后，当栈溢出(StackOverflowError)的时候, 抛出ParsingException。

---

{% spoiler "SqlParser invokeParser 源码" %}
```java
public class SqlParser{
    private Node invokeParser(String name, String sql, Function<SqlBaseParser, ParserRuleContext> parseFunction, ParsingOptions parsingOptions) {
        try {
            SqlBaseLexer lexer = new SqlBaseLexer(new CaseInsensitiveStream(CharStreams.fromString(sql)));
            CommonTokenStream tokenStream = new CommonTokenStream(lexer);
            SqlBaseParser parser = new SqlBaseParser(tokenStream);
            initializer.accept(lexer, parser);

            // Override the default error strategy to not attempt inserting or deleting a token.
            // Otherwise, it messes up error reporting
            parser.setErrorHandler(new DefaultErrorStrategy() {
                @Override
                public Token recoverInline(Parser recognizer) throws RecognitionException {
                    if (nextTokensContext == null) {
                        throw new InputMismatchException(recognizer);
                    } else {
                        throw new InputMismatchException(recognizer, nextTokensState, nextTokensContext);
                    }
                }
            });

            parser.addParseListener(new PostProcessor(Arrays.asList(parser.getRuleNames()), parsingOptions.getWarningConsumer()));

            lexer.removeErrorListeners();
            lexer.addErrorListener(LEXER_ERROR_LISTENER);

            parser.removeErrorListeners();

            if (enhancedErrorHandlerEnabled) {
                parser.addErrorListener(PARSER_ERROR_HANDLER);
            } else {
                parser.addErrorListener(LEXER_ERROR_LISTENER);
            }

            ParserRuleContext tree;
            try {
                // first, try parsing with potentially faster SLL mode
                parser.getInterpreter().setPredictionMode(PredictionMode.SLL);
                tree = parseFunction.apply(parser);
            } catch (ParseCancellationException ex) {
                // if we fail, parse with LL mode
                tokenStream.reset(); // rewind input stream
                parser.reset();

                parser.getInterpreter().setPredictionMode(PredictionMode.LL);
                tree = parseFunction.apply(parser);
            }

            return new AstBuilder(parsingOptions).visit(tree);
        } catch (StackOverflowError e) {
            throw new ParsingException(name + " is too large (stack overflow while parsing)");
        }
    }
}
```
{% endspoiler %}

