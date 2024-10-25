
# 7 Symbol resolver

We need a way to know the definition of each symbol used in to program.

## 7.1 Symbols

We'll start by defining the symbol type we need.

```ts
type Sym = {
    ident: string,
    type: "let" | "fn" | "fn_param" | "builtin",
    pos?: Pos,
    stmt?: Stmt,
    param?: Param,
}
```

The identifier and position are the same, as in chapter 4. But now, instead of knowing the value of a symbol, we instead need to know the definition of the symbol.

The 4 types are currently all the ways to introduce symbols. The defining statement and param is set according to the type.

## 7.2 Symbol maps

```ts
type SymMap = { [ident: string]: Sym }

class Syms {
    private syms: SymMap = {};

    public constructor(private parent?: Syms) {}

    public define(ident: string, sym: Sym) {
        this.syms[ident] = sym;
    }

    public definedLocally(ident: string): boolean {
        return ident in this.syms;
    }

    public get(ident: string): { ok: true, sym: Sym } | { ok: false } {
        if (ident in this.syms)
            return { ok: true, sym: this.syms[ident] };
        if (this.parent)
            return this.parent.get(ident);
        return { ok: false };
    }
}
```

We'll start out with the same `SymMap` type and `Syms` class as chapter 4.

## 7.3 Symbols in AST

We don't just need to check that every symbol is defined, we also need to be able to get the definition of each symbol, whenever we need it. In that sense we need to store the symbol maps in a convenient manner in relation to the usage of the symbols in the AST. We'll therefore store the symbol maps in the AST.

### 7.3.1 lock expression symbols

Each block expression introduces a new scope, meaning a new child symbol table. It's therefore natural to store the child symbol table alongside the expression.

We'll therefore add a field to the block expression.

```ts
type ExprKind =
    // ...
    | {
        type: "block",
        stmts: Stmt[],
        expr?: Expr,
        syms?: Syms,
    }
    // ...
    ;
```

The field is optional so that the parser don't have to set it, but we can set it later.

### 7.3.2 Function definitions symbols

Function definitions also include a symbol table. This is because the defined body expressions need the symbols defined at the function's definition time, as opposed to at time of calling.


```ts
type StmtKind =
    // ...
    | {
        type: "fn",
        ident: string,
        params: Param[],
        body: Expr,
        syms?: Syms,
    }
    // ...
    ;
```


