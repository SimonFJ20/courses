
# 3 Parser

In this chaper I'll show how I would make a parser.

A parser, in addition to our lexer, transforms the input program as text, meaning an unstructured sequence of characters, into a structered representation. Structured meaning the representation tells us about the different constructs such as if statements and expressions.

## 3.1 Abstract Syntax Tree AST

The result of parsing is a tree structure representing the input program.

This structure is a recursive acyclic structure storing the different parts of the program.

This is how I would define an AST data type.

```ts
type Stmt = {
    kind: StmtKind,
    pos: Pos,
};

type StmtKind =
    | { type: "error" }
    // ...
    | { type: "let", ident: string, value: Expr }
    // ...
    ;

type Expr = {
    kind: ExprKind,
    pos: Pos,
};

type ExprKind =
    | { type: "error" }
    // ...
    | { type: "int", value: number }
    // ...
    ;
```

Both `Stmt` (statement) and `Expr` (expression) are polymorphic types, meaning an expression, for example, can be either an addition operation containing 2 inner expressions or an integer expression containing the integer value, etc. This can also be implemented with classes and sub classes.

For both `Stmt` and `Expr` there's an error-kind. This makes the parser simpler, as we won't need to manage parsing failures differently than successful parslings.

## 3.2 Consumer of lexer

To start, we'll implement a `Parser` class, which for now is simply a consumer of a token iterater, meaning the lexer. In simple terms, whereas the lexer is a transformation from text to tokens, the parser is a transformation from token to an AST, except that the parser is not an iterator.

```ts
class Parser {
    private currentToken: Token | null;

    public constructor(private lexer: Lexer) {
        this.currentToken = lexer.next();
    }
    // ...
    private step() { this.currentToken = this.lexer.next() }
    private done(): bool { return this.currentToken == null; }
    private current(): Token { return this.currentToken!; }
    // ...
}
```

This implementation should look familiar compared to the lexer. We use the `currentToken` as a 'buffer', and then just use the `.next()` on the `lexer`.

Just as the lexer, we'll have a `.pos()` method, returning the current position.

For convenience, although there are other ways of doing it, we'll implement another public method on `Lexer`, which will return the lexer's current position.

```ts
class Lexer {
    // ...
    public currentPos(): Pos { return this.pos(); }
    // ...
}
```
The reason, is that when the lexer has reached the end of the file, the `.next()` method will return `null` instead of a token with a position, meaning we won't get the position after the last token.

```ts
class Parser {
    // ...
    private pos(): Pos {
        if (this.done())
            return this.lexer.currentPos();
        return this.current().pos;
    }
    // ...
}
```

The parser does not need to keep track of `index`, `line` and `col` as those are stored in the tokens. The token's position is prefered to the lexer's.

Also like the lexer, we'll have a `.test()` method in the parser, which will test for token type rather than strings or regex.

```ts
class Parser {
    // ...
    private test(type: string): bool {
        return !this.done() && this.current().type === type;
    }
    // ...
}
```

When testing, we first check that we have not reach the end. Either we have to do that here, or the caller will have to write something like `!this.done() && this.test(...)`, and it's easy to do it here.

We'll also want a method for reporting errors.

```ts
class Parser {
    // ...
    private report(msg: string, pos = this.pos()) {
        console.log(`Parser: ${msg} at ${pos.line}:${pos.col}`);
    }
    // ...
}
```

## 3.3 Operands

Operands are the individual parts of an operation. For example, in the math expression `a + b`, (would be `+ a b` in the input language), `a` and `b` are the *operands*, while `+` is the *operator*. In the expression `a + b * c`, the operands are `a`, `b` and `c`. But in the expression `a * (b + c)`, the operands of the multiply operation are `a` and `(b + c)`. `(b + c)` is an operands, because it is enclosed on both sides. This is how we'll define operands.

We'll make a public method in `Parser` called `parseOperand`.

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        const pos = this.pos();
        // ...
        this.report("expected expr", pos);
        this.step();
        return { kind: { type: "error" }, pos };
    }
    // ...
}
```

### 3.3.1 Identifiers and literals

Identifiers and literals (integers, strings) are single token constructs, meaning the parsing consists of translating a token into an ast-node with the value.

```ts
type ExprKind =
    // ...
    | { type: "ident", value: string }
    | { type: "int", value: number }
    | { type: "string", value: string }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        // ...
        if (this.test("ident")) {
            const value = this.current().identValue;
            this.step();
            return { kind: { type: "ident", value }, pos };
        }
        if (this.test("int")) {
            const value = this.current().intValue;
            this.step();
            return { kind: { type: "int", value }, pos };
        }
        if (this.test("string")) {
            const value = this.current().stringValue;
            this.step();
            return { kind: { type: "string", value }, pos };
        }
        // ...
    }
    // ...
}
```

### 3.3.2 Group expressions

A group expression is an expression enclosed in parenthesis, eg `(1 + 2)`. Because the expression is enclosed, meaning starts with a `(`-token and ends with a `)`-token, we will treat is like an operand.

```ts
type ExprKind =
    // ...
    | { type: "group", expr: Expr }
    // ...
    ;
