
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

```ts
class Parser {
    // ...
    private pos(): Pos { return this.current().pos; }
    // ...
}
```

The parser does not need to keep track of `index`, `line` and `col` as those are stored in the tokens.

Also like the lexer, we'll have a `.test()` method in the parser, which will test for token type rather than strings or regex.

```ts
class Parser {
    // ...
    private test(type: string): bool { return this.current().type === type; }
    // ...
}
```

## Operands

