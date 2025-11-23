# Part 3: Pattern Matching, Enums, and How KCL Parses Your Code

You've seen enums and pattern matching in action. Now you'll understand how they work and why they make parsing both elegant and safe. We're going to look at KCL's tokenizer and parser, then build a simplified version that handles basic arithmetic.

## Enums Are Not What You Think

If you come from Python, you've seen `enum.Enum`:

```python
from enum import Enum

class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

color = Color.RED
print(color.value)  # 1
```

That's a named constant. Useful, but limited. Rust enums are algebraic data types (sum types). They can hold data:

```rust
enum Color {
    RGB(u8, u8, u8),
    HSL(f64, f64, f64),
    Named(String),
}

let red = Color::RGB(255, 0, 0);
let blue = Color::Named("blue".to_string());
```

Each variant can have different data. `RGB` holds three bytes. `HSL` holds three floats. `Named` holds a string. This is impossible with Python's Enum.

You can approximate it with classes and inheritance:

```python
class Color:
    pass

class RGB(Color):
    def __init__(self, r, g, b):
        self.r, self.g, self.b = r, g, b

class HSL(Color):
    def __init__(self, h, s, l):
        self.h, self.s, self.l = h, s, l

color = RGB(255, 0, 0)
```

But Python won't force you to handle all cases. Rust will.

## Pattern Matching: Exhaustiveness Checking

In Python, you'd use isinstance:

```python
if isinstance(color, RGB):
    print(f"RGB: {color.r}, {color.g}, {color.b}")
elif isinstance(color, HSL):
    print(f"HSL: {color.h}, {color.s}, {color.l}")
# What if we forget Named? Runtime error waiting to happen.
```

In Rust:

```rust
match color {
    Color::RGB(r, g, b) => println!("RGB: {}, {}, {}", r, g, b),
    Color::HSL(h, s, l) => println!("HSL: {}, {}, {}", h, s, l),
    Color::Named(name) => println!("Named: {}", name),
}
```

The compiler forces you to handle all three cases. If you forget `Named`, the code won't compile. This is exhaustiveness checking. It catches bugs at compile time that Python would only catch at runtime (if you're lucky).

You can use a wildcard for "everything else":

```rust
match color {
    Color::RGB(255, 0, 0) => println!("Pure red"),
    Color::RGB(_, _, _) => println!("Some other RGB"),
    _ => println!("Not RGB"),
}
```

The `_` pattern matches anything. But if you use it, you lose exhaustiveness checking. Use it sparingly.

## KCL's Token Type: An Enum Masterclass

Open `rust/kcl-lib/src/parsing/token/mod.rs` and look at the `Token` enum. Here's a simplified version:

```rust
pub enum Token {
    // Literals
    Number { value: f64, suffix: Option<String> },
    String(String),
    True,
    False,

    // Identifiers and keywords
    Ident(String),
    Fn,
    Let,
    Return,
    If,
    Else,

    // Operators
    Plus,
    Minus,
    Star,
    Slash,
    Pipe,
    Percent,

    // Delimiters
    LeftParen,
    RightParen,
    LeftBrace,
    RightBrace,
    LeftBracket,
    RightBracket,

    // Special
    Eof,
    Unknown(char),
}
```

Some variants are unit-like (`Plus`, `True`). Some hold data (`Number { value, suffix }`, `Ident(String)`). This flexibility makes the token representation both compact and expressive.

In Python, you might use:

```python
@dataclass
class Token:
    type: str
    value: Optional[Any] = None
    suffix: Optional[str] = None

# Usage:
token = Token(type="Number", value=42.0)
token = Token(type="Plus")
```

But you'd need runtime checks to ensure `Plus` doesn't have a value. Rust's type system enforces it at compile time.

## The Tokenizer: From Text to Tokens

Tokenizing (lexing) is the first step in parsing. You take source code and split it into meaningful chunks (tokens). KCL's tokenizer is in `rust/kcl-lib/src/parsing/token/tokeniser.rs`.

Here's a simplified tokenizer for basic arithmetic:

```rust
#[derive(Debug, PartialEq)]
enum Token {
    Number(f64),
    Plus,
    Minus,
    Star,
    Slash,
    LeftParen,
    RightParen,
    Eof,
    Unknown(char),
}

struct Tokenizer<'a> {
    input: &'a str,
    position: usize,
}

impl<'a> Tokenizer<'a> {
    fn new(input: &'a str) -> Self {
        Tokenizer { input, position: 0 }
    }

    fn current_char(&self) -> Option<char> {
        self.input.chars().nth(self.position)
    }

    fn advance(&mut self) {
        self.position += 1;
    }

    fn skip_whitespace(&mut self) {
        while let Some(c) = self.current_char() {
            if c.is_whitespace() {
                self.advance();
            } else {
                break;
            }
        }
    }

    fn read_number(&mut self) -> f64 {
        let start = self.position;
        while let Some(c) = self.current_char() {
            if c.is_numeric() || c == '.' {
                self.advance();
            } else {
                break;
            }
        }
        self.input[start..self.position].parse().unwrap()
    }

    fn next_token(&mut self) -> Token {
        self.skip_whitespace();

        match self.current_char() {
            None => Token::Eof,
            Some(c) if c.is_numeric() => Token::Number(self.read_number()),
            Some('+') => { self.advance(); Token::Plus }
            Some('-') => { self.advance(); Token::Minus }
            Some('*') => { self.advance(); Token::Star }
            Some('/') => { self.advance(); Token::Slash }
            Some('(') => { self.advance(); Token::LeftParen }
            Some(')') => { self.advance(); Token::RightParen }
            Some(c) => { self.advance(); Token::Unknown(c) }
        }
    }
}
```

