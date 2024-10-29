
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
    private resolveIdentExpr(expr: Expr, syms: Syms) {
        if (expr.type !== "ident")
            throw new Error("expected ident");
        const ident = expr.kind.ident;
        const { sym, ok: symFound } = syms.get(ident);
        if (!symFound) {
            this.reportUseOfUndefined(ident, expr.pos, syms);
            return;
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
    }
    // ...
}
```

When resolving an identifier, we essentially convert the identifier expression in-AST into a symbol expression. Therefore we need to take the expression, so we can mutate it. Because the `Expr` type is unspecific, we have to assert we've gotten an identifer. Then we try and find the symbol in the symbol table. And then we do the mutation, ie. converting the identifier expression into a symbol expression. We have to check that `sym.stmt` and `sym.param` are present, before we assign them.

## 7.6 Resolving expressions

```ts
class Resolver {
    // ...
    private resolveExpr(expr: Expr, syms: Syms) {
        if (expr.kind.type === "error") {
            return;
        }
        if (expr.kind.type === "ident") {
            this.resolveIdentexpr(expr, syms);
            return;
        }
        // ...
        if (expr.kind.type === "binary") {
            this.resolveExpr(expr.kind.left, syms);
            this.resolveExpr(expr.kind.right, syms);
            return;
        }
        // ...
        if (expr.kind.type === "block") {
            const childSyms = new Syms(syms);
            for (const stmt of expr.stmts) {
                this.resolveStmt(stmt, childSyms);
            }
            if (expr.expr) {
                this.resolveExpr(expr.expr, childSyms);
            }
            return;
        }
        // ...
        throw new Error(`unknown expression ${expr.kind.type}`);
    }
    // ...
}
```

We traverse the tree, meaning we call `resolveExpr` on each childnode recursively. This way, we reach all identifiers in the AST, that need to be resolved.

Binary expressions are resolved by resolving the left and right operands.

Block expressions are resolved by making a child symbol table, and resolving each statement and the expression if present.

### Exercises

1. Implement the rest of the expressions.

## 7.7 Resolving let statements

```ts
class Resolver {
    // ...
    private resolveLetStmt(stmt: Stmt, syms: Syms) {
        if (stmt.kind.type !== "let")
            throw new Error("expected let statement");
        this.resolveExpr(stmt.kind.value, syms);
        const ident = stmt.kind.param.ident;
        if (syms.locallyDefined(ident)) {
            this.reportAlreadyDefined(ident, stmt.pos, syms);
            return;
        }
        syms.define(ident, {
            ident,
            type: "let",
            pos: stmt.param.pos,
            stmt,
            param: stmt.param,
        });
    }
    // ...
}
```

To resolve a let statement, we resolve the value expression, then we check that the symbol has not been defined already, and then we define the symbol as a let-symbol.

## 7.8 Resolving function definition statements

```ts
class Resolver {
    // ...
    private resolveFnStmt(stmt: Stmt, syms: Syms) {
        if (stmt.kind.type !== "fn")
            throw new Error("expected fn statement");
        if (syms.locallyDefined(stmt.kind.ident)) {
            this.reportAlreadyDefined(stmt.kind.ident, stmt.pos, syms);
            return;
        }
        syms.define(ident, {
            ident: stmt.kind.ident,
            type: "fn",
            pos: stmt.pos,
            stmt,
        });
        const fnScopeSyms = new Syms(syms);
        for (const param of stmt.kind.params) {
            if (fnScopeSysm.locallyDefined(param.ident)) {
                this.reportAlreadyDefined(param.ident, param.pos, syms);
                continue;
            }
            fnScopeSyms.define(param.ident, {
                ident: param.ident,
                type: "fn_param",
                pos: param.pos,
                param,
            });
        }
        this.resolveExpr(stmt.kind.body, fnScopeSyms);
    }
    // ...
}
```

To resolve a function definition we first check that the symbol is not already defined, then we define it. Then we make a child symbol table, define all the parameters and lastly resolve the function body. 

Parameters must not have the same name, to that end we check that each parameters identififer is not already defined.

Contrary to resolving the let statement, we define the function symbol before resolving the body expression. This is so that the function body is able to call the function recursively.

## 7.9 Resolving statements

```ts
class Resolver {
    // ...
    private resolveStmt(stmt: Stmt, syms: Syms) {
        if (stmt.kind.type === "error") {
            return;
        }
        if (stmt.kind.type === "let") {
            this.resolveLetStmt(stmt, syms);
            return;
        }
        if (stmt.kind.type === "fn") {
            this.resolveFnStmt(stmt, syms);
            return;
        }
        if (stmt.kind.type === "return") {
            if (stmt.kind.expr)
                this.resolveExpr(stmt.kind.expr, syms);
            return;
        }
        // ...
        throw new Error(`unknown statement ${expr.kind.type}`);
    }
    // ...
}
```

Just like expressions, we traverse the AST and resolve every sub-statement and expression.

### Exercises

1. Implement the rest of the expressions.

## 7.10 Resolving AST

```ts
class Resolver {
    // ...
    private resolveStmts(stmts: Stmt[]) {
        const scopeSyms = new Syms(this.root);
        for (const stmt of stmts) {
            this.resolveStmt(stmt, scopeSyms);
        }
    }
    // ...
}
```

We'll make a child symbol table of the root and resolve each statement.

## 7.11 Errors

### 7.11.1 Use of undefined

```ts
class Resolver {
    // ...
    private reportUseOfUndefined(ident: Ident, pos: Pos, syms: Syms) {
        console.error(`use of undefined symbol '${ident}' at ${pos.line}${pos.col}`);
    }
    // ...
}
```

### 7.11.2 Already defined

```ts
class Resolver {
    // ...
    private reportAlreadyDefined(ident: Ident, pos: Pos, syms: Syms) {
        console.error(`symbol already defined '${ident}', at ${pos.line}${pos.col}`);
        const prev = syms.get(ident);
        if (!prev.ok)
            throw new Error("expected to be defined");
        if (!prev.sym.pos)
            return;
        const { line: prevLine, col: prevCol } = prev.sym.pos;
        console.error(`previous definition of '${ident}' at ${prevLine}:${prevPos}`);
    }
    // ...
}
```

Print the last definition to help the user.

## Exercises

1. \* Implement, so that `reportUseOfUndefined` searches for similar symbols, eg. using lLevenshtein distance.
