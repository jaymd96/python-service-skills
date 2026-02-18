# Lark — Advanced Features

> Part of the lark skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Top-Down Processing with Interpreter](#top-down-processing-with-interpreter)
- [Additional Transformer Patterns](#additional-transformer-patterns)
- [Advanced Features](#advanced-features)
  - [Ambiguity Handling](#ambiguity-handling)
  - [Contextual Lexing](#contextual-lexing)
  - [Standalone Parser Generation](#standalone-parser-generation)
  - [Custom Lexer](#custom-lexer)
  - [Interactive Parsing (LALR only)](#interactive-parsing-lalr-only)
  - [Error Recovery](#error-recovery)
  - [Propagating Source Positions](#propagating-source-positions)

## Top-Down Processing with `Interpreter`

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

## Additional Transformer Patterns

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
