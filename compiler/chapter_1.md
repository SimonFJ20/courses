
# 1 Make a calculator

As an introduction I'll show, how I would make simple calculator. To do this, I'll use many of the principles of compiler theory.

## 1.1 Source language

This is what I'll propose for the input language.

Subtract 12 from 34:
```py
- 34 12
```
Add 2 to the result of multiplying 3 by 4:
```py
+ 2 * 3 4
```
Divide 21 by 3:
```py
/ 21 3
```

The rationale: I've chosen [polish notation](https://en.wikipedia.org/wiki/Polish_notation) also called prefix notation as it suits the needs of the language while being simple to parse.

### Exercises

1. Try to translate the math expressions `2 + 3 * 4` and `(2 + 3) * 4` into the source language described. How do the two expressions differ?
2. Does the source language need parenthesis `(`, `)`? Why/why not?

## 1.2 Representation in code

One part of the process, is designing how the input program, or calculator expressions in our case, should be represented in code. I'll propose a representation like the following for a simple calculator.

For the input program:
```
+ 1 * 2 3
```

I'll propose a representation like
```
Add {
    left: 1
    right: Multiply {
        left: 2
        right: 3
    }
}
```

I would implement this in Javascript like so:
```js
const expr = {
    type: "add",
    left: { type: "int", value: 1 },
    right: {
        type: "multiply",
        left: { type: "int", value: 2 },
        right: { type: "int", value: 3 },
    },
};
```

### Exercises

1. Make the expression representation for the math expressions `+ / 9 3 1`.

## 1.3 Evaluating expressions

To evaluate the expressions means to calculate the result. I've shown how to represent the input program above in code form.

To evaluate, we use a function, which takes each 'node' and calculates the result. I'll propose this implementation in Javascript:

```ts
function evaluateExpr(node) {
    if (node.type === "int") {
        return node.value;
    } else if (node.type === "add") {
        const left = evaluateExpr(node.left);
        const right = evaluateExpr(node.right);
        return left + right;
    } else if (node.type === "multiply") {
        const left = evaluateExpr(node.left);
        const right = evaluateExpr(node.right);
        return left * right;
    } else {
        throw new Error("unknown expr type");
    }
}
```

Using this function, we can try evaluating the expression formed above:

```js
const result = evaluateExpr(expr);
console.log(result); // should be 7
```

Important to notice in the implementation is the calls to the function itself. (We call `evaluateExpr` inside the body of `evaluateExpr`). This is called recursion, it's perfectly allowed and is quite useful for this program.

### Exercises

1. Implement `subtract` and `divide`.
2. Is it possible, to make an expression evaluating to a negative result. How/why not?

## 1.4 Parsing source code

Users don't want to write their programs as object representation. Instead, they want to write their programs in source code. To make our program understand the input program, it needs to be able to parse the source language.

I'll propose an implementation in which I'll split parsing up in 2 steps.

### 1.4.1 Lexer

The first step take will take text like this:
```
+ 12 34
```
and produce a list of *tokens* representing the input program like this:
```
Plus Int(12) Int(34)
```

In Javascript, I'll propose implementing the representation like this:

```js
const tokens = [
    { type: "+" },
    { type: "int", value: 12 },
    { type: "int", value: 34 },
];
```

