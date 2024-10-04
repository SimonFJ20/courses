
# 4 AST Evaluator

In this chapter, I'll show how you could implement an AST evaluator.

AST evaluation is the process of taking the parsed source in AST form and computing the values expression, ie. running the code. AST evaluation particularly, is a conceptually way of understanding execution of the code. The AST evaluator simply walks through the tree structure and evaluates the resulting value of each node.

## 4.1 Values

Values are what the expressions of the program will evaluate to. We'll define values as a variant type, just like `StmtKind` and `ExprKind`.

We'll start by defining the simple values, those being integers, strings, boolean values, null values and a special error value.

```ts
type Value =
    | { type: "error" }
    | { type: "null" }
    | { type: "int", value: number }
    | { type: "string", value: string }
    | { type: "bool", value: boolean }
    // ...
```

We'll also define a built in array type. (aka. list, vector).

```ts
type Value =
    // ...
    | { type: "array", values: Value[] }
    // ...
```

An array is implemented as a Typescript array of values.

We'll also define a struct type. (aka. object, table, dictionary, map).

```ts
type Value =
    // ...
    | { type: "struct", fields: { [key: string]: Value } }
    // ...
```

A struct is defined as key value map, where the key is a string. The struct fields will be accessible via. both field expressions, eg. `my_struct.field`,  and index expressions, eg. `my_struct["field"]`. We'll also want to be able to index with integers, in that case, we have to convert the integer to a string before indexing.

Then we'll need a value for function definitions. This will simply consist of the node id of the function definition.

```ts
type Value =
    // ...
    | { type: "fn", fnDefId: number }
    // ...
```

Lastly we'll define a type for builtin functions.

```ts
type Value =
    // ...
    | { type: "builtin_fn", name: string }
    // ...
```

A builtin function will have a name, which the evaluator will understand and treat accordingly.

### 4.2.1 Stringification

We'll need a way to represent values as text in strings.

```ts
function valueToString(value: Value): string {
    if (value.type === "error") {
        return "<error>";
    } else if (value.type === "null") {
        return "null";
    } else if (value.type === "int") {
        return value.value.toString();
    } else if (value.type === "string") {
        return `"${value.value}"`;
    } else if (value.type === "bool") {
        return value.value ? "true" : "false";
    } else if (value.type === "array") {
        const valueStrings = result.values
            .map(value => value.toString());
        return `[${valueStrings.join(", ")}]`;
    } else if (value.type === "struct") {
        const fieldStrings = Object.entries(result.fields)
            .map(([key, value]) => `${key}: ${valueToString(value)}`);
        return `struct { ${fieldStrings.join(", ")} }`;
    } else if (value.type === "fn") {
        return `<fn: ${value.fnDefId}>`;
    } else if (value.type === "builtin_fn") {
        return `<builtin_fn: ${value.name}>`;
    } else {
        throw new Error("unexhaustive");
    }
}
```

The `valueToString` function takes a value (variable of type `Value`) and checks its type. For each type of value, it returns a string representing that value. For error and null we return a static string of `"<error>"` and `"null"` respectably. For the others, we return a string also representing the *value's value*, eg. for int, we return the int value as a string. For array and struct, we call `valueToString` recursively on the contained values.

## 4.2 Symbols

An identifier is just a name. A symbol is a definition with an associated identifier. When evaluating the code, we have to keep track of symbols, both their definitions and their usage.

### 4.2.1 Scopes

Symbols are dependent on which scope they're in, eg. a symbol `a` defined outside of a pair of `{` `}` will be visible to code inside the braces, but a symbol `b` defined inside the braces will not be visible outside. Eg.

```rs
let a = 5;
{
    let b = 4;
    a; // ok
}
b; // b is not defined
```

Symbols are introduced in such statements as let and function defitions, the latter where both the function identifier and the parameters' identifiers will be introduces as symbols. Symbols may also be defined pre evaluation, which is the case for builtin functions such as `println` and `array`.

## 4.2.2 Symbol maps

To keep track of symbols throughout evaluation, we'll create a data structure to store symbols, ie. map identifiers to their definition values.

```ts
type SymMap = { [ident: string]: Value }

class Syms {
    private syms: SymMap = {};

    public constructor(private parent?: SymMap) {}
    // ...
}
```

The `SymMap` type is a key value map, which maps identifiers to their definition. To keep track of symbols in regard to scopes, we also define a `Syms` class. An instance of `Syms` is a node in a tree structure.

We'll define a method for defining symbols.

```ts
class Syms {
    // ...
    public define(ident: string, value: Value) {
        this.syms[ident] = value;
    }
    // ...
}
```

Then a method for checking, if a symbol is defined in the current scope.