The `'a` lifetime parameter says "this tokenizer borrows a string for lifetime `'a`." It doesn't own the input, just borrows it. When the tokenizer is done, the input is still available to the caller.

Let's use it:

```rust
fn main() {
    let code = "3 + 4 * (2 - 1)";
    let mut tokenizer = Tokenizer::new(code);

    loop {
        let token = tokenizer.next_token();
        println!("{:?}", token);
        if token == Token::Eof {
            break;
        }
    }
}
```

Output:

```
Number(3.0)
Plus
Number(4.0)
Star
LeftParen
Number(2.0)
Minus
Number(1.0)
RightParen
Eof
```

Clean, simple, and type-safe. The Python equivalent:

```python
def tokenize(code):
    tokens = []
    i = 0
    while i < len(code):
        c = code[i]
        if c.isspace():
            i += 1
        elif c.isdigit():
            start = i
            while i < len(code) and (code[i].isdigit() or code[i] == '.'):
                i += 1
            tokens.append(('Number', float(code[start:i])))
        elif c == '+':
            tokens.append(('Plus', None))
            i += 1
        # ... and so on
    return tokens
```

Python uses tuples or dicts. Rust uses enums. Rust's version is both more efficient (no tuple allocation for unit variants) and more type-safe (can't accidentally create invalid tokens).

## Parser Combinators: KCL's Secret Weapon

KCL uses the `winnow` library for parsing. Parser combinators are functions that parse small parts of the input and combine to parse larger structures.

Here's a simple parser combinator in Rust (without winnow, for educational purposes):

```rust
type ParseResult<'a, T> = Option<(&'a str, T)>;

fn parse_number(input: &str) -> ParseResult<f64> {
    let mut end = 0;
    for c in input.chars() {
        if c.is_numeric() || c == '.' {
            end += 1;
        } else {
            break;
        }
    }

    if end == 0 {
        return None;
    }

    let (num_str, rest) = input.split_at(end);
    let num = num_str.parse().ok()?;
    Some((rest, num))
}

fn parse_plus(input: &str) -> ParseResult<char> {
    if input.starts_with('+') {
        Some((&input[1..], '+'))
    } else {
        None
    }
}
```

Each parser returns `Option<(&str, T)>`: either `Some((remaining_input, parsed_value))` or `None` if parsing failed.

You combine parsers:

```rust
fn parse_addition(input: &str) -> ParseResult<f64> {
    let (input, left) = parse_number(input)?;
    let (input, _) = parse_plus(input)?;
    let (input, right) = parse_number(input)?;
    Some((input, left + right))
}

// Usage:
if let Some((rest, result)) = parse_addition("3+4") {
    println!("Result: {}, remaining: '{}'", result, rest);
}
```

The `?` operator propagates `None` (like `return None` if the result is `None`). This is error handling without exceptions.

KCL's parser uses winnow, which provides combinators like `alt` (try alternatives), `many0` (parse zero or more), `delimited` (parse between delimiters), etc. The parser in `rust/kcl-lib/src/parsing/parser.rs` is 6,214 lines of these combinators.

## How KCL Parses Expressions

Let's look at how KCL parses a binary expression like `3 + 4`. Open `rust/kcl-lib/src/parsing/parser.rs` and search for `binary_expression`. You'll find code like this (simplified):

```rust
fn binary_expression(input: &str) -> ParseResult<Expr> {
    // Parse left side
    let (input, left) = primary_expression(input)?;

    // Parse operator
    let (input, op) = operator(input)?;

    // Parse right side
    let (input, right) = primary_expression(input)?;

    // Build AST node
    let expr = Expr::BinaryExpression(Box::new(BinaryExpression {
        left: Box::new(left),
        operator: op,
        right: Box::new(right),
    }));

    Some((input, expr))
}
```

This is recursive descent parsing. Each parser calls other parsers. `binary_expression` calls `primary_expression` twice (for left and right sides). `primary_expression` might call `binary_expression` again (for nested expressions).

The recursion is why we need `Box<T>`. Without it, the types would be infinite in size.

## Precedence and Associativity

Binary operators have precedence (multiplication before addition) and associativity (left-to-right or right-to-left). KCL's parser handles this with a precedence climbing algorithm.

