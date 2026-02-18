---
name: lark
description: Modern parsing library with grammar-as-data design. Use when building parsers, defining grammars (EBNF), creating DSLs, or processing structured text with Earley/LALR algorithms. Triggers on parsing, grammar, DSL, lark, parser generator, EBNF, Earley, LALR, Transformer.
---

# lark — Parsing Library (v1.2.2)

## Quick Start

```bash
pip install lark
```

```python
from lark import Lark

grammar = """
    start: NAME "=" value
    value: NUMBER | STRING
    NAME: /[a-zA-Z_]\w*/
    STRING: /"[^"]*"/
    NUMBER: /\d+(\.\d+)?/
    %ignore /\s+/
"""
parser = Lark(grammar)
tree = parser.parse('x = 42')
```

## Key Patterns

### Grammar basics
```
rule: item1 item2 | alternative    # Rules (lowercase)
TERMINAL: /regex/ | "literal"      # Terminals (UPPERCASE)
?rule: ...                          # Inline (removes from tree)
%import common.NUMBER               # Import from common grammar
%ignore /\s+/                       # Ignore whitespace
```

### Transformer (process parse tree)
```python
from lark import Transformer, v_args

@v_args(inline=True)
class CalcTransformer(Transformer):
    def number(self, n):
        return float(n)
    def add(self, a, b):
        return a + b

result = CalcTransformer().transform(tree)
```

## References

- **[api.md](references/api.md)** — Lark constructor, grammar syntax (EBNF), parser options, Tree/Token classes, Transformer basics
- **[advanced.md](references/advanced.md)** — Advanced Transformer patterns, Visitor, Interpreter, grammar imports, error recovery
- **[examples.md](references/examples.md)** — Complete examples, gotchas, common grammar patterns
