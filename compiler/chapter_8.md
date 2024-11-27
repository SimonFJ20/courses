
# 8 Types

We'd like to be able to compile to a relatively low level. To this end, we need to have predetermined the types of the values used in the program code.

We'll use 2 different concepts of types. One is parsed types. There are the ones the programmer writes explicitly. These we'll callexplicit types or `EType`. The other is calcaluted types, ie. result of type checking. These types we'll call value types or `VType`, because they tell us, within the compiler, the types of values.

## 8.1 Explicit types

For a type checker to be able to determine all the types of all values, the program is required to contain the information need. That information will be dictated by the programmer, therefore the programmer needs a way to write types explicitly. We'll do this by adding types to the language.

### 8.1.1 Syntax

We'll need explicit typing for the following types: null, int, string, bool, array, struct and function.

We want to be able to specify types in let-statements and fn-statements, like the following.

```rs
let a: int = 5;

fn add(a: int, b: int) -> int { /*...*/ }
```

For array and function types, we also want to specify details like contained type, return type and parameter types, like the following.

```rs
let op: fn(int, int) -> int = add;

let values: [int] = array();
```

### 8.1.2 Parsed explicit types

To parse explicit types, we'll need to add them to the AST definitions.

```ts
type EType = {
    kind: ETypeKind,
    pos: Pos,
    id: number,
}

type ETypeKind =
    | { type: "error" }
    | { type: "ident", value: string }
    | { type: "array", inner: EType }
    ;
```

```ts
class Parser {
    // ...
    private etype(kind: ETypeKind, pos: Pos): EType {
        const id = this.nextNodeId;
        this.nextNodeId += 1;
        return { kind, pos, id };
    }
    // ...
}
```

Just like expressions and statements, explicit types consists of multiple forms. We also want ids, like on statements and expressions.

```ts
class Parser {
    // ...
    public parseEType(): EType {
        const pos = this.pos();
        if (this.test("ident")) {
            const ident = this.current().identValue!;
            return this.etype({ type: "ident", value: ident });
        }
        if (this.test("[")) {
            this.step();
            const inner = this.parseEType();
            if (!this.test("]")) {
                this.report("expected ']'", pos);
                return this.etype({ type: "error" }, pos);
            }
            this.step();
            return this.etype({ type: "array", inner });
        }
        // ...
        this.report("expected type");
        return this.etype({ type: "error" }, pos);
    }
    // ...
}
```

We currently have implemented 2 types of explicit types: identifier types and array types. Identifier types are just like identifier expressions, ie. an identifier token. An array type is a type enclosed in `[` and `]`, eg. `[int]`.

### 8.1.3 Parsing parameters

Both function definitions and let-statements use parameters.

We'll therefore add types to parameters.

```ts
type Param = {
    ident: string,
    etype?: EType,
    pos: Pos,
};
```

We'll add the `etype` field. The field is optional, because we'll do type inference.

```ts
class Parser {
    // ...
    public parseParam(): { ok: true, value: Param } | { ok: false } {
        const pos = this.pos();
        if (this.test("ident")) {
            const ident = this.current().identValue!;
            this.step();
            if (this.test(":")) {
                const etype = this.parseEType();
                return { ok: true, value: { ident, etype, pos } };
            }
            return { ok: true, value: { ident, pos } };
        }
        this.report("expected param");
        return { ok: false };
    }
    // ...
}
```

We've added the if-statement checking for a `:`-token.

### 8.1.4 Parsing functions

```ts
type StmtKind =
    // ...
    | {
        type: "fn",
        ident: string,
        params: Param[],
        returnType?: EType,
        body: Expr,
    }
    // ...
    ;
```

We've added the `returnType` field. It is optional, because the return type my be omitted, in which case, it will be assigned null as the return type.

```ts
class Parser {
    // ...
    public parseFn(): Stmt {
        const pos = this.pos();
        this.step();
        if (!this.test("ident")) {
            this.report("expected ident");
            return this.stmt({ type: "error" }, pos);
        }
        const ident = this.current().identValue!;
        this.step();
        if (!this.test("(")) {
            this.report("expected '('");
            return this.stmt({ type: "error" }, pos);
        }
        const params = this.parseFnParams();
        let returnType: EType | null = null;
        if (this.test("->")) {
            this.step();
            returnType = this.parseEType();
        }
        if (!this.test("{")) {
            this.report("expected block");
            return this.stmt({ type: "error" }, pos);
        }
        const body = this.parseBlock();
        if (returnType === null) {
            return this.stmt({ type: "fn", ident, params, body }, pos);
        }
        return this.stmt({ type: "fn", ident, params, returnType, body }, pos);
    }
    // ...
}
```

