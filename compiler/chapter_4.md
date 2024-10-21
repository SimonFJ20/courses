
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
    }
    if (value.type === "null") {
        return "null";
    }
    if (value.type === "int") {
        return value.value.toString();
    }
    if (value.type === "string") {
        return `"${value.value}"`;
    }
    if (value.type === "bool") {
        return value.value ? "true" : "false";
    }
    if (value.type === "array") {
        const valueStrings = value.values
            .map(value => value.toString());
        return `[${valueStrings.join(", ")}]`;
    }
    if (value.type === "struct") {
        const fieldStrings = Object.entries(value.fields)
            .map(([key, value]) => `${key}: ${valueToString(value)}`);
        return `struct { ${fieldStrings.join(", ")} }`;
    }
    if (value.type === "fn") {
        return `<fn: ${value.fnDefId}>`;
    }
    if (value.type === "builtin_fn") {
        return `<builtin_fn: ${value.name}>`;
    }
    throw new Error("unexhaustive");
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
type Sym = { value: Value, pos?: Pos }

type SymMap = { [ident: string]: Sym }

class Syms {
    private syms: SymMap = {};

    public constructor(private parent?: Syms) {}
    // ...
}
```

The `Sym` structure represents a symbol, and contains its details such as the value and the position where the symbol is declared. The `SymMap` type is a key value map, which maps identifiers to their definition. To keep track of symbols in regard to scopes, we also define a `Syms` class. An instance of `Syms` is a node in a tree structure.

We'll define a method for defining symbols.

```ts
class Syms {
    // ...
    public define(ident: string, sym: Sym) {
        this.syms[ident] = sym;
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

And then, we'll define a method for getting the symbol by its identifier.

```ts
class Syms {
    // ...
    public get(ident: string): { ok: true, sym: Sym } | { ok: false } {
        if (ident in this.syms)
            return { ok: true, sym: this.syms[ident] };
        if (this.parent)
            return this.parent.get(ident);
        return { ok: false };
    }
    // ...
}
```

If the symbol is defined locally, return the symbol. Else if the parent node is defined, defer to the parent. Otherwise, return a not-found result.

## 4.3 Control flow

Most code will run with unbroken control flow, but some code will 'break' control flow. This is the case for return statements in functions and break statements in loops. To keep track of, if a return or break statement has been run, we'll define a data structure representing the control flow action of evaluated code.

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
function flowValue(value: Value): Flow {
    return { type: "value", value };
}
```

### 4.3.2 Flow utility functions

Because we're most interested in the control flow of value type, we'll make a function to conveniently filter out non-value control flow.

```ts
function expectValue(flow: Flow): [null, Flow] | [Value, Flow] {
    if (flow.type !== "value")
        return [null, flow];
    return [flow.value, flow];
}
```

We now have these 2 options for unwrapping control flow.

1)
```ts
const valueFlow = this.evalExpr(...);
if (valueFlow !== "value")
    return valueFlow;
const value = valueFlow.value;
```
2)
```ts
const [value, flow] = expectValue(this.evalExpr(...));
if (!value)
    return flow;
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
}
```

The `defineBuiltins` function will be defined later.

### 4.4.2 Function definitions

The evaluator needs a way to keep track of function definitions, so that we later can call and evaluate the function. Our definition of a function definition will be the following.

```ts
type FnDef = {
    params: Param[],
    body: Expr,
    id: number,
};
```

The parameters are needed, so that we can verify when calling, that we call with the correct amount of arguments. The body is the AST expression to be evaluated. And an identifier, so that we can refer to the definition by its id `fnDefId`.

```ts
class Evaluator {
    private fnDefs: FnDef[] = [];
    // ...
}
```

We'll also add an array of function definitions to the evaluator class. The index of a function definition will also be its id.

## 4.5 Expressions

Let's make a function `evalExpr` for evaluating expressions.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Expr): Flow {
        if (expr.kind.type === "error") {
            throw new Error("error in AST");
        }
        // ...
        throw new Error(`unknown expr type "${expr.kind.type}"`);
    }
    // ...
}
```

The `evalExpr` function will take an expression and a symbol table, match the type of the expression and return a flow. If the expression is an error, meaning an error in the AST, the evaluator throws an error. In case the expression type is unknown, an error is thrown with the error type in the message.

### 4.5.1 Identifiers

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "ident") {
            const result = syms.get(expr.kind.value);
            if (!result.ok)
                throw new Error(`undefined symbol "${expr.kind.value}"`);
            return flowValue(result.sym.value);
        }
        // ...
    }
    // ...
}
```

### 4.5.2 Literal expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "null") {
            return flowValue({ type: "null" });
        }
        if (expr.kind.type === "int") {
            return flowValue({ type: "int", value: expr.kind.value });
        }
        if (expr.kind.type === "string") {
            return flowValue({ type: "string", value: expr.kind.value });
        }
        if (expr.kind.type === "bool") {
            return flowValue({ type: "bool", value: expr.kind.value });
        }
        // ...
    }
    // ...
}
```

