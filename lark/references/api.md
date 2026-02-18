# Lark — API Reference

> Part of the lark skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [Grammar Definition Syntax](#grammar-definition-syntax)
  - [Terminals (UPPERCASE) vs Rules (lowercase)](#terminals-uppercase-vs-rules-lowercase)
  - [Creating Parsers with Lark()](#creating-parsers-with-lark)
  - [Parser Types](#parser-types)
  - [Tree and Token -- Parse Tree Structure](#tree-and-token----parse-tree-structure)
- [Grammar Syntax Reference](#grammar-syntax-reference)
  - [Rules](#rules)
  - [Terminals](#terminals)
  - [Rule Modifiers](#rule-modifiers)
  - [Terminal Modifiers](#terminal-modifiers)
  - [Repeat Modifier ~](#repeat-modifier-)
  - [Priorities](#priorities)
  - [Imports](#imports)
  - [The Common Grammar (%import common)](#the-common-grammar-import-common)
  - [Directives](#directives)
- [Transformers](#transformers)
  - [The Transformer Class](#the-transformer-class)
  - [@v_args(inline=True) for Cleaner Callbacks](#v_argsinlinetrue-for-cleaner-callbacks)
  - [Bottom-Up Processing (Transformer)](#bottom-up-processing-transformer)
- [Quick Reference](#quick-reference)
  - [Lark Constructor Parameters](#lark-constructor-parameters)
  - [Common Imports Cheat Sheet](#common-imports-cheat-sheet)

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