We've added the if statement checking for `->`, and the handling of the mutable `returnType` variable.

## 8.2 Value types

The type checking process takes the program with some amount of explicit types. It then figures out the types of all values and operations. In this process, it will also check that all values and operations operate on and have themselves valid and appropriate value types. The type checker works with and produces value types or `VType`.

```ts
type VType =
    | { type: "error" }
    | { type: "unknown" }
    | { type: "null" }
    | { type: "int" }
    | { type: "string" }
    | { type: "bool" }
    | { type: "array", inner: VType }
    | { type: "struct" }
    | { type: "fn", params: VType[], returnType: VType }
```

The error-type is for convenience of implementation, just like in the parser. The unknown-type is for type inference, when we don't yet know the type. the null-, int-, string-, bool- and struct-types are uniform in that all values of each type can be assigned to each type. There's no difference between to integers, although the values might differ, and the same for two struct values, even though they may contain a different set of fields. Array-types and function-types also contain their details, ie. inner type in arrays and params and return type in functions.

### 8.2.1 Type equality

We need a way, of checking the equality of two types. We'll do this, by creating a `vtypesEqual` function.

```ts
function vtypesEqual(a: VType, b: VType): boolean {
    if (a.type !== b.type)
        return false;
    if (["error", "unknown", "null", "int", "string", "bool", "struct"].includes(a.type))
        return true;
    if (a.type === "array")
        return valueTypesEqual(a.inner, b.inner);
    if (a.type === "fn") {
        if (a.params.length !== b.params.length)
            return false;
        for (let i = 0; i < a.params.length; ++i) {
            if (!vtypesEqual(a.params[i], b.params[i])) {
                return false;
            }
        }
        return vtypesEqual(a.returnType, b.returnType);
    }
    return false;
}
```

For all types it applies, that the (kind) types must be equal. *Simple* types, such as `"int"` and `"bool"`, need just to be equal in (kind) type. For *complex* types, such as `"array"`, we also need to check the (value) type equality of the sub-types.

### 8.2.2 Value type string representation

We'll want a way to represent value types as strings, for use, for example, in error messages.

```ts
function vtypeToString(vtype: VType): string {
    if (["error", "unknown", "null", "int", "string", "bool", "struct"].includes(vtype.type))
        return vtype.type;
    if (a.type === "array")
        return `[${vtypeToString(vtype.inner)}]`;
    if (a.type === "fn") {
        const paramString = vtype.params.map(param => vtypeToString(param)).join(", ");
        return `fn (${paramString}) -> ${vtypeToString(vtype.returnType)}`;
    }
    throw new Error(`unhandled vtype '${vtype.type}'`);
}
```

### 8.2.3 Value types in AST

When checking the types, we need to store the resulting value types. We'll choose to store these inside the AST itself. This means we'll mutate the AST in the type checker.

```ts
type Expr = {
    // ...
    vtype?: ValueType,
}
```
```ts
type Stmt = {
    // ...
    vtype?: ValueType,
}
```
```ts
type Param = {
    // ...
    vtype?: ValueType,
}
```

## 8.3 The checker class

```ts
class Checker {
    // ...
}
```

The checker class will by similar to the resolver class, from an architectural perspective.

### 8.3.1 Reporting errors

```ts
class Checker {
    // ...
    public report(msg: string, pos: Pos) {
        console.error(`${msg} at ${pos.line}:${pos.col}`);
    }
    // ...
}
```

## 8.4 Checking explicit types

```ts
class Checker {
    // ...
    public checkEType(etype: EType): VType {
        const pos = etype.pos;
        // ...
        throw new Error(`unknown explicit type ${etype.type}`);
    }
    // ...
}
```

### 8.4.1 Checking explicit identifier types

```ts
class Checker {
    // ...
    public checkEType(etype: EType): VType {
        // ...
        if (etype.type === "ident") {
            if (etype.value === "null")
                return { type: "null" };
            if (etype.value === "int")
                return { type: "int" };
            if (etype.value === "bool")
                return { type: "bool" };
            if (etype.value === "string")
                return { type: "string" };
            if (etype.value === "struct")
                return { type: "struct" };
            this.report(`undefined type '${etype.value}'`);
            return { type: "error" };
        }
        // ...
    }
    // ...
}
```