To evaluate a literal, we basically translate the value in AST form into a value, and then return the value.

### 4.5.3 Group expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "group") {
            return this.evalExpr(expr.kind.expr, syms);
        }
        // ...
    }
    // ...
}
```

To evaluate a group expression, we simply evaluate the contained expression.

### 4.5.4 Field expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "field") {
            const [subject, subjectFlow] = expectValue(this.evalExpr(expr.kind.subject, syms));
            if (!subject)
                return subjectFlow;
            if (subject.type !== "struct")
                throw new Error(`cannot use field operator on ${subject.type} value`);
            if (!(expr.kind.value in subject.fields))
                throw new Error(`field ${expr.kind.value} does not exist on struct`);
            return flowValue(subject.fields[expr.kind.value]);
        }
        // ...
    }
    // ...
}
```

We first evaluate the subject expression, break in case the control flow isn't a value and store the value. After checking that the value is a struct and that the field exists on the struct, the field's value is returned.

### 4.5.5 Index expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "index") {
            const [subject, subjectFlow] = expectValue(this.evalExpr(expr.kind.subject, syms));
            if (!subject)
                return subjectFlow;
            const [value, valueFlow] = expectValue(this.evalExpr(expr.kind.value, syms));
            if (!value)
                return valueFlow;
            if (subject.type === "struct") {
                if (value.type !== "string")
                    throw new Error(`cannot index into struct with ${value.type} value`);
                if (!(value.value in subject.fields))
                    return flowValue({ type: "null" });
                return flowValue(subject.fields[value.value]);
            }
            if (subject.type === "array") {
                if (value.type !== "int")
                    throw new Error(`cannot index into array with ${value.type} value`);
                if (value.value >= subject.values.length)
                    throw new Error("index out of range");
                if (value.value < 0) {
                    const negativeIndex = subject.values.length + value.value;
                    if (negativeIndex < 0 || negativeIndex >= subject.values.length)
                        throw new Error("index out of range");
                    return flowValue(subject.values[negativeIndex]);
                }
                return flowValue(subject.values[value.value]);
            }
            if (subject.type === "string") {
                if (value.type !== "int")
                    throw new Error(`cannot index into string with ${value.type} value`);
                if (value.value >= subject.value.length)
                    throw new Error("index out of range");
                if (value.value < 0) {
                    const negativeIndex = subject.value.length + value.value;
                    if (negativeIndex < 0 || negativeIndex >= subject.value.length)
                        throw new Error("index out of range");
                    return flowValue({ type: "int", value: subject.value.charCodeAt(negativeIndex) });
                }
                return flowValue({ type: "int", value: subject.value.charCodeAt(value.value) });
            }
            throw new Error(`cannot use index operator on ${subject.type} value`);
        }
        // ...
    }
    // ...
}
```

The index operator can be evaluated on a subject of either struct, array or string type. If evaluated on the struct type, we expect a string containing the field name. If the field does not exist, we return a null value. This is in contrast to the field operator, which throws an error, if no field is found. If the subject is instead an array, we expect a value of type int. We check if either the int value index or negative index is in range of the array values. If so, return the value at the index or the negative index. If the subject is a string, evaluation will behave similarly to an array, evaluating to an int value representing the value of the text character at the index or negative index.

The negative index is when a negative int value is passed as index, where the index will start at the end of the array. Given an array `vs` containing the values `["a", "b", "c"]` in listed order, the indices `0`, `1` and `2` will evaluate to the values `"a"`, `"b"` and `"c"`, whereas the indices `-1`, `-2`, `-3` will evaluate to the values `"c"`, `"b"` and `"a"`. A negative index implicitly starts at the length of the array and subtracts the absolute index value.

### 4.5.6 Call expressions

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "call") {
            const [subject, subjectFlow] = expectValue(this.evalExpr(expr.kind.subject, syms));
            if (!subject)
                return subjectFlow;
            const args: Value[] = [];
            for (const arg of expr.kind.args) {
                const [value, valueFlow] = expectValue(this.evalExpr(arg, syms));
                if (!value)
                    return valueFlow;
                args.push(value);
            }
            if (subject.type === "builtin_fn") {
                return this.executeBuiltin(subject.name, args, syms);
            }
            if (subject.type !== "fn")
                throw new Error(`cannot use call operator on ${subject.type} value`);
            if (!(subject.fnDefId in this.fnDefs))
                throw new Error("invalid function definition id");
            const fnDef = this.fnDefs[subject.fnDefId];
            if (fnDef.params.length !== args.length)
                throw new Error("incorrect amount of arguments in call to function");
            let fnScopeSyms = new Syms(this.root);
            for (const [i, param] of fnDef.params.entries()) {
                fnScopeSyms.define(param.ident, { value: args[i], pos: param.pos });
            }
            const flow = this.evalExpr(fnDef.body, fnScopeSyms);
            if (flow.type === "return")
                return flowValue(flow.value);
            if (flow.type !== "value")
                throw new Error(`${flow.type} on the loose!`);
            return flow;
        }
        // ...
    }
    // ...
}
```

