
# 2 Lexer

In this chapter I'll propose a lexer for a compiler.

The objective of the lexer is to take text as input and ouput tokens.

## 2.1 Tokens

A token is a piece of our input program, eg. an integer, a plus operator, etc.

Tokens are the result of grouping together relevant characters, but without incorporating structure. For example, if the input text is the characters `1`, then `2`, then ` `, then `+`, the tokens would be an integer token, then a plus operator token. Whitespace is usually filtered out, meaning the lexer will not emit whitespace tokens, it'll just skip over.

Tokens serve 2 purposes. 1) Categorize characters or groups of characters and 2) provide a way to retrieve the text/value the characters represent.

I'll propose this Typescript type for representing tokens:
```ts
type Token = {
    type: string,
    pos: Pos,
    length: number,
    identValue?: string,
    intValue?: number,
    stringValue?: string,
};

type Pos = {
    index: number,
    line: number,
    col: number,
};
```

The `type` field contians the *token type*, eg. `"int"`, `"+"`, etc. The `pos` field is used primarily for error messages and with `length` for retrieving the raw text. The `identValue` and 2 other value fields contain the values of tokens when relevant, for example with integers we're also interested in the value.

## 2.2 Transformation iterator

The lexer I propose is implemented as a 'transformation' of one iterator, a text character iterator, into a token iterator.

Let's start with the lexer code from chapter 1.

```ts
function lex(text: string): Token[] {
    let index = 0;
    const tokens: Token[] = [];
    while (index < text.length) {
        // ...
    }
    return tokens;
}
```

Because it'll make the lexer's code a bit cleaner, I'll refactor the lexer using the iterator pattern.

```ts
class Lexer {
    private index = 0;

    public constructor (private text: string) {}

    public next(): Token | null { /*...*/ }
}
```

The difference is that before when calling `lex` you would get an array of all tokens, now you instanciate a `Lexer` and call the `.next()` function for each token, until the end-of-file (EOF).

I'll add 3 functions for iterating through characters of the text:

```ts
class Lexer {
    // ...
    private step() { /*...*/ }
    private done(): bool { return this.index >= this.text.length; }
    private current(): string { return this.text[this.index]; }
    // ...
}
```

The `.done()` is the same as in the while loop condition above. The `.current()` is instead of indexing directly into the text. `.step()` isn't implemented yet, that's because I also want line and column number in the tokens. To do this I'll add `line` and `column` as properties on `Lexer` in addition to `index`:

```ts
class Lexer {
    private index = 0;
    private line = 1;
    private col = 1;
    // ...
}
```

Line and column numbers both start at one. Text in a file goes from left to right, top to bottom in that order. The column number should therefore increase when we step, until we step over a newline, then the line number should be incremented and column number reset to `1`. The index should just increment as we step. In case the lexer is done (hit the end of the file, usually called "EOF"), we won't increment anything.

```ts
class Lexer {
    // ...
    private step() {
        if (this.done())
            return;
        if (this.current() === "\n") {
            this.line += 1;
            this.col = 1;
        } else {
            this.col += 1;
        }
        this.index += 1;
    }
    // ...
}
```

I'll also propose a method for optaining the current position:

```ts
class Lexer {
    // ...
    private pos(): Pos {
        return {
            index: this.index,
            line: this.line,
            col: this.col,
        };
    }
    // ...
}
```

And a method for creating valueless tokens:

```ts
class Lexer {
    // ...
    private token(type: string, pos: Pos): Token {
        return {
            index: this.index,
            line: this.line,
            col: this.col,
        };
    }
    // ...
}
```

And a method for testing/matching the `.current()` against a regex pattern or string:

```ts
class Lexer {
    // ...
    private test(pattern: RegExp | string): Token {
        if (typeof pattern === "string")
            return this.current === pattern;
        else
            return pattern.test(this.current);
    }
    // ...
}
```

Lastly I'll make `.next()` return `null` when done and handle whenever an unrecognized/illegal character is reached.

