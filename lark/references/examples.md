# Lark — Examples & Gotchas

> Part of the lark skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Earley vs LALR Tradeoffs](#1-earley-vs-lalr-tradeoffs)
  - [2. Whitespace Handling](#2-whitespace-handling)
  - [3. Priority Conflicts](#3-priority-conflicts)
  - [4. Performance with Large Grammars](#4-performance-with-large-grammars)
  - [5. Tree vs Token Confusion](#5-tree-vs-token-confusion)
  - [6. Anonymous Terminals (String Literals) Are Filtered](#6-anonymous-terminals-string-literals-are-filtered)
  - [7. The ? Rule Modifier Can Surprise You](#7-the--rule-modifier-can-surprise-you)
  - [8. Start Rule Must Be Named start (by default)](#8-start-rule-must-be-named-start-by-default)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Simple Calculator](#example-1-simple-calculator)
  - [Example 2: JSON Parser](#example-2-json-parser)
  - [Example 3: Custom DSL for Configuration](#example-3-custom-dsl-for-configuration)
  - [Example 4: Evaluating Expressions with a Transformer](#example-4-evaluating-expressions-with-a-transformer)
  - [Example 5: Error Handling and Recovery](#example-5-error-handling-and-recovery)
- [Links](#links)

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

## Links

- **PyPI:** [https://pypi.org/project/lark/](https://pypi.org/project/lark/)
- **GitHub:** [https://github.com/lark-parser/lark](https://github.com/lark-parser/lark)
- **Documentation:** [https://lark-parser.readthedocs.io/](https://lark-parser.readthedocs.io/)
- **Grammar Reference:** [https://lark-parser.readthedocs.io/en/latest/grammar.html](https://lark-parser.readthedocs.io/en/latest/grammar.html)
- **Cheat Sheet:** [https://lark-parser.readthedocs.io/en/latest/lark_cheatsheet.pdf](https://lark-parser.readthedocs.io/en/latest/lark_cheatsheet.pdf)
- **Online IDE (try Lark in browser):** [https://www.lark-parser.org/ide/](https://www.lark-parser.org/ide/)