The first thing we do is evaluate the subject expression of the call (`subject(...args)`). If that yields a value, we continue. Then we evaluate each of the arguments in order. If evaluation of an argument doesn't yield a value, we return immediately. Then, if the subject evaluated to a builtin value, we call `executeBuiltin`, which we will define later, with the builtin name, call arguments and symbol table. Otherwise, we assert that the subject value is a function and that a function definition with the id exists. We then check that the correct amount of arguments are passed. Then, we make a new symbol table with the root table as parent, which will be the called function's symbols. We assign each argument value to the corresponding parameter name, dictated by argument order. We then evaluate the function body. Finally, we check that the control flow results in either a value, which we simply return, or a return flow, which we convert to a value.

### 4.5.7 Unary expressions

Next, we will implement evaluation of unary expressions, meaning postfix expressions with one operand, such as when using the `not` operator.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "unary") {
            if (expr.kind.unaryType === "not") {
                const [subject, subjectFlow] = expectValue(this.evalExpr(expr.kind.subject, syms));
                if (!subject)
                    return subjectFlow;
                if (subject.type === "bool") {
                    return flowValue({ type: "bool", value: !subject.value });
                }
                throw new Error(`cannot apply not operator on type ${subject.type}`);
            }
            throw new Error(`unhandled unary operation ${expr.kind.unaryType}`);
        }
    // ...
    }
    // ...
}
```

The not operator should only work on values of type bool.

### 4.5.8 Binary expressions

Binary expressions conceptually are similiar to unary expressions, so too is the implementation.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "binary") {
            const [left, leftFlow] = expectValue(this.evalExpr(expr.kind.left, syms));
            if (!left)
                return leftFlow;
            const [right, rightFlow] = expectValue(this.evalExpr(expr.kind.right, syms));
            if (!right)
                return rightFlow;
            if (expr.kind.binaryType === "+") {
                if (left.type === "int" && right.type === "int") {
                    return flowValue({ type: "int", value: left.value + right.value });
                }
                if (left.type === "string" && right.type === "string") {
                    return flowValue({ type: "string", value: left.value + right.value });
                }
                throw new Error(`cannot apply ${expr.kind.binaryType} operator on types ${left.type} and ${right.type}`);
            }
            if (expr.kind.binaryType === "==") {
                if (left.type === "null" && right.type === "null") {
                    return flowValue({ type: "bool", value: true });
                }
                if (left.type === "null" && right.type !== "null") {
                    return flowValue({ type: "bool", value: false });
                }
                if (left.type !== "null" && right.type === "null") {
                    return flowValue({ type: "bool", value: false });
                }
                if ((left.type === "int" || left.type === "string" || left.type === "bool")
                    && left.type === right.type) {
                    return flowValue({ type: "bool", value: left.value === right.value });
                }
                throw new Error(`cannot apply ${expr.kind.binaryType} operator on types ${left.type} and ${right.type}`);
            }
            throw new Error(`unhandled binary operation ${expr.kind.binaryType}`);
        }
        // ...
    }
    // ...
}
```

