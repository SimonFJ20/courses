
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

We don't just need to check that every symbol is defined, we also need to be able to get the definition of each symbol, whenever we need it. We'll do this, by replacing all identifiers, *identifying a symbol*, by the symbol itself.

To that end, we'll add symbol as an expression in the AST.

```ts
type ExprKind =
    // ...
    | {
        type: "sym",
        ident: string,
        defType: "let" | "fn" | "fn_param" | "builtin",
        stmt?: Stmt,
        param?: Param,
    }
    // ...
    ;
```

The estute reader will have recognized that the fields `stmt` and `param` may refer back to a related node in the structure, which implies that the structure is cyclic. We therefore no longer have a tree (directed aclyclic graph), but instead a graph. This will have no real consequences, only theoritical implications. I will still refer to the structure as the AST.

## 7.4 The resolver class

We'll make a resolver class, which, just like the evaluat in chapter 4, has a root symbol table.

```ts
class Resolver {
    private root = new Syms();
    // ...
}
```

## 7.5 Resolving identifiers

We'll start by defining a way to resolve identifiers.

```ts
class Resolver {
    // ...
    private resolveIdentExpr(expr: Expr, syms: Syms): { ok: boolean } {
        if (expr.type !== "ident")
            throw new Error("expected ident");
        const ident = expr.kind.ident;
        const { sym, ok: symFound } = syms.get(ident);
        if (!symFound) {
            this.reportUseOfUndefined(ident, expr.pos, syms);
            return { ok: false };
        }
        expr.kind = {
            type: "sym",
            ident,
            defType: sym.type,
        };
        if (sym.stmt)
            expr.kind.stmt = sym.stmt;
        if (sym.param)
            expr.kind.param = sym.param;
        return { ok: true };
    }
    // ...
}
```

When resolving an identifier, we essentially convert the identifier expression in-AST into a symbol expression. Therefore we need to take the expression, so we can mutate it. Because the `Expr` type is unspecific, we have to assert we've gotten an identifer.

Then we try and find the symbol in the symbol table, and if we don't, we report and error and return a non-ok result.

And then we do the mutation, ie. converting the identifier expression into a symbol expression. We have to check that `sym.stmt` and `sym.param` are present, before we assign them.