```ts
class Lexer {
    // ...
    public next(): Token | null {
        if (this.done())
            return null;
        const pos = this.pos();
        // ...
        console.error(`Lexer: illegal character '${this.current()}' at ${pos.line}:${pos.col}`);
        this.step();
        return next();
    }
    // ...
}
```

When we've checked, we haven't hit EOF yet, we get the current position. In `.next()` we should return after each valid token. This means, when we hit the end, we could not make any token, therefore we report an error. After reporting, we step over the character, essentially ignoring it, and call `.next()` recursively to start again. We don't need to stop the compilation, just because we hit an invalid character.

The scaffolding is now complete.

## 2.3 Skipping whitespace

We don't need to know anything about whitespace, so we'll skip over it without making a token. This is what I'll propose adding to `.next()` to make that happen:

```ts
class Lexer {
    // ...
    public next(): Token | null {
        if (this.done())
            return null;
        const pos = this.pos();
        if (this.test(/[ \t\n]/)) {
            while (!this.done() && this.test(/[ \t\n]/))
                this.step();
            return next();
        }
        // ...
        console.error(
            `Lexer: illegal character '${this.current()}'`
                + ` at ${pos.line}:${pos.col}`,
        );
        this.step();
        return next();
    }
    // ...
}
```

We test if the current character matches the regex `[ \t\n]`, meaning space, tab and newline characters. And then we `.step()` over all the whitespace characters. Then we return a call to `.next()`, instead of just continuing, because we need to check `.done()` again.

## 2.4 Comments

Same as in Python, Bash and so many others, comments in this language is a hashtag `#` and then all the characters afterwards is ignored, until a newline `\n` is reached.

```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test("#")) {
            while (!this.done() && !this.test("\n"))
                this.step();
            return next();
        }
        // ...
    }
    // ...
}
```

## 2.5 Identifiers and keywords

Identifiers, also called names or symbols, are what we use to refer to functions, variables, etc. Keywords are the special words we use in specialt syntax, such as `if`-statements.

To lex identifiers and keywords, we'll look for identifier characters, find all the characters in an identifier, save the text value.

Identifers can start with a letter or an underscore `[a-zA-Z_]`, but not numbers `[0-9]`, because that would be hard to implement. Every character after the first may also contain numbers, ie. `[a-zA-Z_0-9]`.

Lastly, we check if the identifier is in fact not an identifier, but one of the hardcoded keywords.

```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test(/[a-zA-Z_]/)) {
            let value = "";
            while (!this.done() && this.test(/[a-zA-Z0-9_]/)) {
                value += this.current();
                this.step();
            }
            switch (value) {
                case "if":
                    return this.token("if", pos);
                case "else":
                    return this.token("else", pos);
                default:
                    return { ...this.token("ident", pos), identValue: value };
            }
        }
        // ...
    }
    // ...
}
```

Again we use the if-while pattern to match multiple characters. Each character is appended to the `value` variable, so that we have the entire value at the end. If then, the value matches one of the keywords, a keyword specific token is returned, else an `ident` token is returned and the `identValue` field is added on.

## 2.6 Integers

Integers, just like identifers, are sequences of characters. Unlike identifiers, there are no keywords (keynumbers) we need to be aware of. An integer, in this language, is a sequence consisting of and starting with any number characters `[0-9]`. (Sidenote: In most other languages, a base 10 integer is __either__ a `0` character or a sequence starting with any number __except zero__ `[1-9]`, after the first character any number is allowed.)

After matching the integer, we also parse the string representing a number into the number value itself.

```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test(/[0-9]/)) {
            let textValue = "";
            while (!this.done() && this.test(/[0-9]/)) {
                textValue += this.current();
                this.step();
            }
            return { ...this.token("int", pos), intValue: parseInt(textValue) };
        }
        // ...
    }
    // ...
}
```

This implementation is a bit naive, but I've prioritized simplicity.

## 2.7 Strings

Strings, like identifiers and integers, are sequences of characters represent a value. Like integers, we don't need to check for keywords.

