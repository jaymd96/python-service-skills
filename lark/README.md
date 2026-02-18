# Lark - A Modern Parsing Library for Python

## Overview

Lark is a parsing library for Python that builds parsers from grammars defined in an EBNF-like notation. It treats **grammar as data** rather than code, allowing you to define a complete language grammar as a multi-line string and have Lark generate a fully functional parser from it at runtime (or ahead of time via standalone mode).

Lark supports two parsing algorithms:

- **Earley** (default) -- a general parser that can handle any context-free grammar, including ambiguous ones.
- **LALR(1)** -- a high-performance table-based parser that works with a large subset of context-free grammars and is significantly faster than Earley.

Key design principles:

- Grammars are readable, declarative, and separate from your processing logic.
- Automatic tree construction means you never have to manually build an AST.
- The `Transformer` pattern provides a clean, visitor-like interface for processing parse trees.
- Built-in support for common patterns via `%import common` (numbers, strings, whitespace, etc.).

**Latest stable version:** 1.2.2 (released on PyPI). Lark requires Python 3.8 or later.

> **Note on version:** The version listed was accurate as of early 2025. Check [https://pypi.org/project/lark/](https://pypi.org/project/lark/) for the most current release.

---

## Installation

```bash
# Standard installation
pip install lark

# With the regex module for improved Unicode support and performance
pip install lark[regex]

# With the atomic_cache extra for thread-safe caching of LALR tables
pip install lark[atomic_cache]
```

The package name on PyPI is `lark`. An older package named `lark-parser` exists but is deprecated; always use `lark`.

---

## Core Concepts

### Grammar Definition Syntax

Lark grammars use an EBNF-like notation. A grammar is a string containing **rules** (lowercase names) and **terminals** (UPPERCASE names):

```python
from lark import Lark

grammar = r"""
    start: greeting NAME

    greeting: "hello" | "hi" | "hey"

    NAME: /[A-Za-z]+/

    %import common.WS
    %ignore WS
"""

parser = Lark(grammar)
tree = parser.parse("hello World")
print(tree.pretty())
```

Output:

```
start
  greeting
  World
```

### Terminals (UPPERCASE) vs Rules (lowercase)

This distinction is fundamental in Lark:

| Aspect | Terminals (UPPERCASE) | Rules (lowercase) |
|---|---|---|
| Naming convention | `NAME`, `NUMBER`, `STRING` | `start`, `expr`, `statement` |
| What they match | Atomic tokens (regex or literal) | Compositions of terminals and other rules |
| In the parse tree | Become `Token` objects (leaf nodes) | Become `Tree` objects (branch nodes) |
| Defined with | `NAME: /regex/` or `NAME: "literal"` | `rule: item1 item2` |
| Processing order | Lexed first (tokenization phase) | Parsed second (parsing phase) |

### Creating Parsers with `Lark()`

```python
from lark import Lark

# From a grammar string
parser = Lark(grammar_string, parser="lalr", start="start")

# From a grammar file
parser = Lark.open("my_grammar.lark", parser="earley")

# Key constructor parameters:
#   grammar       -- the grammar string (or use Lark.open for files)
#   parser        -- "earley" (default) or "lalr"
#   lexer         -- "auto" (default), "standard", "contextual", "dynamic"
#   start         -- start rule name (default: "start")
#   ambiguity     -- "explicit" (Earley only) to keep all parse trees
#   transformer   -- a Transformer to apply during parsing (LALR only)
#   propagate_positions -- True to track line/column info on Tree nodes
#   maybe_placeholders  -- True to insert None for unfilled ?optionals (LALR)
#   regex         -- True to use the `regex` module instead of `re`
#   keep_all_tokens     -- True to keep all tokens including literals
```

### Parser Types

#### Earley Parser (`parser="earley"`)

- Handles **any** context-free grammar, including ambiguous ones.
- Default parser.
- Slower than LALR, especially for large inputs.
- Supports `ambiguity="explicit"` to return all possible parse trees.
- Uses a dynamic lexer by default (tokens can depend on parse context).

```python
parser = Lark(grammar, parser="earley", ambiguity="explicit")
```

#### LALR(1) Parser (`parser="lalr"`)

- Significantly **faster** than Earley (roughly 4-10x).
- Only handles LALR(1) grammars (no ambiguity allowed).
- Supports **contextual lexing**, which resolves many terminal conflicts.
- Supports **interactive parsing** for incremental/streaming use cases.
- Can generate **standalone parsers** (no Lark dependency at runtime).

```python
parser = Lark(grammar, parser="lalr")
```

### `Tree` and `Token` -- Parse Tree Structure

Every call to `parser.parse(text)` returns a `Tree` object:

```python
from lark import Tree, Token

tree = parser.parse("1 + 2")

# Tree attributes
tree.data       # str: the rule name, e.g. "expr"
tree.children   # list: child Tree and Token objects
tree.meta       # Meta: line/column info (if propagate_positions=True)

# Token attributes (Token is a subclass of str)
token = tree.children[0]
str(token)      # the matched text
token.type      # str: the terminal name, e.g. "NUMBER"
token.line      # int: line number (1-based)
token.column    # int: column number (1-based)
token.end_line  # int: end line number
token.end_column # int: end column number

# Tree utilities
tree.pretty()           # human-readable tree string
tree.find_data("rule")  # iterate over subtrees with data == "rule"
tree.find_pred(func)    # iterate over subtrees matching a predicate
tree.iter_subtrees()    # iterate bottom-up over all subtrees
tree.iter_subtrees_topdown()  # iterate top-down
```

Example tree traversal:

```python
from lark import Lark

grammar = r"""
    start: add
    add: NUMBER "+" NUMBER
    NUMBER: /\d+/
    %import common.WS
    %ignore WS
"""

parser = Lark(grammar)
tree = parser.parse("3 + 5")

print(type(tree))              # <class 'lark.tree.Tree'>
print(tree.data)               # "start"
print(type(tree.children[0]))  # <class 'lark.tree.Tree'>

add_node = tree.children[0]
print(add_node.data)           # "add"
print(type(add_node.children[0]))  # <class 'lark.lexer.Token'>
print(add_node.children[0])        # "3"
print(add_node.children[0].type)   # "NUMBER"
```

---

## Grammar Syntax Reference

### Rules

Rules define how non-terminals are composed. They use lowercase names:

```lark
// Basic rule
start: expr

// Sequence -- match item1 THEN item2
expr: item1 item2

// Alternatives -- match item1 OR item2
expr: item1 | item2

// Multi-line alternatives
expr: item1
    | item2
    | item3

// Grouping with parentheses
expr: (item1 | item2) item3

// Optional (zero or one)
expr: item1 item2?

// Zero or more
expr: item1 item2*

// One or more
expr: item1 item2+

// Named alternatives (aliases) -- each branch produces a differently-named tree
expr: item1 "+" item2   -> add
    | item1 "-" item2   -> sub
    | item1              -> identity
```

### Terminals

Terminals match atomic tokens. They use UPPERCASE names:

```lark
// Regex terminal
NAME: /[a-zA-Z_][a-zA-Z0-9_]*/

// Literal string terminal
PLUS: "+"
ARROW: "->"

// Case-insensitive literal
SELECT: "select"i

// Concatenation in terminals
FULL_NAME: /[A-Z]/ /[a-z]+/

// Alternation in terminals
SIGN: "+" | "-"

// Repetition in terminals
DIGITS: /[0-9]/+
```

### Rule Modifiers

```lark
// ? -- Inline rule: if the rule has a single child, replace the tree node
//      with that child (flattens the tree)
?expr: term
     | expr "+" term   -> add

// ! -- Keep all tokens: prevents automatic filtering of literals
!expr: term "+" term

// Without !, the "+" token would be removed from the tree.
// With !, the "+" token is preserved as a child.
```

### Terminal Modifiers

```lark
// _ prefix -- Anonymous/filtered terminal: won't appear in the tree
_SEPARATOR: /[,;]/

// Inline regex (anonymous) -- string literals in rules are auto-terminals
expr: NUMBER "+" NUMBER
// The "+" becomes an anonymous terminal and is filtered from children
```

### Repeat Modifier `~`

```lark
// Exact repetition
four_digits: DIGIT~4          // exactly 4 DIGIT tokens

// Range repetition
digits: DIGIT~2..6            // between 2 and 6 DIGIT tokens

DIGIT: /[0-9]/
```

### Priorities

Priorities resolve ambiguities. Higher numbers mean higher priority:

```lark
// Rule priority (after the rule name, before the colon)
expr.2: NUMBER "+" NUMBER     // priority 2 (higher, preferred)
expr.1: NUMBER "-" NUMBER     // priority 1

// Terminal priority (after the terminal name, before the colon)
KEYWORD.2: "if" | "else" | "while"
NAME.1: /[a-zA-Z_]\w*/

// Negative priorities are allowed
low_priority.-1: something
```

Terminal priority determines which terminal wins when multiple terminals match the same text. Rule priority influences which derivation the parser prefers.

### Imports

Lark has a powerful import system:

```lark
// Import a single terminal or rule from another grammar
%import common.NUMBER
%import common.WS
%import common.ESCAPED_STRING

// Import with rename
%import common.NUMBER as NUM
%import common.WORD as IDENTIFIER

// Import everything from a module
%import common (NUMBER, WORD, WS, NEWLINE)

// Import from a relative path
%import .my_other_grammar.some_rule

// Import from an absolute path
%import /path/to/grammar.some_rule
```

### The Common Grammar (`%import common`)

Lark ships with a `common` grammar containing frequently used terminals:

```lark
// Numbers
%import common.NUMBER          // /\d+(\.\d+)?([eE][+-]?\d+)?/  (int or float)
%import common.INT             // /[0-9]+/
%import common.SIGNED_INT      // /[+-]?[0-9]+/
%import common.SIGNED_NUMBER   // signed int or float
%import common.DECIMAL         // /\d+\.\d+/
%import common.HEXDIGIT        // /[0-9a-fA-F]/
%import common.FLOAT           // /\d+\.\d*([eE][+-]?\d+)?/

// Strings
%import common.ESCAPED_STRING  // "..." with standard escape sequences
%import common.SH_COMMENT      // shell-style comments: # ...

// Words / Identifiers
%import common.WORD            // /\w+/
%import common.LETTER          // /[a-zA-Z]/
%import common.CNAME           // C-style identifier: /[a-zA-Z_]\w*/
%import common.UCASE_LETTER    // /[A-Z]/
%import common.LCASE_LETTER    // /[a-z]/

// Whitespace
%import common.WS              // /(\s)+/  (spaces, tabs, newlines)
%import common.WS_INLINE       // /[\t \f]+/  (spaces and tabs only, no newlines)
%import common.NEWLINE          // /\n/ or /\r\n/

// Ignore directive -- tells the lexer to skip matches
%ignore WS                     // ignore all whitespace
%ignore /[\t \f]+/             // ignore inline whitespace (regex form)
%ignore SH_COMMENT             // ignore shell comments
```

### Directives

```lark
// Ignore -- skip matching text during lexing
%ignore WS
%ignore /\/\/.*/               // ignore C++ style line comments
%ignore /\/\*[\s\S]*?\*\//    // ignore C style block comments

// Declare -- forward-declare terminals (useful for contextual lexer)
%declare NAME TYPE

// Override -- redefine an imported rule or terminal
%override NAME: /[a-zA-Z]\w*/
```

---

## Transformers

### The `Transformer` Class

A `Transformer` walks the parse tree bottom-up and calls a method for each rule it encounters. The method receives the already-transformed children:

```python
from lark import Lark, Transformer, v_args

grammar = r"""
    start: expr
    expr: term ((ADD | SUB) term)*
    term: NUMBER
    ADD: "+"
    SUB: "-"
    NUMBER: /\d+/
    %import common.WS
    %ignore WS
"""

class MyTransformer(Transformer):
    def start(self, args):
        return args[0]

    def expr(self, args):
        # args contains transformed children
        return args  # or some computation

    def term(self, args):
        return int(args[0])

    # Terminal handlers (UPPERCASE method names)
    def NUMBER(self, token):
        return int(token)

parser = Lark(grammar)
tree = parser.parse("3 + 5")
result = MyTransformer().transform(tree)
```

### `@v_args(inline=True)` for Cleaner Callbacks

Without `inline`, every callback receives a single list of children. With `@v_args(inline=True)`, children are unpacked as positional arguments:

```python
from lark import Transformer, v_args

# Applied to the entire class
@v_args(inline=True)
class CalcTransformer(Transformer):
    from operator import add, sub, mul, truediv as div

    def number(self, value):
        return float(value)

    # Or applied to individual methods
    @v_args(inline=False)
    def start(self, args):
        return args[0]
```

### Bottom-Up Processing (Transformer)

`Transformer` processes the tree **bottom-up** (post-order): leaf nodes and deeper subtrees are transformed first, then their parents. This means every callback receives already-transformed children.

```python
class EvalExpr(Transformer):
    def number(self, args):
        return float(args[0])       # leaf: Token -> float

    def add(self, args):
        return args[0] + args[1]    # children already floats

    def mul(self, args):
        return args[0] * args[1]    # children already floats
```

### Top-Down Processing with `Interpreter`

`Interpreter` processes the tree **top-down** and does not transform children automatically. You call `self.visit(child)` or `self.visit_children(tree)` explicitly, giving you full control over evaluation order and context passing:

```python
from lark import Interpreter

class MyInterpreter(Interpreter):
    def __init__(self):
        self.env = {}

    def start(self, tree):
        for child in tree.children:
            self.visit(child)

    def assignment(self, tree):
        name = str(tree.children[0])
        value = self.visit(tree.children[1])
        self.env[name] = value
        return value

    def if_stmt(self, tree):
        cond = self.visit(tree.children[0])
        if cond:
            return self.visit(tree.children[1])  # then branch
        elif len(tree.children) > 2:
            return self.visit(tree.children[2])   # else branch

    def number(self, tree):
        return float(tree.children[0])
```

### Additional Transformer Patterns

```python
from lark import Transformer, Discard, v_args

class AdvancedTransformer(Transformer):
    # Return Discard to remove a subtree entirely
    def comment(self, args):
        raise Discard()

    # __default__ handles any rule without a specific method
    def __default__(self, data, children, meta):
        return children

    # __default_token__ handles any terminal without a specific method
    def __default_token__(self, token):
        return token

# Inline transformer with LALR for maximum performance
# (transforms during parsing, no intermediate tree built)
parser = Lark(grammar, parser="lalr", transformer=CalcTransformer())
result = parser.parse("1 + 2 * 3")  # returns the computed value directly
```

---

## Advanced Features

### Ambiguity Handling

When a grammar is ambiguous (multiple valid parse trees), Earley can return all of them:

```python
grammar = r"""
    start: expr
    expr: expr "+" expr
        | NUMBER
    NUMBER: /\d+/
    %import common.WS
    %ignore WS
"""

parser = Lark(grammar, parser="earley", ambiguity="explicit")
trees = parser.parse("1 + 2 + 3")

# With ambiguity="explicit", the result may contain _ambig nodes
# representing points where multiple derivations exist.
# trees.find_data("_ambig") finds them.
```

### Contextual Lexing

The contextual lexer is an LALR-only feature that resolves terminal ambiguities by considering which terminals are valid at each parser state:

```python
# Automatically enabled with LALR when needed
parser = Lark(grammar, parser="lalr", lexer="contextual")

# This allows terminals to overlap in their patterns as long as
# they don't appear in the same parser context.
```

### Standalone Parser Generation

Generate a standalone Python parser file that has **no dependency on Lark** at runtime:

```bash
# Generate standalone parser
python -m lark.tools.standalone my_grammar.lark > my_parser.py
```

```python
# Using the generated standalone parser (no lark import needed)
from my_parser import Lark_StandAlone, Transformer, v_args

parser = Lark_StandAlone()
tree = parser.parse(input_text)

# You can also pass a transformer
parser = Lark_StandAlone(transformer=MyTransformer())
```

This is ideal for distribution -- your users do not need Lark installed.

### Custom Lexer

You can provide a custom lexer class for specialized tokenization needs:

```python
from lark import Lark, Token

class MyCustomLexer:
    """A custom lexer must implement __init__ and lex."""

    def __init__(self, lexer_conf):
        # lexer_conf contains terminal definitions and configuration
        self.lexer_conf = lexer_conf

    def lex(self, data):
        # Must yield Token objects
        # Token(terminal_name, value, line=..., column=...)
        for i, char in enumerate(data):
            if char.isdigit():
                yield Token("NUMBER", char)
            elif char == "+":
                yield Token("PLUS", char)

parser = Lark(grammar, parser="lalr", lexer=MyCustomLexer)
```

### Interactive Parsing (LALR only)

Interactive parsing lets you feed tokens incrementally and inspect parser state:

```python
from lark import Lark

parser = Lark(grammar, parser="lalr")
interactive = parser.parse_interactive()

# Feed tokens one at a time
for token in lexer_output:
    interactive.feed_token(token)

# Check what tokens are accepted next
accepted = interactive.accepts()

# Useful for:
# - Tab completion / auto-suggest
# - Streaming input
# - Syntax highlighting
# - Partial parsing
```

### Error Recovery

Lark provides mechanisms for handling and recovering from parse errors:

```python
from lark import Lark, UnexpectedInput, UnexpectedCharacters, UnexpectedToken

parser = Lark(grammar, parser="lalr")

try:
    tree = parser.parse(text)
except UnexpectedCharacters as e:
    # Lexer could not match any terminal at this position
    print(f"Unexpected character at line {e.line}, column {e.column}")
    print(f"Allowed characters: {e.allowed}")  # set of expected terminals

except UnexpectedToken as e:
    # Parser received a token it did not expect
    print(f"Unexpected token {e.token!r} at line {e.line}, column {e.column}")
    print(f"Expected one of: {e.expected}")  # set of expected terminal names
    print(f"Token type: {e.token.type}")

except UnexpectedInput as e:
    # Base class for all parse errors
    # Use match_examples for user-friendly error messages
    pass
```

#### Pattern-Based Error Messages with `match_examples`

```python
def parse_with_nice_errors(parser, text):
    try:
        return parser.parse(text)
    except UnexpectedInput as e:
        # Define examples of common mistakes and their error messages
        examples = {
            'Missing closing parenthesis': [
                '(1 + 2',
                '(3 * (4 + 5)',
            ],
            'Missing operator': [
                '1 2',
                '3 4 5',
            ],
            'Unbalanced brackets': [
                '[1, 2)',
                '(1, 2]',
            ],
        }
        label = e.match_examples(parser.parse, examples)
        if label:
            raise SyntaxError(label) from e
        raise SyntaxError("Syntax error") from e
```

#### LALR Puppet-Based Error Recovery

```python
from lark import Token

# Using on_error with interactive parsing for error recovery
parser = Lark(grammar, parser="lalr")

def on_error(e):
    # e is an UnexpectedToken
    # Return True to retry after modifying state, False to raise
    if e.token.type == "$END" and "SEMICOLON" in e.expected:
        # Insert a missing semicolon
        e.interactive_parser.feed_token(Token("SEMICOLON", ";"))
        return True
    return False

tree = parser.parse(text, on_error=on_error)
```

### Propagating Source Positions

```python
parser = Lark(grammar, propagate_positions=True)
tree = parser.parse(text)

# Now every Tree node has position information
node = tree.children[0]
node.meta.line          # start line (1-based)
node.meta.column        # start column (1-based)
node.meta.end_line      # end line
node.meta.end_column    # end column
node.meta.start_pos     # character offset from beginning
node.meta.end_pos       # end character offset
```

---

## Gotchas and Common Mistakes

### 1. Earley vs LALR Tradeoffs

| | Earley | LALR(1) |
|---|---|---|
| Speed | Slower (O(n^3) worst case) | Fast (linear time) |
| Grammar restrictions | None -- any CFG | Must be LALR(1) |
| Ambiguity | Can handle and enumerate | Raises conflict errors |
| Contextual lexing | No | Yes |
| Standalone generation | No | Yes |
| Interactive parsing | No | Yes |
| Inline transformers | No | Yes |

**Recommendation:** Start with LALR for performance. Switch to Earley only if you have an ambiguous grammar or one that is not LALR(1).

### 2. Whitespace Handling

Whitespace is **not** automatically ignored. You must declare it explicitly:

```lark
// WRONG -- will fail on "1 + 2" (spaces not handled)
start: NUMBER "+" NUMBER
NUMBER: /\d+/

// CORRECT -- explicitly ignore whitespace
start: NUMBER "+" NUMBER
NUMBER: /\d+/
%import common.WS
%ignore WS
```

Understanding whitespace terminals:

- `WS` matches spaces, tabs, **and newlines** -- use when whitespace including line breaks is insignificant.
- `WS_INLINE` matches spaces and tabs **only** -- use when newlines are meaningful (e.g., line-oriented languages).
- For languages where indentation matters, you need a custom postlexer (e.g., `IndentIndenter`).

```lark
// Line-oriented language where newlines matter
%import common.WS_INLINE
%import common.NEWLINE
%ignore WS_INLINE

start: line (NEWLINE line)*
line: statement
```

### 3. Priority Conflicts

When two terminals match the same text, the one with higher priority wins. If priorities are equal, the **longer match** wins. If both the same length, the one defined **earlier** in the grammar wins.

Common pitfall -- keywords vs identifiers:

```lark
// WRONG -- "if" matches both KEYWORD and NAME equally
KEYWORD: "if" | "else" | "while"
NAME: /[a-z]+/

// CORRECT -- give keywords higher priority
KEYWORD.2: "if" | "else" | "while"
NAME.1: /[a-z]+/

// ALSO CORRECT -- use negative priority on the catch-all
KEYWORD: "if" | "else" | "while"
NAME.-1: /[a-z]+/
```

### 4. Performance with Large Grammars

- Use `parser="lalr"` for production workloads; it is dramatically faster.
- Use the **standalone parser** to avoid grammar compilation overhead at startup.
- For LALR, pass `transformer=MyTransformer()` to the `Lark` constructor to transform during parsing (avoids building the full tree).
- Cache the Lark instance; do not re-create it for every parse call. Grammar compilation is expensive; parsing is cheap.

```python
# WRONG -- recompiles grammar every call
def parse_input(text):
    parser = Lark(grammar, parser="lalr")
    return parser.parse(text)

# CORRECT -- compile once, parse many
parser = Lark(grammar, parser="lalr")

def parse_input(text):
    return parser.parse(text)
```

### 5. Tree vs Token Confusion

A very common mistake is treating `Token` objects (leaf nodes) like `Tree` objects or vice versa:

```python
# Token is a subclass of str
token = tree.children[0]  # might be Token or Tree!

# Always check the type if unsure
from lark import Tree, Token

for child in tree.children:
    if isinstance(child, Tree):
        print(f"Subtree: {child.data}")
    elif isinstance(child, Token):
        print(f"Token: {child.type} = {child}")
```

### 6. Anonymous Terminals (String Literals) Are Filtered

When you write `"+"` or `"if"` directly in a rule, Lark creates an anonymous terminal for it. By default, anonymous terminals are **excluded** from `tree.children`:

```lark
// The "+" will NOT appear in tree.children
expr: NUMBER "+" NUMBER
```

```python
tree = parser.parse("3 + 5")
# tree.children == [Token('NUMBER', '3'), Token('NUMBER', '5')]
# The "+" is gone!
```

To keep them, use `!` on the rule or `keep_all_tokens=True` on the Lark constructor:

```lark
!expr: NUMBER "+" NUMBER
```

### 7. The `?` Rule Modifier Can Surprise You

The `?` modifier inlines a rule when it has exactly one child. This is useful for eliminating wrapper nodes but can change your tree structure unexpectedly:

```lark
?atom: NUMBER | "(" expr ")"
```

If `atom` matches just `NUMBER`, the `atom` tree node disappears and `NUMBER` is placed directly into the parent. But if `atom` matches `"(" expr ")"`, a full `atom` node is created. Design your Transformers accordingly.

### 8. Start Rule Must Be Named `start` (by default)

If your grammar's entry point is not named `start`, you must specify it:

```python
# If your grammar starts with "program:" instead of "start:"
parser = Lark(grammar, start="program")
```

---

## Complete Code Examples

### Example 1: Simple Calculator

A calculator supporting `+`, `-`, `*`, `/`, parentheses, and unary negation:

```python
from lark import Lark, Transformer, v_args

calc_grammar = r"""
    ?start: expr

    ?expr: term
         | expr "+" term   -> add
         | expr "-" term   -> sub

    ?term: factor
         | term "*" factor -> mul
         | term "/" factor -> div

    ?factor: atom
           | "-" atom      -> neg

    ?atom: NUMBER          -> number
         | "(" expr ")"

    NUMBER: /\d+(\.\d+)?/

    %import common.WS_INLINE
    %ignore WS_INLINE
"""

@v_args(inline=True)
class CalculateTree(Transformer):
    def number(self, token):
        return float(token)

    def add(self, left, right):
        return left + right

    def sub(self, left, right):
        return left - right

    def mul(self, left, right):
        return left * right

    def div(self, left, right):
        return left / right

    def neg(self, value):
        return -value

# Create parser with inline transformer for maximum performance
calc_parser = Lark(calc_grammar, parser="lalr", transformer=CalculateTree())

def calculate(expression: str) -> float:
    return calc_parser.parse(expression)

# Usage
print(calculate("1 + 2"))             # 3.0
print(calculate("(1 + 2) * 3"))       # 9.0
print(calculate("10 / (2 + 3)"))      # 2.0
print(calculate("-5 + 3"))            # -2.0
print(calculate("2 * (3 + 4) - 1"))  # 13.0
```

### Example 2: JSON Parser

A complete JSON parser demonstrating most grammar features:

```python
from lark import Lark, Transformer, v_args

json_grammar = r"""
    ?value: object
          | array
          | string
          | number
          | "true"             -> true
          | "false"            -> false
          | "null"             -> null

    object: "{" [pair ("," pair)*] "}"
    pair: string ":" value

    array: "[" [value ("," value)*] "]"

    string: ESCAPED_STRING
    number: SIGNED_NUMBER

    %import common.ESCAPED_STRING
    %import common.SIGNED_NUMBER
    %import common.WS
    %ignore WS
"""

class JsonTransformer(Transformer):
    @v_args(inline=True)
    def string(self, s):
        return s[1:-1].replace('\\"', '"')  # strip quotes

    @v_args(inline=True)
    def number(self, n):
        return float(n)

    def array(self, items):
        return list(items)

    def pair(self, args):
        key, value = args
        return (key, value)

    def object(self, items):
        return dict(items)

    def null(self, _):
        return None

    def true(self, _):
        return True

    def false(self, _):
        return False

json_parser = Lark(json_grammar, parser="lalr", transformer=JsonTransformer())

# Usage
data = json_parser.parse('''
{
    "name": "Lark",
    "version": 1.2,
    "features": ["parsing", "transforming", null],
    "active": true,
    "metadata": {
        "author": "Erez Shinan",
        "license": "MIT"
    }
}
''')

print(data)
# {'name': 'Lark', 'version': 1.2, 'features': ['parsing', 'transforming', None],
#  'active': True, 'metadata': {'author': 'Erez Shinan', 'license': 'MIT'}}
```

### Example 3: Custom DSL for Configuration

A DSL that defines typed configuration sections:

```python
from lark import Lark, Transformer, v_args

config_grammar = r"""
    start: section+

    section: "[" NAME "]" NEWLINE setting+

    setting: NAME "=" value NEWLINE

    ?value: STRING        -> string_val
          | NUMBER        -> number_val
          | BOOL          -> bool_val
          | list_val

    list_val: "[" value ("," value)* "]"

    BOOL.2: "true" | "false"
    STRING: /\"[^\"]*\"/
    NAME: /[a-zA-Z_][a-zA-Z0-9_]*/

    %import common.NUMBER
    %import common.NEWLINE
    %import common.WS_INLINE
    %ignore WS_INLINE
"""

@v_args(inline=True)
class ConfigTransformer(Transformer):
    def start(self, *sections):
        config = {}
        for section in sections:
            config.update(section)
        return config

    def section(self, name, *settings):
        section_dict = {}
        for key, val in settings:
            section_dict[key] = val
        return {str(name): section_dict}

    def setting(self, name, value):
        return (str(name), value)

    def string_val(self, s):
        return str(s)[1:-1]  # strip quotes

    def number_val(self, n):
        s = str(n)
        return int(s) if "." not in s else float(s)

    def bool_val(self, b):
        return str(b) == "true"

    @v_args(inline=False)
    def list_val(self, items):
        return items

config_parser = Lark(config_grammar, parser="lalr")
transformer = ConfigTransformer()

config_text = """
[database]
host = "localhost"
port = 5432
ssl = true

[cache]
ttl = 300
backends = ["redis", "memcached"]
enabled = false
"""

tree = config_parser.parse(config_text)
config = transformer.transform(tree)
print(config)
# {
#   'database': {'host': 'localhost', 'port': 5432, 'ssl': True},
#   'cache': {'ttl': 300, 'backends': ['redis', 'memcached'], 'enabled': False}
# }
```

### Example 4: Evaluating Expressions with a Transformer

A more advanced example showing variable assignment and evaluation:

```python
from lark import Lark, Transformer, v_args, Token

expr_grammar = r"""
    start: instruction+

    ?instruction: assignment
                | expr

    assignment: NAME "=" expr

    ?expr: comparison

    ?comparison: arith
               | arith ">" arith   -> gt
               | arith "<" arith   -> lt
               | arith "==" arith  -> eq

    ?arith: term
          | arith "+" term  -> add
          | arith "-" term  -> sub

    ?term: power
         | term "*" power   -> mul
         | term "/" power   -> div
         | term "%" power   -> mod

    ?power: atom
          | atom "**" power -> pow

    ?atom: NUMBER           -> number
         | NAME             -> var
         | "(" expr ")"

    NAME: /[a-zA-Z_]\w*/
    NUMBER: /\d+(\.\d+)?/

    %import common.WS_INLINE
    %import common.NEWLINE
    %ignore WS_INLINE
    %ignore NEWLINE
"""

@v_args(inline=True)
class EvalExpressions(Transformer):
    def __init__(self):
        super().__init__()
        self.variables = {}

    def number(self, token):
        return float(token)

    def var(self, name):
        name = str(name)
        if name not in self.variables:
            raise NameError(f"Undefined variable: {name}")
        return self.variables[name]

    def assignment(self, name, value):
        self.variables[str(name)] = value
        return value

    def add(self, a, b): return a + b
    def sub(self, a, b): return a - b
    def mul(self, a, b): return a * b
    def div(self, a, b): return a / b
    def mod(self, a, b): return a % b
    def pow(self, a, b): return a ** b
    def gt(self, a, b):  return a > b
    def lt(self, a, b):  return a < b
    def eq(self, a, b):  return a == b

    @v_args(inline=False)
    def start(self, results):
        return results[-1]  # return last expression's value

parser = Lark(expr_grammar, parser="lalr")
evaluator = EvalExpressions()

program = """
x = 10
y = 20
result = x * y + 5
result
"""

tree = parser.parse(program)
output = evaluator.transform(tree)
print(output)  # 205.0
print(evaluator.variables)
# {'x': 10.0, 'y': 20.0, 'result': 205.0}
```

### Example 5: Error Handling and Recovery

Robust parsing with detailed error reporting:

```python
from lark import Lark, UnexpectedInput, UnexpectedToken, UnexpectedCharacters

grammar = r"""
    start: statement+

    statement: assignment ";"
    assignment: NAME "=" expr

    ?expr: expr "+" term  -> add
         | expr "-" term  -> sub
         | term

    ?term: term "*" atom  -> mul
         | term "/" atom  -> div
         | atom

    ?atom: NUMBER
         | NAME           -> var
         | "(" expr ")"

    NAME: /[a-zA-Z_]\w*/
    NUMBER: /\d+(\.\d+)?/

    %import common.WS
    %ignore WS
"""

parser = Lark(grammar, parser="lalr", propagate_positions=True)


# --- Approach 1: Simple try/except ---

def parse_simple(text: str):
    try:
        return parser.parse(text)
    except UnexpectedCharacters as e:
        print(f"[Lexer Error] Unexpected character {text[e.pos_in_stream]!r}"
              f" at line {e.line}, column {e.column}")
        print(f"  Expected one of: {e.allowed}")
    except UnexpectedToken as e:
        print(f"[Parser Error] Unexpected token {e.token!r}"
              f" at line {e.line}, column {e.column}")
        print(f"  Expected one of: {e.expected}")
    return None


# --- Approach 2: match_examples for user-friendly messages ---

ERROR_EXAMPLES = {
    "Missing semicolon at end of statement": [
        "x = 1\n",
        "x = 1 + 2\n",
    ],
    "Missing right-hand side of assignment": [
        "x = ;",
    ],
    "Missing closing parenthesis": [
        "x = (1 + 2;",
    ],
}

def parse_with_matching(text: str):
    try:
        return parser.parse(text)
    except UnexpectedInput as e:
        label = e.match_examples(parser.parse, ERROR_EXAMPLES)
        if label:
            raise SyntaxError(
                f"{label} (line {e.line}, column {e.column})"
            ) from None
        raise SyntaxError(
            f"Syntax error at line {e.line}, column {e.column}"
        ) from None


# --- Approach 3: LALR error recovery via on_error ---

from lark import Token

def parse_with_recovery(text: str):
    """Attempt to recover from missing semicolons."""
    def on_error(e):
        # If parser expected a semicolon, insert one
        if "SEMICOLON" in getattr(e, "expected", set()):
            e.interactive_parser.feed_token(Token("__ANON_0", ";"))
            return True  # retry parsing
        return False      # give up, raise the error

    try:
        return parser.parse(text, on_error=on_error)
    except UnexpectedInput as e:
        print(f"Unrecoverable error at line {e.line}, column {e.column}")
        return None


# Usage
print("--- Simple error handling ---")
parse_simple("x = 1 + @;")  # Lexer error: unexpected '@'
parse_simple("x = ;")       # Parser error: unexpected ';'

print("\n--- match_examples ---")
try:
    parse_with_matching("x = 1 + 2\n")
except SyntaxError as e:
    print(f"SyntaxError: {e}")

print("\n--- Recovery ---")
tree = parse_with_recovery("x = 1 + 2\ny = 3;")
if tree:
    print(tree.pretty())
```

---

## Quick Reference

### Lark Constructor Parameters

| Parameter | Default | Description |
|---|---|---|
| `parser` | `"earley"` | Parser algorithm: `"earley"` or `"lalr"` |
| `lexer` | `"auto"` | Lexer type: `"auto"`, `"standard"`, `"contextual"`, `"dynamic"` |
| `start` | `"start"` | Name of the start rule |
| `ambiguity` | `"auto"` | Ambiguity handling: `"auto"`, `"explicit"`, `"resolve"` |
| `transformer` | `None` | Apply transformer during parsing (LALR only) |
| `propagate_positions` | `False` | Track line/column on Tree nodes |
| `maybe_placeholders` | `True` | Insert `None` for unfilled `?` options (LALR) |
| `keep_all_tokens` | `False` | Keep anonymous tokens in the tree |
| `regex` | `False` | Use the `regex` module instead of `re` |
| `g_regex_flags` | `0` | Global regex flags applied to all terminals |
| `source_path` | `None` | Path for relative grammar imports |

### Common Imports Cheat Sheet

```lark
%import common.NUMBER           // Integer or decimal: 42, 3.14
%import common.INT              // Integer only: 42
%import common.SIGNED_NUMBER    // Signed: -42, +3.14
%import common.FLOAT            // Float only: 3.14
%import common.WORD             // Word: hello123
%import common.CNAME            // C-name: my_var_1
%import common.ESCAPED_STRING   // Quoted: "hello \"world\""
%import common.WS               // All whitespace (spaces, tabs, newlines)
%import common.WS_INLINE        // Inline whitespace (spaces, tabs only)
%import common.NEWLINE           // Line endings
%import common.SH_COMMENT       // Shell comments: # ...
```

### Links

- **PyPI:** [https://pypi.org/project/lark/](https://pypi.org/project/lark/)
- **GitHub:** [https://github.com/lark-parser/lark](https://github.com/lark-parser/lark)
- **Documentation:** [https://lark-parser.readthedocs.io/](https://lark-parser.readthedocs.io/)
- **Grammar Reference:** [https://lark-parser.readthedocs.io/en/latest/grammar.html](https://lark-parser.readthedocs.io/en/latest/grammar.html)
- **Cheat Sheet:** [https://lark-parser.readthedocs.io/en/latest/lark_cheatsheet.pdf](https://lark-parser.readthedocs.io/en/latest/lark_cheatsheet.pdf)
- **Online IDE (try Lark in browser):** [https://www.lark-parser.org/ide/](https://www.lark-parser.org/ide/)
