
# 8 Types

We'd like to be able to compile to a relatively low level. To this end, we need to have predetermined the types of the values used in the program code.

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



### 8.1.2 Parsing parameters

Both function definitions and let-statements use parameters.

### 8.1.3 Parsing functions

## 8.2 Types in AST

```ts
type Expr = {
    // ...
    valueType: ValueType,
}
```
```ts
type Stmt = {
    // ...
    valueType: ValueType,
}
```

