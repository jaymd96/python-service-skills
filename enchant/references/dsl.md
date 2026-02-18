# Enchant — DSL Reference

> Part of the enchant skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Type System](#type-system)
- [Validation Keywords](#validation-keywords)
- [Safety Annotations](#safety-annotations)
- [Services](#services)
- [Errors](#errors)
- [Cross-File Imports](#cross-file-imports)
- [Compiler Validation Rules](#compiler-validation-rules)

## Type System

### Primitives
`string`, `integer`, `safelong`, `double`, `boolean`, `datetime`, `uuid`, `binary`

### Objects

```yaml
types:
  SearchRequest:
    fields:
      origin:
        type: string
        safety: safe
        docs: "IATA airport code"
        validation:
          min-length: 3
          max-length: 3
          pattern: "^[A-Z]{3}$"
      passenger_name:
        type: string
        safety: unsafe        # Redacted in logs
```

### Enums

```yaml
types:
  FlightClass:
    values:
      - ECONOMY
      - BUSINESS
      - FIRST
```

### Unions (tagged discriminated)

```yaml
types:
  PaymentMethod:
    union:
      credit_card: CreditCardPayment
      bank_transfer: BankTransferPayment
```

### Aliases (transparent wrappers with validation)

```yaml
types:
  AirportCode:
    alias: string
    validation:
      min-length: 3
      max-length: 3
      pattern: "^[A-Z]{3}$"
```

### Containers

```yaml
flights: { type: "list<FlightResult>" }
metadata: { type: "optional<SearchMetadata>" }
tags: { type: "set<string>" }
prices: { type: "map<string,double>" }
```

## Validation Keywords

| Keyword | Applies To | Example |
|---------|-----------|---------|
| `min-length` | string | `min-length: 1` |
| `max-length` | string | `max-length: 255` |
| `pattern` | string | `pattern: "^[A-Z]{3}$"` |
| `minimum` | integer, double | `minimum: 0` |
| `maximum` | integer, double | `maximum: 100` |
| `format` | string | `format: email` / `uri` / `ipv4` / `ipv6` |

## Safety Annotations

| Level | Behavior |
|-------|----------|
| `safe` | Logged normally in all environments |
| `unsafe` | Redacted to `<REDACTED>` in production logs |
| `do-not-log` | Never appears in any log output |

Safety flows from DSL -> generated models -> logging -> wire format.

## Services

```yaml
services:
  FlightSearchService:
    base-path: /api/flights
    default-auth:
      type: header
      header: Authorization
    endpoints:
      searchFlights:
        http: GET /search
        args:
          origin: { type: string, param-type: query }
          destination: { type: string, param-type: query }
        returns: list<FlightResult>
        errors:
          - InvalidSearchCriteria

      getFlight:
        http: GET /{flightId}
        args:
          flightId: { type: string, param-type: path }
        returns: FlightResult
        errors:
          - FlightNotFound

      bookFlight:
        http: POST /book
        args:
          request: { type: BookingRequest, param-type: body }
        returns: BookingConfirmation
        errors:
          - FlightNotFound
          - PaymentFailed
```

### Param Types
- `path` — URL path parameter (primitives/enums/aliases only)
- `query` — Query string parameter (primitives/enums/aliases only)
- `body` — Request body (any type, at most one per endpoint)

## Errors

```yaml
errors:
  FlightNotFound:
    code: NOT_FOUND
    safe-args:
      flightId: string
    unsafe-args: {}

  PaymentFailed:
    code: FAILED_PRECONDITION
    safe-args:
      reason: string
    unsafe-args:
      cardLastFour: string
```

## Cross-File Imports

```yaml
types:
  imports:
    SharedTypes:
      file: shared-types.enchant.yml

  FlightResult:
    fields:
      airport: { type: SharedTypes.Airport }
```

## Compiler Validation Rules

9 rules enforced at compile time:

| Rule | What It Checks |
|------|---------------|
| **UniqueNames** | No duplicate type/service/error names across files |
| **NoRecursiveTypes** | Direct type recursion forbidden |
| **NoNestedOptionals** | `optional<optional<T>>` is illegal |
| **FieldCasing** | Fields: camelCase, types: PascalCase, enums: UPPER_SNAKE_CASE |
| **EndpointConstraints** | Path/query/header params must be primitives, enums, or aliases |
| **PathParamPresence** | All `{param}` in URL must have matching arg |
| **SingleBody** | At most one body param per endpoint |
| **NoBodyOnGet** | GET/DELETE cannot have body params |
| **ValidationTargets** | Validation keywords must match field types |
