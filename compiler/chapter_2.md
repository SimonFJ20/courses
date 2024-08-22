
## 2 Specifying a language

**You can skip this chapter and come back to it, if you need.**

The language will at first consist of these features.

- Literals
- Expressions
- Variables
- Blocks
- If-conditions
- Loops
- Function calls
- Function definitions

## 2.1 Literals

Integers (ints) are numbers without decimal point.

Strings are text enclosed in double-quotes `" "`. Backslash `\` can be used to escape `"` and invoke specialt characters `\n` and `\t` for newline and tab characters.

Booleans (bools) represent logical true and false and are the symbols `true` and `false`.

Null represents a non-value (null value) and is the symbol `null`.

Examples:
```
0
123
"hello"
"{ \"key\": 123 }"
true
false
null
```

## 2.2 Expressions

I'll be reusing the prefix notation expressions from chapter 1. These are also called S-expressions.

I'll be using logical expressions (`or`, `and` and `not`), comparison expressions (`==`, `!=`, `<` and `>`), arithmetic expressions (`+`, `-`, `*`, `/`) and group expressions (`(...)`).

The grammer, meaning the rules for the structure of the code, is the following:
```
expr ->
    ...
    |   "or" expr expr
    |   "and" expr expr
    |   "not" expr
    |   "==" expr expr
    |   "!=" expr expr
    |   "<" expr expr
    |   ">" expr expr
    |   "+" expr expr
    |   "-" expr expr
    |   "*" expr expr
    |   "/" expr expr
    |   "(" expr ")"
    ...
    |   literal
```

Notice `not` only has 1 operand.

Examples:
```
+ 1 * 2 3

+ + + 1 2 3 4

* 3 (* 3 3)
```

## 2.3 Variables

A variable is a storage container for a value.

A variable is defined using a `let`-statement. A variable is used by refering to its name. A variable can be reassigned after definition.

Variables are block-scoped, more on this later.

Grammar:
```
let ->  "let" ident "=" expr ";"

assign -> ident "=" expr ";"

expr ->
    ...
    | ident
    ...
```

Examples:
```
let a = 5;

a = 10;

+ a 10
```

## 2.4 Blocks

A block is code enclosed in curly-braces (`{` and `}`). They can be used as both statements and expressions. Single-line statements in a block need semicolon (`;`) at the end. Expressions can be used as statements.

If the last entry in a block is an expression and the semicolon is omitted, the block as a whole yields that value.

Grammar:
```
block -> "{" statements expr:? "}"

expr ->
    ...
    | block
    ...
```

Examples:
```
{}

{
    let a = 5;
}

let a = {
    let b = 5;
    + b 5
};
```

## 2.5 If-statements

An `if`-statement has a condition and a block it  runs, if the condition evaluates to `true` ("truthy"), and an optional `else`-block, if it the condition evaluates to `false`.  

Grammar:
```
if  ->  "if" expr block ("else" block):?

expr ->
    ...
    | if
    ...
```

I've added `if` to `expr`, because, in addition to a statement, I want to be able to use `if` as an expression just like blocks.

Examples:
```
let a = 5;
if == b 5 {
    a = 5;
}

let value = if > a b { c } else { d };
```

## 2.6 Loops

`loop`-statements are blocks that run continually (from top to bottom, then from the top again) until a `break`-statement is evaluated.

Loops can also be expressions, as a value can be supplied when breaking.

Grammar:
```
loop    ->  "loop" block

break   ->  "break" expr:? ";"

expr ->
    ...
    | loop
    ...
```

Examples:
```
let i = 0;
let v = 1;
loop {
    if not < i 10 {
        break;
    }
    v = * v 2;
    i = + i 1;
}

let v = loop {
    break 5;
};
```

## 2.7 Function calls

Grammar:
```
expr ->
    ...
    | expr "(" args ")"
    ...
```

Examples:
```
let a = pow(2, 3);

exit();
```

## 2.8 Function definition

Grammar:
```
fn      ->  "fn" ident "(" params ")" block

return  ->  "return" expr:? ";"
```

