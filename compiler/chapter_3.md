
# Parser

In this chaper I'll show how I would make a parser.

A parser, in addition to our lexer, transforms the input program as text, meaning an unstructured sequence of characters, into a structered representation. Structured meaning the representation tells us about the different constructs such as if statements and expressions.

## Abstract Syntax Tree AST

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

## Consumer of lexer

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
    private report(pos: Pos, msg: string) {
        console.log(`Parser: ${msg} at ${pos.line}:${pos.col}`);
    }
    // ...
}
```

## Operands

Operands are the individual parts of an operation. For example, in the math expression `a + b`, (would be `+ a b` in the input language), `a` and `b` are the *operands*, while `+` is the *operator*. In the expression `a + b * c`, the operands are `a`, `b` and `c`. But in the expression `a * (b + c)`, the operands of the multiply operation are `a` and `(b + c)`. `(b + c)` is an operands, because it is enclosed on both sides. This is how we'll define operands.

We'll make a public method in `Parser` called `parseOperand`.

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        const pos = this.pos();
        if (this.test("int")) {
            const value = this.current().intValue;
            this.step();
            return { kind: { type: "int", value }, pos };
        }
        this.report(pos "expected expr");
        this.step();
        return { kind: { type: "error" }, pos };
    }
    // ...
}
```

### Integer

Parsing an integer is a 1:1 translation between the integer token and an integer expression.

```ts
type ExprKind =
    // ...
    | { type: "int", value: number }
    // ...
    ;
```

```ts
class Parser {
    // ...
    public parseOperand(): Expr {
        // ...
        if (this.test("int")) {
            const value = this.current().intValue;
            this.step();
            return { kind: { type: "int", value }, pos };
        }
        // ...
    }
    // ...
}
```