We test if the identifier value matches the known types.

### 8.4.2 Checking explicit array types

```ts
class Checker {
    // ...
    public checkEType(etype: EType): VType {
        // ...
        if (etype.type === "array") {
            const inner = this.checkEType(etype.inner);
            return { type: "array", inner };
        }
        // ...
    }
    // ...
}
```

We check the inner type recursively, then return an array value type.

## 8.5 Checking expressions

```ts
class Checker {
    // ...
    public checkExpr(expr: Expr): VType {
        const pos = expr.pos;
        if (expr.kind.type === "error") {
            throw new Error("error in AST");
        }
        // ...
        throw new Error(`unknown expression ${expr.type}`);
    }
    // ...
}
```

The `checkExpr` method takes an AST expression and returns a value type.

### 8.5.1 Checking symbol expressions

```ts
class Checker {
    // ...
    public checkExpr(expr: Expr): VType {
        // ...
        if (expr.kind.type === "sym") {
            return this.checkSymExpr(expr);
        }
        // ...
    }
    // ...
}
```

Because checking symbols is a bit involved, we'll defer it to the `checkSymExpr` method.

### 8.5.2 Checking literal expressions

Literal expressions are all expressions with a direct value, including null, ints, bools and strings.

```ts
class Checker {
    // ...
    public checkExpr(expr: Expr): VType {
        // ...
        if (expr.kind.type === "null") {
            return { type: "null" };
        }
        if (expr.kind.type === "int") {
            return { type: "int" };
        }
        if (expr.kind.type === "bool") {
            return { type: "bool" };
        }
        if (expr.kind.type === "string") {
            return { type: "string" };
        }
        // ...
    }
    // ...
}
```

All literal types have a definite corresponding type.

### 8.5.3 Checking binary expressions

There are many kinds of binary expressions. To implement checking for them, we'll make a table of each combination. We'll assume that operand types are consistent.

```ts
const simpleBinaryOperations: {
    binaryType: string,
    operand: VType,
    result?: VType,
}[] = [
    { binaryType: "+", operand: { type: "int" } },
    { binaryType: "-", operand: { type: "int" } },
    { binaryType: "*", operand: { type: "int" } },
    // ...
    { binaryType: "and", operand: { type: "bool" }, result: { type: "int" } },
    // ...
    { binaryType: "==", operand: { type: "null" }, result: { type: "bool" } },
    { binaryType: "==", operand: { type: "int" }, result: { type: "bool" } },
    { binaryType: "==", operand: { type: "string" }, result: { type: "bool" } },
    { binaryType: "==", operand: { type: "bool" }, result: { type: "bool" } },
    // ...
];
// ...
class Checker {
    // ...
    public checkExpr(expr: Expr): VType {
        // ...
        if (expr.kind.type === "binary") {
            const left = this.checkExpr(expr.kind.left);
            const right = this.checkExpr(expr.kind.right);
            for (const operation of simpleBinaryOperations) {
                if (operation.binaryType !== expr.binaryType)
                    continue;
                if (!vtypesEqual(operation.operand, left))
                    continue;
                if (!vtypesEqual(left, right))
                    continue;
                return operation.result ?? operation.operand;
            }
            this.report(
                `cannot apply binary operation '${expr.binaryType}' `
                    + `on types '${vtypeToString(left)}' and '${vtypeToString(right)}'`,
            );
            return { type: "error" };
        }
        // ...
    }
    // ...
}
```

#### Exercises

1. Implement all previous types of binary operators.


```ts
class Checker {
    // ...
    public checkSymExpr(expr: Expr): VType {
        // ...
        if (expr.kind.type === "sym") {
            if (expr.kind.defType === "let") {
                const vtype = expr.kind.param.valueType;
                if (vtype.type === "error") {
                    return { type: "error" };
                }
                if (vtype.type === "unknown") {
                    this.report("unknown type", pos);
                    return { type: "error" };
                }
                return vtype;
            }
            if (expr.kind.defType === "fn") {
                const params = expr.kind.stmt.params
                    .map(param => this.checkType(param.etype!));
                const returnType = this.checkType(expr.kind.stmt.returnType);
                return { type: "fn", params, returnType };
            }
            if (expr.kind.defType === "fn_param") {
                return this.checkType(expr.kind.param.etype!);
            }
            if (expr.kind.defType === "builtin") {
                // ...
            }
            throw new Error(`unhandled defType ${expr.kind.defType}`);
        }
        // ...
    }
    // ...
}
```