Add operation (`+`) is straight forward. Evaluate the left expressions, evaluate the right expressions and return a value with the result of adding left and right. Addition should work on integers and strings. Adding two strings results in a new string consisting of the left and right values concatenated.

The equality operator (`==`) is a bit more complicated. It only results in values of type bool. You should be able to check if any value is null. Otherwise, comparison should only be allowed on two values of same type.

#### Exercises

1. Implement the binary operators: `-`, `*`, `/`, `!=`, `<`, `>`, `<=`, `>=`, `or` and `and`.

### 4.5.9 If expressions

An if expression should evaluate either the truthy expression or the falsy expression, depending on the condition expression. It should return the resulting value or a null value in case the condition is false and no falsy expression was supplied.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "if") {
            const [cond, condFlow] = expectValue(this.evalExpr(expr.kind.cond, syms));
            if (!cond)
                return condFlow;
            if (cond.type !== "bool")
                throw new Error(`cannot use value of type ${cond.type} as condition`);
            if (cond.value)
                return this.evalExpr(expr.kind.truthy, syms);
            if (expr.kind.falsy)
                return this.evalExpr(expr.kind.falsy, syms);
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

We start by evaluating the condition expression. The condition value should be a bool value. Then, depending on the condition value, we either evaluate the truthy or the falsy branch, or return null.

### 4.5.10 Loop expressions

Next, we'll implement the loop expression. The loop expression will repeatedly evaluate the body expression while throwing away the resulting values, until it results in breaking control flow. If the control flow is of type break, the loop expression itself will evaluate to the break's value.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "loop") {
            while (true) {
                const flow = this.evalExpr(expr.kind.body, syms);
                if (flow.type === "break")
                    return flowValue(flow.value);
                if (flow.type !== "value")
                    return flow;
            }
        }
        // ...
    }
    // ...
}
```

First, start an infinite loop. In each iteration, evaluate the loop body. If the resulting control flow is breaking, return the break value. If the control flow is not a value, meaning return or other unimplemented control flow, just return the control flow. Otherwise, discard the value and repeate.


## 4.5.11 Block expressions

The block expressions evaluate the statements in order, discard their values and return the value of the tailing expression if present, else a null. Symbols are scoped inside the block.

```ts
class Evaluator {
    // ...
    public evalExpr(expr: Expr, syms: Syms): Flow {
        // ...
        if (expr.kind.type === "block") {
            let scopeSyms = new Syms(syms);
            for (const stmt of expr.kind.stmts) {
                const flow = this.evalStmt(stmt, scopeSyms);
                if (flow.type !== "value")
                    return flow;
            }
            if (expr.kind.expr)
                return this.evalExpr(expr.kind.expr, scopeSyms);
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

Make a new symbol table with outer symbol table as parent. Iterate through each statement. If a statement results in breaking flow, return the flow. Then evaluate the tailing expression if present and return the result or a null value.

### Excercises

1. \* Refactor `evalExpr`, eg. move each expression type into its own function for evaluation, in order to make the code more manageable.
2. \* Implement hex literals, array and struct literal syntax in evaluator.

## 4.6 Statements

For evaluating statements, we'll make a function called `evalStmt` .

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        if (stmt.kind.type === "error") {
            throw new Error("error in AST");
        }
        // ...
        throw new Error(`unknown stmt type "${stmt.kind.type}"`);
    }
    // ...
}
```

The `evalStmt` function, like `evalExpr` or expressions, will take a statement and a symbol table, match the type of the statement and return a flow. Handle errors in AST and unknown statement types likewise.

### 4.6.1 Break statements

The break statement simply returns a breaking flow, and an optional value, depending on if it is present.

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "break") {
            if (!stmt.kind.expr)
                return { type: "break", value: { type: "null" } };
            const [value, valueFlow] = expectValue(this.evalExpr(stmt.kind.expr, syms));
            if (!value)
                return valueFlow;
            return { type: "break", value };
        }
        // ...
    }
    // ...
}
```

If the expression is not a value, the resulting flow is returned, as inner break statements or return flow should take precedence.

### 4.6.2 Return statements

The return statement returns a returning flow, and, like break, an optional value, depending on if it is present.

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "return") {
            if (!stmt.kind.expr)
                return { type: "return", value: { type: "null" } };
            const [value, valueFlow] = expectValue(this.evalExpr(stmt.kind.expr, syms));
            if (!value)
                return valueFlow;
            return { type: "return", value };
        }
        // ...
    }
    // ...
}
```

If the expression is not a value, like with the break statement, the resulting flow is returned, as inner break and return statements take precedence.

### 4.6.3 Expression statements

An expression statement is simply an expression used as a statement. It should be evaluated accordingly.

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "expr") {
            return this.evalExpr(stmt.kind.expr, syms);
        }
        // ...
    }
    // ...
}
```