Important to note, is that the tokens in this step do not represent structure in the code. (The representation doesn't know it is an add-expression). It instead does 2 things:
1. It groups together text characters. (`12` is 2 text characters but 1 token).
2. It assigns a type to each token, (`12` and `34` are both integers).

I'll propose this implementation in Javascript:

```js
function lex(text) {
    let i = 0;
    const tokens = [];
    while (i < text.length) {
        // ignore whitespace
        while (" \t\n".includes(text[i])) {
            i += 1;
        }
        if ("1234567890".includes(text[i])) {
            let textValue = "";
            while ("1234567890".includes(text[i])) {
                textValue += text[i];
                i += 1;
            }
            tokens.push({ type: "int", value: parseInt(textValue) });
        } else if (text[i] === "+") {
            i += 1;
            tokens.push({ type: "+" });
        } else if (text[i] === "*") {
            i += 1;
            tokens.push({ type: "*" });
        } else {
            throw new Error("illegal character")
        }
    }
    return tokens;
}
```

Using some if-statement and loops, this implementation goes through each `i` character `text[i]` of the input string `text`. In this step, I discard whitespace, meaning I don't produce tokens for whitespace, I instead skip over it.

A visualization of how this code would lex the expression `+ 12 34` could look like this:

```
text        i   state           tokens

+ 12 34     0   make Plus       []
^
+ 12 34     1   skip whitespace [Plus]
 ^
+ 12 34     2   make Int        [Plus]
  ^
+ 12 34     3   make Int        [Plus]
   ^
+ 12 34     4   skip whitespace [Plus Int(12)]
    ^
+ 12 34     5   make Int        [Plus Int(12)]
     ^
+ 12 34     6   make Int        [Plus Int(12)]
      ^
+ 12 34     7   done            [Plus Int(12) Int(34)]
       ^
```

#### Exercises

1. Implement `-` and `/` for subtraction and division.
2. \* Add position (line and column number) to each token.
3. \*\* Rewrite lexer into an iterator (eg. use the OOP iterator pattern).


### 1.4.2 Parser

I mentioned the tokens don't represent any structure. To know that `+ 1 2` is an addition operation, we need to parse the structure.

When parsing, it's common to have a set of rules, determining how code should be parsed. These rules could look like this, for the source language:

```
expr ->
    |   "+" expr expr
    |   "*" expr expr
    |   int
```

A visualization of how these rules could be applied to the expression `+ 1 * 2 3` could look like this:

```
input           state
+ 1 * 2 3       <expr>
^
+ 1 * 2 3       + <expr> <expr>
  ^
+ 1 * 2 3       + 1 <expr>
    ^
+ 1 * 2 3       + 1 * <expr> <expr>
      ^
+ 1 * 2 3       + 1 * 2 <expr>
        ^
+ 1 * 2 3       + 1 * 2 3
         ^
```

I'll propose this implementation for the parser in Javascript:

```js
class Parser {
    constructor(tokens) {
        this.tokens = tokens;
        this.i = 0;
    }

    parseExpr() {
        if (this.done()) {
            throw new Error("expected expr, got end-of-file");
        } else if (this.current().type === "+") {
            this.step();
            const left = this.parseExpr();
            const right = this.parseExpr();
            return { type: "add", left, right };
        } else if (this.current().type === "*") {
            this.step();
            const left = this.parseExpr();
            const right = this.parseExpr();
            return { type: "multiply", left, right };
        } else if (this.current().type === "int") {
            const value = this.current().value;
            this.step();
            return { type: "int", value };
        } else {
            throw new Error("expected expr");
        }
    }

    step() { this.i += 1; }
    done() { return this.i >= this.tokens.length; }
    current() { return this.tokens[this.i]; }
}
```

Notice that I again use recursion, ie. I call `parseExpr` inside the `parseExpr` function.

The implemented parser goes through each token and checks the structure of the program using if-statements. The `parseExpr` function should somewhat resemble the rules for `expr` defined above.

#### Exercises

1. Implement subtraction and division.
2. \* Add position (line and column number) to each expression.
3. \*\* Rewrite parser, to use infix notation (`1 + 2 * 3`) instead of prefix/polish notation (`+ 1 * 2 3`).

## 1.5 Putting it together

Then I'll propose a Read-Evaluate-Print-Loop (REPL) implemented as so:
```js
const text = "+ 1 2";

const tokens = lex(text);
const parser = new Parser(tokens);
const expr = parser.parseExpr();
const result = evaluateExpr(expr);
console.log(result);
```

To run the code, you could use NodeJS.
1. Make a file called `calculator.js`.
2. Write the code in the file.
3. Save the file (important).
4. Use a terminal (cmd, bash, powershell, etc.) and run the command `node calculator.js`.

## Exercises

1. Implement comments. A comment could for example start with a `#`-character and stop at line end `\n`. (Tip: examine how whitespace is ignored)
2. \* Implement constants such as `pi`, and make it evaluate to `3.14` or `4` for ease.
3. \* Implement decimal numbers (eg. `1.23`).
4. \*\* Implement variables such that you can write for example `let a = + 2 3` and then write `* 4 a` in the next expression.
5. \*\* Implement comparison and if-expressions. The syntax could look like `"==" expr expr` and `"if" expr "?" expr ":" expr`.
6. \*\* Implement function calls such that you can write `(sqrt 9)`, `sqrt` being a built in function and `9` being the first and only argument. Function calls should themselves be expressions, so that for example `+ 1 (sqrt 9)` is valid.
7. \*\* (Impressive) Implement function definitions. The syntax could look like `fn doubleIt a = * a 2`. Then after the definition, the function can be called, for example like `(doubleIt 5)`.
8. \*\* (Really impressive) Implement higher level function expressions, meaning for example `fn addOne a = + a 1`, `fn applyTwice f v = (f (f v))`, then `(applyTwice addOne 2)` will evaluate to `4`. Implement [currying](https://en.wikipedia.org/wiki/Currying) to be god level.

