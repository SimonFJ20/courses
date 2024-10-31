
# Types

We'd like to be able to compile to a relatively low level. To this end, we need to have predetermined the types of the values used in the program code.

## Explicit types

For a type checker to be able to determine all the types of all values, the program is required to contain the information need. That information will be dictated by the programmer, therefore the programmer needs a way to write types explicitly. We'll do this by adding types to the language.

### Syntax

We'll need explicit typing for the following types: null, int, string, bool, array, struct and function.

```
```

## Types in AST

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