### 4.6.4 Assignment statements

An assignment statement should assign a new value to either a symbol, a field or an array index. Because of this, we'll also have to look at the first layer of the subject expression.

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "assign") {
            const [value, valueFlow] = expectValue(this.evalExpr(stmt.kind.value, syms));
            if (!value)
                return valueFlow;
            if (stmt.kind.subject.kind.type === "ident") {
                const ident = stmt.kind.subject.kind.value;
                const getResult = syms.get(ident);
                if (!getResult.ok)
                    throw new Error(`cannot assign to undefined symbol "${ident}"`);
                const { sym } = getResult;
                if (sym.value.type !== value.type)
                    throw new Error(`cannot assign value of type ${value.type} to symbol originally declared ${sym.value.type}`);
                sym.value = value;
                return flowValue({ type: "null" });
            }
            if (stmt.kind.subject.type === "field") {
                const [subject, subjectFlow] = expectValue(this.evalExpr(stmt.kind.subject.kind.subject, syms));
                if (!subject)
                    return subjectFlow;
                if (subject.type !== "struct")
                    throw new Error(`cannot use field operator on ${subject.type} value`);
                subject.fields[stmt.kind.subject.kind.value] = value;
                return flowValue({ type: "null" });
            }
            if (stmt.kind.subject.kind.type === "index") {
                const [subject, subjectFlow] = expectValue(this.evalExpr(stmt.kind.subject.kind.subject, syms));
                if (!subject)
                    return subjectFlow;
                const [index, indexFlow] = expectValue(this.evalExpr(stmt.kind.subject.kind.value, syms));
                if (!index)
                    return indexFlow;
                if (subject.type === "struct") {
                    if (index.type !== "string")
                        throw new Error(`cannot index into struct with ${index.type} value`);
                    subject.fields[index.value] = value;
                    return flowValue({ type: "null" });
                }
                if (subject.type === "array") {
                    if (index.type !== "int")
                        throw new Error(`cannot index into array with ${index.type} value`);
                    if (index.value >= subject.values.length)
                        throw new Error("index out of range");
                    if (index.value >= 0) {
                        subject.values[index.value] = value;
                    } else {
                        const negativeIndex = subject.values.length + index.value;
                        if (negativeIndex < 0 || negativeIndex >= subject.values.length)
                            throw new Error("index out of range");
                        subject.values[negativeIndex] = value;
                        return flowValue({ type: "null" });
                    }
                    return flowValue({ type: "null" });
                }
                throw new Error(`cannot use field operator on ${subject.type} value`);
            }
            throw new Error(`cannot assign to ${stmt.kind.subject.kind.type} expression`);
        }
        // ...
    }
    // ...
}
```

We start by evaluating the assigned value, meaning the value expression is evaluated before the assigned-to subject expression.

For assigning to identifiers, eg. `a = 5`, we start by finding the symbol. If not found, we raise an error. Then we check that the old and new values are the same type. Then we assign the new value to the symbol.

For assigning to fields, eg. `a.b = 5`, we evaluate the inner (field expression) subject expression, `a` in this case. Then we reassign the field value or assign to a new field, if it doesn't exist.

And then, for assigning to indices, eg. `a[b] = 5`, we evaluate the inner (index expression) subject `a` and index value `b` in that order. If `a` is a struct, we check that `b` is a string and assign to the field, the string names. Else, if `a` is an array, we check that `b` is an int and assign to the index or negative index (see 4.5.5 Index expressions).

### 4.6.5 Let statements

A let statement declares and initializes a new symbol.

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "let") {
            if (syms.definedLocally(stmt.kind.param.ident))
                throw new Error(`cannot redeclare symbol "${stmt.kind.param.ident}"`);
            const [value, valueFlow] = expectValue(this.evalExpr(stmt.kind.value, syms));
            if (!value)
                return valueFlow;
            syms.define(stmt.kind.param.ident, { value: value, pos: stmt.pos });
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

A let statement cannot redeclare a symbol that already exists in the same scope. Therefore we first check if that the symbol does not already exist. Then we evaluate the value and define the symbol.

## 4.6.6 Function definition statements

```ts
class Evaluator {
    // ...
    public evalStmt(stmt: Stmt, syms: Syms): Flow {
        // ...
        if (stmt.kind.type === "fn") {
            if (syms.definedLocally(stmt.kind.ident))
                throw new Error(`cannot redeclare function "${stmt.kind.ident}"`);
            const { params, body } = stmt.kind;

            let paramNames: string[] = [];
            for (const param of params) {
                if (paramNames.includes(param.ident))
                    throw new Error(`cannot redeclare parameter "${param.ident}"`);
                paramNames.push(param.ident);
            }

            const id = this.fnDefs.length;
            this.fnDefs.push({ params, body, id });
            syms.define(stmt.kind.ident, { value: { type: "fn", fnDefId: id }, pos: stmt.pos });
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

First, like the let statement, we chech that the identifier has not already been declared as a symbel. Then we check that no two parameters have the same name. We then get a function definition id, store the function definition and define a symbol with the function value.

## 4.7 Statements

We'll want a function for evaluating the top-level statements.

```ts
class Evaluator {
    // ...
    public evalStmts(stmts: Stmt[], syms: Syms) {
        let scopeSyms = new Syms(syms);
        for (const stmt of stmts) {
            const flow = this.evalStmt(stmt, scopeSyms);
            if (flow.type !== "value")
                throw new Error(`${flow.type} on the loose!`);
        }
    }
    // ...
}
```

Any break or return flow is at this point an error. We discard any resulting value and the function should not return anything.

## 4.8 Builtin functions

Lastly, we'll define the builtin functions. A builtin function is a function that tells the evaluator to do something, as opposed to a normal function which is simply evaluated. Functions to interact with the outside world, need to be builtins in this evaluator.

First, we'll define the function called `executeBuiltin`, which takes a builtin name, arguments and the relevant symbol table.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        throw new Error(`unknown builtin "${name}"`);
    }
    // ...
}
```

### 4.8.1 Array and struct constructors

Because we don't have literal syntax for arrays and structs, we need builtins to create a value of each type.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "array") {
            if (args.length !== 0)
                throw new Error("incorrect arguments");
            return flowValue({ type: "array", values: [] });
        }
        if (name === "struct") {
            if (args.length !== 0)
                throw new Error("incorrect arguments");
            return flowValue({ type: "struct", fields: {} });
        }
        // ...
    }
    // ...
}
```

No arguments should be given.

### 4.8.2 Array push

To add values to an array, we need a push function.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "array_push") {
            if (args.length !== 2)
                throw new Error("incorrect arguments");
            const array = args[0];
            const value = args[1];
            if (array.type !== "array")
                throw new Error("incorrect arguments");
            array.values.push(value);
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

This function expects that the first parameter is an array, and the second is the value to push.

### 4.8.3 Array length


We'll also need a way to get the length of an array.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "array_len") {
            if (args.length !== 1)
                throw new Error("incorrect arguments");
            const array = args[0];
            if (array.type !== "array")
                throw new Error("incorrect arguments");
            return flowValue({ type: "int", value: array.values.length });
        }
        // ...
    }
    // ...
}
```

### 4.8.3 String functions

Like arrays, we need push and length functions for strings too.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "string_concat") {
            if (args.length !== 2)
                throw new Error("incorrect arguments");
            if (args[0].type === "string") {
                if (args[1].type === "string")
                    return flowValue({ type: "string", value: args[0].value + args[1].value });
                if (args[1].type === "int")
                    return flowValue({ type: "string", value: args[0].value + String.fromCharCode(args[1].value) });
            }
            if (args[0].type === "int" && args[1].type === "string") {
                return flowValue({ type: "string", value: String.fromCharCode(args[0].value) + args[1].value });
            }
            throw new Error("incorrect arguments");
        }
        if (name === "string_len") {
            if (args.length !== 1)
                throw new Error("incorrect arguments");
            if (args[0].type !== "string")
                throw new Error("incorrect arguments");
            return flowValue({ type: "int", value: args[0].value.length });
        }
        // ...
    }
    // ...
}
```

