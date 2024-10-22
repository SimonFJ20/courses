
# 5 Make a test program

To test the evaluator and any future runtimes we'll remake the calculator from chapter 1.

## 5.1 Hello world

First, we'll need to setup the parser and evaluator.

I'll show this example using Deno.

```ts
const filename = Deno.args[0];
const text = await Deno.readTextFile(filename);
const parser = new Parser(new Lexer(text));
const ast = parser.parseStmts();
const evaluator = new Evaluator();
evaluator.defineBuiltins();
evaluator.evalStmts(ast);
```

Now we can write source code in a file.
```rs
println("hello world");
```
Save it to a file, eg. `program.txt`, run it with the following command.
```sh
deno run -A main.ts program.txt
```
And it should output the following.
```
hello world
```

## 5.2 Representation in code

We'll use the same representation in code as we did in chapter one. The following is how it was done in Javascript.

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

To do this is our language, (unless you've implemented struct literals), we'll need to use the `struct` builtin and field expression assignment.

```rs
let expr = struct();
expr["type"] = "add";
expr["left"] = struct();
expr.left["type"] = "int";
expr.left["value"] = 1;
expr["right"] = struct();
expr.right["type"] = "multiply";
expr.right["left"] = struct();
expr.right.left["type"] = "int";
expr.right.left["value"] = 2;
expr.right["right"] = struct();
expr.right.right["type"] = "int";
expr.right.right["value"] = 3;
```

If we print this value, we should expect an output like the following.
```
{ type: "add", left: { type: "int", value: 1 }, right: { type: "multiply", left: { type: "int", value: 2 }, right: { type: "int", value: 3 } } }
```

## 5.3 Evaluating expressions

Exactly like in chapter 1, we need a function for evaluating the expression structure described above.

```rs
fn eval_expr(node) {
    if (node.type == "int") {
        return node.value;
    }
    if (node.type == "add") {
        let left = eval_expr(node.left);
        let right = eval_expr(node.right);
        return + left right;
    }
    if (node.type == "multiply") {
        let left = eval_expr(node.left);
        let right = eval_expr(node.right);
        return * left right;
    }
    println("unknown expr type");
    exit(1);
}
```

### Exercises

1. Implement `subtract` and `divide`.


## 5.4 Parsing source code

### 5.4.1 Lexer

```rs
fn char_in_string(ch, val) {
    let i = 0;
    let val_length = string_len(val);
    loop {
        if >= i val_len {
            break;
        }
        if == ch val[i] {
            return true;
        }
        i = + i 1;
    }
    false
}

fn lex(text) {
    let i = 0;
    let tokens = array();
    let text_length = array_len(text);
    loop {
        if >= i text_length {
            break;
        }
        loop {
            if not char_in_string(text[i], " \t\n") {
                break;
            }
            i = + i 1;
        }
        if char_in_string(text[i], "1234567890") {
            let text_val = "";
            loop {
                if not char_in_string(text[i], "1234567890") {
                    break;
                }
                text_val = string_concat(text_val, text[i]);
                i = + i 1;
            }
            let token = struct();
            token["type"] = "int";
            token["value"] = string_to_int(text_val);
            array_push(token);
        } else if == text[i] "+"[0] {
            i = + i 1;
            let token = struct();
            token["type"] = "+"[0];
            array_push(token);
        } else if == text[i] "*"[0] {
            i = + i 1;
            let token = struct();
            token["type"] = "*"[0];
            array_push(token);
        } else {
            println("illegal character");
            exit(1);
        }
    }
    tokens
}
```

#### Exercises

1. Implement `-` and `/` for subtraction and division.
2. \* Add position (line and column number) to each token.
3. \*\* Rewrite lexer into an iterator (eg. use the OOP iterator pattern).

### 5.4.2 Parser

```rs
fn parser_new(tokens) {
    let self = struct();
    self["tokens"] = tokens;
    self["i"] = 0;
}

fn parser_step(self) { self.i = + self.i 1; }
fn parser_done(self) { >= self.i array_len(self.tokens) }
fn parser_current(self) { self.tokens[self.i] }

fn parser_parse_expr(self) {
    if parser_done(self) {
        println("expected expr, got end-of-file");
        exit(1);
    } else if == parser_current(self) "+" {
        parser_step(self);
        let left = parser_parse_expr(self);
        let right = parser_parse_expr(self);
        let node = struct();
        node["type"] = "add";
        node["left"] = left;
        node["right"] = right;
        node
    } else if == parser_current(self) "*" {
        parser_step(self);
        let left = parser_parse_expr(self);
        let right = parser_parse_expr(self);
        let node = struct();
        node["type"] = "multiply";
        node["left"] = left;
        node["right"] = right;
        node
    } else if == parser_current(self) "int" {
        let value = this.current().value;
        parser_step(self);
        node["type"] = "int";
        node["value"] = value;
        node
    } else {
        println("expected expr");
        exit(1);
    }
}
```


#### Exercises

1. Implement subtraction and division.
2. \* Add position (line and column number) to each expression.

## 5.5 Putting it together

```rs
let text = "+ 1 2";

let tokens = lex(text);
let parser = parser_new(tokens);
let expr = parser_parse_expr();
let result = eval_expr(expr);
println("result of {} is {}", text, result);
```

Running the code, we should expect console output like the following.
```
result of + 1 2 is 3
```

## Exercises

1. Make a performance benchmark.