```ts
class Syms {
    // ...
    public definedLocally(ident: string): boolean {
        return ident in this.syms;
    }
    // ...
}
```

And then, we'll define a method for getting the value of a defined symbol.

```ts
class Syms {
    // ...
    public get(ident: string): { ok: true, value: Value } | { ok: false } {
        if (ident in this.syms)
            return { ok: true, value: this.syms[ident] };
        if (this.parent)
            return this.parent.get(ident);
        return { ok: false };
    }
}
```

If the symbol is defined locally, return the value. Else if a the parent node is defined, defer to the parent. Otherwise, return a not-found result.

## 4.3 Control flow

Most code will run with unbroken control flow, but some code will 'break' control flow. This is the case for return statements in functions and break statements in loops. To keep track of, if a return or break statement has been run, we'll define a data structure representing the control flow action of evaluted code.

```ts
type Flow = {
    type: "value" | "return" | "break",
    value: Value,
}
```

The 3 implemented options for control flow is breaking in a loop, returning in a function and the non-breaking flow. All 3 options have an associated value.

### 4.3.1 Flow and value constructors

For ease of use, we'll add some functions to create the commonly used flow types and values.

```ts
function flowWalue(value: Value): Flow {
    return { type: "value", value };
}

function nullValue(): Value {
    return { type: "null" };
}
```

## 4.4 The evaluator class

To run/evaluate the code, we'll make an evaluator class.

```ts
class Evaluator {
    // ...
}
```

### 4.4.1 Root symbol table

We'll want a *root* symbol table, which stores all the predefined symbols. We also want a function for defining predefined symbols, ie. builtins.

```ts
class Evaluator {
    private root = new Syms();
    // ...
    public defineBuiltins() { /*...*/ }
    // ...
}
```

The `defineBuiltins` function will be defined later.

### 4.5 Expressions

Let's make a function `evalExpr` for evaluating expressions.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr): Flow {
        if (expr.type === "error") {
            throw new Error("error in AST");
        // ...
        } else {
            throw new Error(`unknown expr type "${expr.type}"`);
        }
    }
    // ...
}
```

The `evalExpr` function will take an expression and a symbol table, match the type of the expression and return a flow. If the expression is an error, meaning an error in the AST, the evaluator throws an error. In case the expression type is unknown, an error is thrown with the error type in the message.

#### 4.5.1 Identifiers

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        if (expr.type === "error") {
        // ...
        } else if (expr.type === "ident") {
            const result = syms.get(expr.value);
            if (!result.ok)
                throw new Error(`undefined symbol "${expr.value}"`);
            return this.value(result.value);
        } else {
        // ...
        }
    }
    // ...
}
```

#### 4.5.2 Literal expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        if (expr.type === "error") {
        // ...
        } else if (expr.type === "null") {
            return this.value(this.nullValue);
        } else if (expr.type === "int") {
        } else if (expr.type === "string") {
        } else if (expr.type === "bool") {
        } else {
        // ...
        }
    }
    // ...
}
```

```ts
class Evaluator {
    private root = new Syms();

    public evalStmts(stmts: Stmt[]): Flow {
        // ...
    }

    public evalStmt(stmt: Stmt): Flow {
        // ...
    }

    public evalExpr(expr: Expr): Flow {
        // ...
    }

    private executeBuiltin(name: string, args: Value[]): Flow {
        if (name === "array") {
            return this.value({ type: "array", values: [] });
        } else if (name === "struct") {
            return this.value({ type: "struct", fields: {} });
        } else if (name === "push") {
            if (args.length !== 2)
                throw new Error("incorrect arguments");
            const array = args[0];
            const value = args[1];
            if (array.type !== "array")
                throw new Error("incorrect arguments");
            array.values.push(value);
            return this.value(this.nullValue);
        } else if (name === "println") {
            if (args.length < 1)
                throw new Error("incorrect arguments");
            let msg = args[0];
            for (const arg of args.slice(1)) {
                if (!msg.includes("{}"))
                    throw new Error("incorrect arguments");
                msg.replace("{}", valueToString(arg));
            }
            console.log(msg);
            return this.value(this.nullValue);
        } else {
            throw new Error(`unknown builtin "${name}"`);
        }
    }

    public defineBuiltins() {
        this.root.define("array", { type: "builtin_fn", name: "array" });
        this.root.define("struct", { type: "builtin_fn", name: "struct" });
        this.root.define("push", { type: "builtin_fn", name: "struct" });
        this.root.define("println", { type: "builtin_fn", name: "println" });
    }

    private value(value: Value): Flow { return { type: "value", value }; }


    private readonly errorValue: Value = { type: "null" };
    private readonly nullValue: Value = { type: "error" };
}
```