The function `string_concant` takes either two strings or a string and a character (int). A new string is returning containing both values concatonated.

### 4.8.4 Println

To print text to the screen, we need a print function. We'll make one called `println` which will also print a newline afterwards.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "println") {
            if (args.length < 1 || args[0].type !== "string")
                throw new Error("incorrect arguments");
            let msg = args[0].value;
            for (const arg of args.slice(1)) {
                if (!msg.includes("{}"))
                    throw new Error("incorrect arguments");
                msg.replace("{}", valueToString(arg));
            }
            console.log(msg);
            return flowValue({ type: "null" });
        }
        // ...
    }
    // ...
}
```

This function takes a format-string as the first argument, and, corresponding to the format-string, values to be formatted in the correct order.

Examples of how to use the function follows.

```
println("hello world")

let a = "world";
println("hello {}", a);

println("{} + {} = {}", 1, 2, 1 + 2);
```

### 4.8.5 Exit

Normally, the evaluator will return a zero-exit code, meaning no error. In case the program should result in an error code, we'll need an exit function.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "exit") {
            if (args.length !== 1)
                throw new Error("incorrect arguments");
            if (args[0].type !== "int")
                throw new Error("incorrect arguments");
            const code = args[0].value;
            // Deno.exit(code)
            // process.exit(code)
            throw new Error(`exitted with code ${code}`);
        }
        // ...
    }
    // ...
}
```