Strings can contain special/escaped characters such as `\"`, `\\`, `\n`. For `\\` and `\"`, the resulting value should just contain the character after the backslash `\`. For `\n`, `\t` and `\0` the resulting value should contain a newline, tab and null-byte character respectively. The implementation will make this transformation while matching the string.

The resulting value also does not contain the surrounding double quotes `" "`

```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test("\"")) {
            this.step();
            let value = "";
            while (!this.done() && !this.test("\"")) {
                if (this.test("\\")) {
                    this.step();
                    if (this.done())
                        break;
                    value += {
                        "n": "\n",
                        "t": "\t",
                        "0": "\0",
                    }[this.current()] ?? this.current();
                } else {
                    value += this.current();
                }
                this.step();
            }
            if (this.done() || !this.test("\"")) {
                console.error(
                    `Lexer: unclosed/malformed string`
                        + ` at ${pos.line}:${pos.col}`,
                );
                return this.token("error", pos);
            }
            this.step();
            return { ...this.token("string", pos), stringValue: value };
        }
        // ...
    }
    // ...
}
```

A string starts with a double quote `"`. This character is ignored. We then look at every character up until an __unescaped__ double quote `"`, meaning a string may contain escaped double quote characters `\"`. When the loop is done, we check whether we've hit a closing double quote `"` or if we've reached EOF. If we've reached EOF, we emit an error and return a special `error` token. If we've reached the enclosing quote `"`, we ignore and step over it and return a string token.

While looking at each character, we check if it's a backslash `\`. If so, we ignore the escaping backslash and handle the next character like either specialt characters, eg. `n` should be a newline character, or just an escaped character, eg. an escaped double quote.

## 2.8 Static tokens

Operators and punctionation such as plus `+`, left curlybrace `{` and semicolon `;` are single character static tokens. Operators such as double equal `==` are multi character static tokens.

```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test(/[\+\{\};]/)) {
            const first = this.current();
            this.step();
            if (first === "=" && !this.done() && this.test("=")) {
                this.step();
                return this.token("==", pos);
            }
            return this.token(first, pos);
        }
        // ...
    }
    // ...
}
```

The clause checks for any one of the characters. For single character tokens, it saves the character for use as the token type. Then it `.step()`s and returns a token with the character as the token type. For multi character tokens, we check the first character again for specific characters, and then match another token if necessary.

Note: The reason `+`, `{`, `}` are escaped, is because they are special regex characters. In Javascript all non-letters can safely be escaped and all letters are safe to use.

The above code can also be writte like this:
```ts
class Lexer {
    // ...
    public next(): Token | null {
        // ...
        if (this.test("=")) {
            this.step()
            if (!this.done() && this.test("=")) {
                this.step();
                return this.token("==", pos);
            }
            return this.token("=", pos);
        }
        if (this.test("{")) {
            this.step();
            return this.token("{", pos);
        }
        // ...
    }
    // ...
}
```

The former is shorter the latter is more readable.

## 2.9 Testing

To test the lexer, give it some text and observe the result and errors.

```ts
const text = `
    a1 123 +
    # comment
    "hello"
    "escaped\"\nnewline"
`;
const lexer = new Lexer(text);
let token = lexer.next();
while (token !== null) {
    console.log(`Lexed ${token}`);
    token = lexer.next();
}
```

## Exercises

1. Implement the operators: `-`, `*`, `/`, `(`, `)`, `.`, `,`, `;`, `[`, `]`, `!=`, `<`, `>`, `<=` and `>=`.
2. Implement the keywords: `true`, `false`, `null`, `or`, `and`, `not`, `loop`, `break`, `let`, `fn` and `return`.
3. \* Implement single line comments using `//` and multiline comments using `\*` and `*\` (\*\* extra points if multiline comments can be nested, eg. `/* ... /* ... */ ... */`).
4. \* Reimplement integers such that integers are either `0` or start with `[1-9]`.
5. \* Implement hexadecimal (hex) integers, eg. `0xAA15`, using base 16.