```

If we find a `(`-token in `.parseOperand()`, we know that we should parse a group expression. We do this by ignoring the `(`-token, parsing an expression using `.parseExpr()` and checking that we find a `)`-token afterwards.

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        // ...
        if (this.test("(")) {
            this.step();
            const expr = this.parseExpr();
            if (!this.test(")")) {
                this.report("expected ')'");
                return { kind: { type: "error" }, pos };
            }
            this.step();
            return { kind: { type: "group", expr }, pos };
        }
        // ...
    }
    // ...
}
```

If we do not find the closing `)`-token, we report an error and return an error expression.

### 3.3.3 Block, if and loop operands

We want to be able to use blocks, if and loop constructs as expressions.

Example:
```rs
let temperature_feeling = if > temperature 20 { "hot" } else { "cold" };
```

Each construct will have their own `.parse...()`-method, so we'll just look for the first `{`-, `if`-, or `loop`-token and call the relevant method.

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        // ...
        if (this.test("{"))
            return this.parseBlock();
        if (this.test("if"))
            return this.parseIf();
        if (this.test("loop"))
            return this.parseLoop();
        // ...
    }
    // ...
}
```

## 3.4 Postfix operators

Postfix operations are expressions were the operators come after the subject expression. This includes field expressions (eg. `subject.field`), index expressions (eg. `subject[index]`) and call expressions (eg. `subject(...args)`).

A notable detail, is that postfix operations are chainable, eg. `subject[index].field` is valid, likewise with `subject.method(arg)` and `matrix[y][x]`.

We'll make a method `.parsePostfix()` to parse postfix operators.

```ts
class Parser {
    // ...
    public parsePostfix(): Expr {
        let subject = this.parseOperand();
        while (true) {
            const pos = this.pos();
            // ...
            break;
        }
        return subject;
    }
    // ...
}
```

We start by parsing an operand. Then we enter a loop, which runs until we no longer find any relevant operator tokens. When we parse a postfix expression, the `subject` will be replaced with the new parsed expression.

Notice we don't define `pos` at the start, but after we've parsed the subject. That's because we want `pos` to the reflect the start of the postfix operator, not the start of the subject.

### 3.4.1 Field expression

A field expression is for accessing fields on an object, and consists of a `.`-token and an identifier, eg. `.field`.

```ts
type ExprKind =
    // ...
    | { type: "field", subject: Expr, value: string }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parsePostfix(): Expr {
        // ...
        while (true) {
            // ...
            if (this.test(".")) {
                this.step();
                if (!this.test("ident")) {
                    this.report("expected ident");
                    return { kind: { type: "error" }, pos };
                }
                const value = this.current().identValue;
                this.step();
                subject = { kind: { type: "field", subject, value }, pos };
                continue;
            }
            // ...
        }
        // ...
    }
    // ...
}
```

If we find a `.`-token, we step over it, and make sure that we've hit an identifier. We save the identifier value and step over the identifier. Then we replace `subject` with a new field expression containing the previous `subject` value. Then we continue to look for the next postfix operator.

### 3.4.2 Index expression

An index operation consists of the subject and an index. The index is an expression, and it is contained in `[`- and `]`-tokens, eg. `subject[value]`.


```ts
type ExprKind =
    // ...
    | { type: "index", subject: Expr, value: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parsePostfix(): Expr {
        // ...
        while (true) {
            // ...
            if (this.test("[")) {
                this.step();
                const value = this.parseExpr();
                if (!this.test("]") {
                    this.report("expected ']'");
                    return { kind: { type: "error" }, pos };
                }
                this.step();
                subject = { kind: { type: "index", subject, value }, pos };
                continue;
            }
            // ...
        }
        // ...
    }
    // ...
}
```

If we find a `[`-token, we parse the index part exactly the same way, we parse a group expression.

### 3.4.3 Call expression

A call expression is like an index expression, except that it uses `(` and `)` instead of `[` and `]` and that there can be 0 or more expressions (arguments or args) inside the `(` and `)`. The arguments are seperated by `,`.

```ts
type ExprKind =
    // ...
    | { type: "call", subject: Expr, args: Expr[] }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parsePostfix(): Expr {
        // ...
        while (true) {
            // ...
            if (this.test("(")) {
                this.step();
                let args: Expr[] = [];
                if (!this.test(")") {
                    args.push(this.parseExpr());
                    while (this.test(",")) {
                        this.step();
                        if (this.test(")"))
                            break;
                        args.push(this.parseExpr());
                    }
                }
                const value = this.parseExpr();
                if (!this.test(")") {
                    this.report("expected ')'");
                    return { kind: { type: "error" }, pos };
                }
                this.step();
                subject = { kind: { type: "call", subject, args }, pos };
                continue;
            }
            // ...
        }
        // ...
    }
    // ...
}
```

Similarly to index epxressions, if we find a `(`-token, we step over it, parse the arguments, check for a `)` and replace `subject` with a call expression containing the previous `subject`.

When parsing the arguments, we start by testing if we've reached a `)` to check if there are any arguments. If not, we parse the first argument. 
The consecutive arguments are all preceded by a `,`-token. There we test or `,`, to check if we should keep parsing arguments. After checking for a seperating `,`, we check if we've reached a `)` and break if so. This is to allow for trailing comma.

```ts
func(
    a,
    b, // trailing comma
)
```

PS. it's called arguments in a function call. We call with arguments, and define with parameters.

## 3.5 Prefix expressions

Contrasting postfix expressions, prefix expression are operations where the operator comes first, then the operands are listed. In some languages, operations such as negation (eg. `-value`) and not-operations (eg. `!value`) are prefix operations. In the language we're making, all binary and unary arithmetic operations are prefix. This includes both expressions with a single operand, such as not (eg. `not value`), but also expressions with 2 operands, such ass addition (eg. `+ a b`) and equation (eg. `== a b`).

This is because infix operators (eg. `a + b`) makes parsing more complicated, as it requires reasoning about operator precedence, eg. why `2 + 3 * 4 != (2 + 3) * 4`.

Operations with 1 operand are called unary expression. Operations with 2 are called binary expressions.

```ts
type ExprKind =
    // ...
    | { type: "unary", unaryType: UnaryType, subject: Expr }
    | { type: "binary", binaryType: BinaryType, left: Expr, right: Expr }
    // ...
    ;

type UnaryType = "not" /*...*/;
type BinaryType = "+" | "*" | "==" /*...*/;
```

```ts
class Parser {
    // ...
    public parsePrefix(): Expr {
        const pos = this.pos();
        // ...
        return this.parsePostfix();
    }
    // ...
}
```

We again get the position immediately, because the operation, eg. `+ a b`, starts at the first `+`-token.

If we don't find any operators, we proceed to try to parse a postfix expression.

### 3.5.1 Unary expressions

```ts
class Parser {
    // ...
    public parsePrefix(): Expr {
        // ...
        if (this.test("not")) {
            this.step();
            const subject = this.parsePrefix();
            return { kind: { type: "unary", unaryType: "not", subject }, pos };
        }
        // ...
    }
    // ...
}
```

If we find a `not`-token, we ignore it, parse a prefix expression recursively, and return a unary expression with the `subject` and unary type.

### 3.5.2 Binary expressions

```ts
class Parser {
    // ...
    public parsePrefix(): Expr {
        // ...
        if (this.test("+")) {
            this.step();
            const left = this.parsePrefix();
            const right = this.parsePrefix();
            return { kind: { type: "binary", binaryType: "+", left, right }, pos };
        }
        // ...
    }
    // ...
}
```

Just as with unary, if we find a `+`-token, we ignore it and parse prefix expression recursively. Then we parse the second operand, by parsing another prefix expressions. And then we return a binary expression with the `left` and `right` operands and the binary type.

## 3.6 Expressions

Lastly for expressions, we'll make a method `.parseExpr()` for parsing an expression.

```ts
class Parser {
    // ...
    public parseExpr(): Expr {
        return this.parsePrefix();
    }
    // ...
}
```

The method just proceeds to try and parse a prefix expression.

## 3.7 If expressions

An if-expression is an `if`-token, followed by an condition in the form of an expressions, followed by a block-expression, optionally followed by an `else`-token and another block.

```ts
type ExprKind =
    // ...
    | { type: "if", cond: Expr, truthy: Expr, falsy?: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseIf(): Expr {
        const pos = this.pos();
        this.step();
        const cond = this.parseExpr();
        if (!this.test("{")) {
            this.report("expected '}'");
            return { kind: { type: "error" }, pos };
        }
        const truthy = this.parseBlock();
        if (!this.test("else")) {
            return { kind: { type: "if", cond, truthy }, pos };
        }
        this.step();
        if (!this.test("{")) {
            this.report("expected '}'");
            return { kind: { type: "error" }, pos };
        }
        const falsy = this.parseBlock();
        return { kind: { type: "if", cond, truthy, falsy }, pos };
    }
    // ...
}
```

When parsing an if-expression, we assume we already have reached an `if`-token.

We skip the `if`-token. Then we parse the condition expression `cond`. Then we check for a `{`-token and parse block. Then we check for an `else`-token. If not present, we return an if-expression with no `falsy`-option. Else we skip the `else`-token, check for and parse the `falsy`-block, and return the if-expression with the `falsy`-option.

## 3.8 Loop expressions

A loop expression is a `loop`-token followed by a block expression.

```ts
type ExprKind =
    // ...
    | { type: "loop", body: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseLoop(): Expr {
        const pos = this.pos();
        this.step();
        if (!this.test("{")) {
            this.report("expected '}'");
            return { kind: { type: "error" }, pos };
        }
        const body = this.parseExpr();
        return { kind: { type: "loop", body }, pos };
    }
    // ...
}
```

We again assume, we've already hit a `loop`-token, which we step over. Then we check for and parse a block expression constituting the loop `body`. We then return the loop expression.

## 3.9 Break statements

A break statement consists of a `break`-token and an optional expression.

The language will allow us to use loop as an expression. The break statement will then deliver the resulting value, eg.
```rs
let value = loop {
    let value = get_value();
    if acceptable(value) {
        break value;
    }
};
```

```ts
type StmtKind =
    // ...
    | { type: "break", expr?: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseBreak(): Stmt {
        const pos = this.pos();
        this.step();
        if (!this.test(";")) {
            return { kind: { type: "break" }, pos };
        }
        const expr = this.parseExpr();
        return { kind: { type: "break", expr }, pos };
    }
    // ...
}
```

We assume we've already hit a `break`-token and step over it. Then we check if we've hit a `;`-token. If so, there's no expression, so we return a break statement with no expression. If there's no `;`-token, we parse an expression and return a break statement with it.

## 3.9 Return statements

The return statement is for functions what break is for loops.

```ts
type StmtKind =
    // ...
    | { type: "return", expr?: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseReturn(): Stmt {
        const pos = this.pos();
        this.step();
        if (!this.test(";")) {
            return { kind: { type: "return" }, pos };
        }
        const expr = this.parseExpr();
        return { kind: { type: "return", expr }, pos };
    }
    // ...
}
```

## 3.10 Let statements

A let statement declares a variable. A let statement consists of a `let`-token, an identifier, a `=`-token, and an expression. The expression is the initial value of the variable.

```ts
type StmtKind =
    // ...
    | { type: "let", ident: string, value: Expr }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseLet(): Stmt {
        const pos = this.pos();
        this.step();
        if (!this.test("ident")) {
            this.report("expected ident");
            return { kind: { type: "error" }, pos };
        }
        const ident = this.current().identValue;
        this.step();
        if (!this.test("=")) {
            this.report("expected '='");
            return { kind: { type: "error" }, pos };
        }
        this.step();
        const value = this.parseExpr();
        return { kind: { type: "let", ident, value }, pos };
    }
    // ...
}
```

We step over the first `let`-token. Then we check for an `ident`-token, save it's value and step over it. Then we check for and step over a `=`-token. We then parse an expressions. And lastly return a let statement with the `ident` and `valie`.

## 3.11 Function definition statement

A function definition statement or 'fn'-statement for short is a statement that defines a function with it's name, parameters and body.

The function name is an identifier. The body is a block expression. The parameters is a list of identifiers seperated by `,`, enclosed in `(` and `)`.

An fn statements consists of an `fn`-token, an it's name as an identifier token, a parameter list, and the body.

```ts
type StmtKind =
    // ...
    | { type: "fn", ident: string, params: Param[], body: Expr }
    // ...
    ;

type Param = {
    ident: string,
    pos: Pos,
};
```

```ts
class Parser {
    // ...
    public parseFn(): Stmt {
        const pos = this.pos();
        this.step();
        if (!this.test("ident")) {
            this.report("expected ident");
            return { kind: { type: "error" }, pos };
        }
        const ident = this.current().identValue;
    }
    // ...
}
```

PS. it's called parameters in a function definition. We call with arguments, and define with parameters.