### 4.8.6 Convert values to strings

This function will return a string representation of any value.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "to_string") {
            if (args.length !== 1)
                throw new Error("incorrect arguments");
            return flowValue({ type: "string", value: valueToString(args[0]) });
        }
        // ...
    }
    // ...
}
```

### 4.8.7 Convert strings to integers

This function will try to convert a string into an integer.

```ts
class Evaluator {
    // ...
    private executeBuiltin(name: string, args: Value[], syms: Syms): Flow {
        // ...
        if (name === "string_to_int") {
            if (args.length !== 1 || args[0].type !== "string")
                throw new Error("incorrect arguments");
            const value = parseInt(args[0].value);
            if (isNaN(value))
                return flowValue({ type: "null" });
            return flowValue({ type: "int", value });
        }
        // ...
    }
    // ...
}
```

The function will return a null value, in case conversion fails.

## 4.9 Define builtins

Finally, we need a way to define the builtin functions in the symbol table.

```ts
class Evaluator {
    // ...
    public defineBuiltins() {
        function defineBuiltin(syms: Syms, name: string) {
            syms.define(name, { value: { type: "builtin_fn", name } });
        }

        defineBuiltin(this.root, "array");
        defineBuiltin(this.root, "struct");
        defineBuiltin(this.root, "array_push");
        defineBuiltin(this.root, "array_len");
        defineBuiltin(this.root, "string_concat");
        defineBuiltin(this.root, "string_len");
        defineBuiltin(this.root, "println");
        defineBuiltin(this.root, "exit");
    }
    // ...
}
```

This function will be called by the consumer before evaluation.

## Exercises

1. Implement an optional feature such that, every time a symbol is defined (let statement or function definition), a record is stored containing the identifier, it's position and the value type. This could be a message printed to the console like `Symbol defined ${ident}: ${value.type} at ${pos.line}:${pos.col}`, eg. `Symbol defined a: int at 5:4`.
2. Implement a similar optional feature such that, every time a function is called, a record is stored containing function id or name and every argument and their type. This could be a console message like `Function called my_function(value: int, message: string)`;
3. \* Implement propagating errors (exceptions). For inspiration, look at Lua's `error` and `pcall` functions [here (PIL 8.4)](https://www.lua.org/pil/8.4.html) and [here (PIL 8.5)](https://www.lua.org/pil/8.5.html).
4. \* Do a performance assessment of the evaluator. Is this fast or slow? Can you explain why this way of executing code may be fast or slow?