Here's a simplified version for arithmetic:

```rust
fn parse_expr(input: &str, min_prec: u8) -> ParseResult<Expr> {
    let (mut input, mut left) = parse_primary(input)?;

    loop {
        let Some((rest, op)) = parse_operator(input) else {
            break;
        };

        let prec = operator_precedence(&op);
        if prec < min_prec {
            break;
        }

        let (rest, right) = parse_expr(rest, prec + 1)?;
        left = Expr::BinaryExpression(Box::new(BinaryExpression {
            left: Box::new(left),
            operator: op,
            right: Box::new(right),
        }));
        input = rest;
    }

    Some((input, left))
}

fn operator_precedence(op: &BinaryOperator) -> u8 {
    match op {
        BinaryOperator::Add | BinaryOperator::Sub => 1,
        BinaryOperator::Mul | BinaryOperator::Div => 2,
    }
}
```

This ensures `3 + 4 * 5` is parsed as `3 + (4 * 5)`, not `(3 + 4) * 5`.

In Python, you'd use a library like `lark` or `pyparsing`, or hand-write a similar recursive descent parser. The logic is the same, but Rust's type system catches more errors at compile time.

## Building Your Own Mini-KCL Parser

Let's build a parser that handles basic arithmetic and variables:

```rust
#[derive(Debug)]
enum Expr {
    Number(f64),
    Name(String),
    Binary {
        left: Box<Expr>,
        op: BinOp,
        right: Box<Expr>,
    },
}

#[derive(Debug)]
enum BinOp {
    Add,
    Sub,
    Mul,
    Div,
}

#[derive(Debug)]
struct VarDecl {
    name: String,
    value: Expr,
}

#[derive(Debug)]
enum Stmt {
    VarDecl(VarDecl),
    Expr(Expr),
}

#[derive(Debug)]
struct Program {
    statements: Vec<Stmt>,
}

// Parser implementation (simplified)
fn parse_program(input: &str) -> Result<Program, String> {
    let mut tokenizer = Tokenizer::new(input);
    let mut statements = Vec::new();

    loop {
        let token = tokenizer.peek();
        match token {
            Token::Eof => break,
            Token::Ident(_) if tokenizer.peek_next() == Some(Token::Equal) => {
                statements.push(parse_var_decl(&mut tokenizer)?);
            }
            _ => {
                statements.push(Stmt::Expr(parse_expr(&mut tokenizer)?));
            }
        }
    }

    Ok(Program { statements })
}
```

You'd extend this to handle more operators, function calls, conditionals, etc. KCL's parser does exactly this, but with 100+ different node types.

## Error Recovery: Parsing Doesn't Stop at the First Error

KCL's parser has a clever feature: it continues parsing even after errors. It accumulates errors in a thread-local context and returns both the AST and the error list.

Look at `rust/kcl-lib/src/parsing/parser.rs` for `ParseContext`:

```rust
thread_local! {
    static PARSE_CONTEXT: RefCell<ParseContext> = RefCell::new(ParseContext::default());
}

pub struct ParseContext {
    errors: Vec<CompilationError>,
    warnings: Vec<CompilationError>,
}
```

When a parser encounters an error, it logs it and tries to recover. This gives better error messages: instead of "syntax error at line 5," you get "syntax error at line 5, line 12, and line 18."

In Python parsers, you'd typically stop at the first error. KCL's approach is more user-friendly.

## What You Just Learned

1. Rust enums are algebraic data types (sum types) that can hold different data per variant
2. Pattern matching is exhaustive (compiler enforces handling all cases)
3. Tokenizing converts source code into a stream of tokens
4. Parser combinators are functions that parse and compose
5. Recursive descent parsing builds ASTs by calling parsers recursively
6. Precedence climbing handles operator precedence correctly
7. KCL's parser accumulates errors for better error reporting

## Exercises

1. **Complete the tokenizer**: Add support for identifiers (variable names), equals sign, and semicolons to the simple tokenizer.

2. **Build a calculator**: Write a parser that handles `3 + 4 * 5 / (2 - 1)` and evaluates it to a number. Use precedence climbing.

3. **Add variables**: Extend your parser to handle `x = 5; y = x + 3; y * 2`. You'll need a symbol table (a `HashMap<String, f64>`) to store variable values.

4. **Parse real KCL**: Use KCL's `parse_str` function to parse a complex KCL program from `public/kcl-samples/` and print the AST. Count how many binary expressions are in the program.

5. **Error handling**: Modify your parser to collect multiple errors instead of stopping at the first one. Return both the partial AST and the error list.

## Next Up

In Part 4, we'll cover traits and generics, then look at KCL's type system. You'll understand how KCL tracks types at runtime (unlike Rust's compile-time types), and you'll implement a simple type checker. We'll also cover the `impl` keyword, trait objects, and how KCL uses traits to abstract over different value types.

The type system is where things get interesting.
