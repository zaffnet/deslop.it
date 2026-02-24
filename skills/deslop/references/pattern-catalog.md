# AI Slop Pattern Catalog

Reference material for the AI Slop Pattern Taxonomy in Step 2 of the `/deslop` skill.

Each pattern includes a before/after code example and a verification note explaining how to confirm the finding.

## Contents

- [Category 1: Over-Defensive Code (1.0x)](#category-1-over-defensive-code-weight-10x)
- [Category 2: Premature Abstraction (1.5x)](#category-2-premature-abstraction-weight-15x)
- [Category 3: Dead Weight (1.5x)](#category-3-dead-weight-weight-15x)
- [Category 4: Verbose Patterns (1.0x)](#category-4-verbose-patterns-weight-10x)
- [Category 5: Structural Bloat (1.5x)](#category-5-structural-bloat-weight-15x)
- [Category 6: Documentation & Logging Noise (1.0x)](#category-6-documentation--logging-noise-weight-10x)
- [Config/YAML Slop](#configyaml-slop-no-weight--reported-separately)
- [Verification Methodology Summary](#verification-methodology-summary)

---

## Category 1: Over-Defensive Code (Weight: 1.0x)

### Unreachable guard

`if x is not None` on values that cannot be None — required params with non-Optional types.

```python
# Before (slop)
def process_item(item: Item) -> str:
    if item is not None:
        return item.name
    return ""
```

```python
# After (idiomatic)
def process_item(item: Item) -> str:
    return item.name
```

**Verification:** Check the type annotation and all callers. If every caller passes a non-None value and the type hint is non-Optional, the guard is unreachable.

### Impossible except

`try/except` around operations that cannot raise the caught exception.

```python
# Before (slop)
def get_name(user: User) -> str:
    try:
        return user.name
    except AttributeError:
        return "Unknown"
```

```python
# After (idiomatic)
def get_name(user: User) -> str:
    return user.name
```

**Verification:** Confirm the type is concrete (not `Any` or dynamic), and the attribute access is defined on the type.

### Redundant isinstance

`isinstance()` on values whose type is statically known.

```python
# Before (slop)
def format_value(value: str) -> str:
    if isinstance(value, str):
        return value.strip()
    return str(value)
```

```python
# After (idiomatic)
def format_value(value: str) -> str:
    return value.strip()
```

**Verification:** Check the type annotation. If the type is concrete (not `Union`, `Any`, or `object`), the isinstance check is redundant.

### Internal validation

Input validation on internal-only functions (not at system boundaries).

```python
# Before (slop)
def _compute_score(values: list[float]) -> float:
    if not isinstance(values, list):
        raise TypeError("values must be a list")
    if not all(isinstance(v, (int, float)) for v in values):
        raise TypeError("all values must be numeric")
    return sum(values) / len(values)
```

```python
# After (idiomatic)
def _compute_score(values: list[float]) -> float:
    return sum(values) / len(values)
```

**Verification:** Confirm the function is internal (leading underscore or only called from within the same module/package). System-boundary validation is NOT slop.

### Shadowed default

Default parameter values that duplicate what every caller always provides.

```python
# Before (slop)
def fetch_data(url: str, timeout: int = 30) -> Response:
    return requests.get(url, timeout=timeout)

# Every single caller:
fetch_data(url, timeout=30)
fetch_data(other_url, timeout=30)
```

```python
# After (idiomatic) — either remove default or remove caller args
def fetch_data(url: str, timeout: int = 30) -> Response:
    return requests.get(url, timeout=timeout)

# Callers simplified:
fetch_data(url)
fetch_data(other_url)
```

**Verification:** `Grep` for all callers of the function. If every caller passes the same value that matches the default, the explicit arguments are redundant.

### Guaranteed key

Defensive `.get()` on dicts where the key is guaranteed to exist.

```python
# Before (slop)
config = {"host": "localhost", "port": 8080}
host = config.get("host", "localhost")
port = config.get("port", 8080)
```

```python
# After (idiomatic)
config = {"host": "localhost", "port": 8080}
host = config["host"]
port = config["port"]
```

**Verification:** Trace the dict construction. If the key is set in a literal or always assigned before access, `.get()` is unnecessary.

### Silent swallow

`try/except` that catches exceptions and returns a default, silently hiding errors that should propagate.

```python
# Before (slop)
def load_config(path: str) -> dict:
    try:
        with open(path) as f:
            return json.load(f)
    except Exception:
        return {}
```

```python
# After (idiomatic)
def load_config(path: str) -> dict:
    with open(path) as f:
        return json.load(f)
```

**Verification:** Check whether callers handle the empty/default return differently from the success case. If they don't, the error is being silently swallowed. Exception: if this is at a system boundary with documented fallback behavior, it may be intentional.

---

## Category 2: Premature Abstraction (Weight: 1.5x)

### One-caller helper

Helper function called exactly once from one call site. Inline it.

```python
# Before (slop)
def _build_header(title: str) -> str:
    return f"=== {title} ==="

def render_report(title: str, body: str) -> str:
    header = _build_header(title)
    return f"{header}\n{body}"
```

```python
# After (idiomatic)
def render_report(title: str, body: str) -> str:
    header = f"=== {title} ==="
    return f"{header}\n{body}"
```

**Verification:** `Grep` for the function name across the entire project. Count unique call sites (not just matches). If exactly 1 call site exists, it's a one-caller helper. Exception: if the function encapsulates complex logic (>15 lines) that benefits from naming for readability.

### Transparent wrapper

Wrapper class that adds no behavior, just delegates to the wrapped object.

```python
# Before (slop)
class DatabaseConnection:
    def __init__(self, pool: ConnectionPool):
        self._pool = pool

    def execute(self, query: str) -> Result:
        return self._pool.execute(query)

    def close(self) -> None:
        self._pool.close()
```

```python
# After (idiomatic) — use the pool directly
pool = ConnectionPool(...)
pool.execute(query)
pool.close()
```

**Verification:** Check every method of the wrapper. If all methods simply delegate to the wrapped object without adding logic, error handling, or state, the wrapper is transparent. Count how many unique attributes/methods are accessed by external code.

### Configuration mirage

Constants for values that are never configurable and never change.

```python
# Before (slop)
DEFAULT_SEPARATOR = ", "
MAX_RETRIES = 3
ENCODING = "utf-8"

def format_list(items: list[str]) -> str:
    return DEFAULT_SEPARATOR.join(items)
```

```python
# After (idiomatic)
def format_list(items: list[str]) -> str:
    return ", ".join(items)
```

**Verification:** `Grep` for all usages of the constant. If it's used in exactly one place, it's a once-used constant (Cat 3). If it's used in multiple places but the value would never realistically change (like `"utf-8"`), it's a configuration mirage. Check: is this value ever overridden, configured via env var, or documented as configurable?

### One-product factory

Factory function that only ever creates one type of object.

```python
# Before (slop)
def create_processor(kind: str = "default") -> Processor:
    if kind == "default":
        return DefaultProcessor()
    raise ValueError(f"Unknown kind: {kind}")
```

```python
# After (idiomatic)
processor = DefaultProcessor()
```

**Verification:** `Grep` for all callers. If every caller passes the same argument (or uses the default), the factory only ever produces one product. Exception: if a second variant is planned and documented.

### Single-variant generalization

Parameterizing behavior that has only one variant.

```python
# Before (slop)
def process(data: list, strategy: str = "standard") -> list:
    if strategy == "standard":
        return [x * 2 for x in data]
    raise ValueError(f"Unknown strategy: {strategy}")
```

```python
# After (idiomatic)
def process(data: list) -> list:
    return [x * 2 for x in data]
```

**Verification:** `Grep` for all callers. If only one variant is ever used, the parameterization is premature.

### Lonely enum

Enum class with a single member.

```python
# Before (slop)
class OutputFormat(Enum):
    JSON = "json"

def export(data: dict, fmt: OutputFormat = OutputFormat.JSON) -> str:
    if fmt == OutputFormat.JSON:
        return json.dumps(data)
```

```python
# After (idiomatic)
def export(data: dict) -> str:
    return json.dumps(data)
```

**Verification:** Check the enum definition. If it has exactly one member, and `Grep` confirms no other members are ever referenced or planned, it's a lonely enum.

### Intermediate data structure addiction

Creating a data class/dict to pass between two functions when direct parameters would suffice.

```python
# Before (slop)
@dataclass
class ProcessingContext:
    input_path: str
    output_path: str
    verbose: bool

def prepare(path: str) -> ProcessingContext:
    return ProcessingContext(
        input_path=path,
        output_path=path + ".out",
        verbose=True,
    )

def execute(ctx: ProcessingContext) -> None:
    process(ctx.input_path, ctx.output_path, ctx.verbose)
```

```python
# After (idiomatic)
def execute(path: str) -> None:
    process(path, path + ".out", verbose=True)
```

**Verification:** Count how many functions consume the data structure. If only 1-2 functions use it, and the structure is only passed linearly (A creates it, B consumes it), direct parameters are simpler. If 3+ consumers exist or the structure is passed through multiple layers, it may earn its keep.

---

## Category 3: Dead Weight (Weight: 1.5x)

### Unused parameter

Function parameter that is accepted but never read in the function body.

```python
# Before (slop)
def calculate_total(items: list[Item], currency: str = "USD") -> float:
    return sum(item.price for item in items)
    # currency is never used
```

```python
# After (idiomatic)
def calculate_total(items: list[Item]) -> float:
    return sum(item.price for item in items)
```

**Verification:** Search the function body for any reference to the parameter name. If it appears nowhere (not in assignments, expressions, or calls), it's unused. Also update all callers to remove the argument.

### Once-used constant

Module-level constant used in exactly one place. Inline the value.

```python
# Before (slop)
BATCH_SIZE = 100

def process_items(items: list) -> None:
    for i in range(0, len(items), BATCH_SIZE):
        handle_batch(items[i:i + BATCH_SIZE])
```

```python
# After (idiomatic)
def process_items(items: list) -> None:
    batch_size = 100
    for i in range(0, len(items), batch_size):
        handle_batch(items[i:i + batch_size])
```

**Verification:** `Grep` for the constant name. If it appears in exactly 2 places (definition + one usage), inline it as a local variable.

### Zero-caller code

Functions, methods, or classes with zero callers anywhere in the project.

```python
# Before (slop)
def format_debug_output(data: dict) -> str:
    """Format data for debug display."""
    lines = []
    for key, value in sorted(data.items()):
        lines.append(f"  {key}: {value}")
    return "\n".join(lines)

# This function is never called anywhere
```

```python
# After (idiomatic)
# Delete the function entirely
```

**Verification:** `Grep` for the function/class name across the entire project (including test files). If zero call sites exist, it's dead code. Exception: public API entry points, CLI handlers, and framework hooks (e.g., `__init__`, signal handlers) that are called implicitly.

### Pre-computed state

Variables computed and stored but only read once immediately after.

```python
# Before (slop)
def get_summary(items: list[Item]) -> str:
    total = sum(item.price for item in items)
    count = len(items)
    average = total / count if count > 0 else 0
    formatted_total = f"${total:.2f}"
    formatted_average = f"${average:.2f}"
    return f"{count} items, total: {formatted_total}, avg: {formatted_average}"
```

```python
# After (idiomatic)
def get_summary(items: list[Item]) -> str:
    total = sum(item.price for item in items)
    count = len(items)
    average = total / count if count > 0 else 0
    return f"{count} items, total: ${total:.2f}, avg: ${average:.2f}"
```

**Verification:** Check if the variable is read more than once after assignment. If it's assigned and then immediately used in exactly one expression, inline it. Exception: if the computation is complex and the variable name aids readability.

### Import-only module

Module that only re-exports imports from other modules without adding any logic.

```python
# Before (slop) — utils/__init__.py
from .string_utils import clean_text
from .date_utils import parse_date
from .file_utils import read_json

# No other code in this file
```

```python
# After (idiomatic) — import directly from source modules
from utils.string_utils import clean_text
from utils.date_utils import parse_date
from utils.file_utils import read_json
```

**Verification:** Read the module. If it contains only import/re-export statements and no functions, classes, or logic, it's an import-only module. Exception: `__init__.py` files that define a public API for a package (documented as the intended import path).

---

## Category 4: Verbose Patterns (Weight: 1.0x)

### Boolean long-hand

`if condition: return True else: return False` instead of `return condition`.

```python
# Before (slop)
def is_valid(value: str) -> bool:
    if len(value) > 0 and value.isalpha():
        return True
    else:
        return False
```

```python
# After (idiomatic)
def is_valid(value: str) -> bool:
    return len(value) > 0 and value.isalpha()
```

**Verification:** No special verification needed — this is a pure syntactic pattern.

### Manual list build

Loop appending to a list where a list comprehension would suffice.

```python
# Before (slop)
def get_names(users: list[User]) -> list[str]:
    names = []
    for user in users:
        names.append(user.name)
    return names
```

```python
# After (idiomatic)
def get_names(users: list[User]) -> list[str]:
    return [user.name for user in users]
```

**Verification:** Confirm the loop body is a single append with no side effects or conditional logic. Complex loops with multiple statements are not candidates.

### String concatenation chain

Multi-line string concatenation instead of f-string or `str.join()`.

```python
# Before (slop)
def build_message(name: str, count: int) -> str:
    message = "Hello, " + name + ". "
    message += "You have " + str(count) + " items. "
    message += "Thank you."
    return message
```

```python
# After (idiomatic)
def build_message(name: str, count: int) -> str:
    return f"Hello, {name}. You have {count} items. Thank you."
```

**Verification:** No special verification needed — pure syntactic pattern.

### Constructor over literal

`dict()` / `list()` constructor instead of `{}` / `[]` literal.

```python
# Before (slop)
config = dict(host="localhost", port=8080)
items = list()
```

```python
# After (idiomatic)
config = {"host": "localhost", "port": 8080}
items = []
```

**Verification:** No special verification needed. Exception: `dict()` with keyword args is sometimes preferred in codebases for readability — check codebase convention.

### One-shot import

Importing a module to access one attribute exactly once.

```python
# Before (slop)
import os

path = os.path.join(base_dir, filename)
# os is never used again in this file
```

```python
# After (idiomatic)
from os.path import join

path = join(base_dir, filename)
```

**Verification:** `Grep` within the file for other usages of the imported module. If only one attribute is accessed once, use a targeted import.

### Explicit length check

`len(x) == 0` / `len(x) > 0` instead of truthiness check.

```python
# Before (slop)
if len(items) == 0:
    return "No items"
if len(name) > 0:
    print(name)
```

```python
# After (idiomatic)
if not items:
    return "No items"
if name:
    print(name)
```

**Verification:** Confirm the variable is a collection type (list, dict, set, str, tuple) where truthiness is equivalent to length check. Does not apply to custom objects without `__len__`.

### Redundant else

`else` after `return` / `raise` / `continue`.

```python
# Before (slop)
def classify(value: int) -> str:
    if value > 0:
        return "positive"
    else:
        return "non-positive"
```

```python
# After (idiomatic)
def classify(value: int) -> str:
    if value > 0:
        return "positive"
    return "non-positive"
```

**Verification:** Confirm the `if` branch unconditionally returns/raises/continues. The `else` keyword is then purely redundant.

### Deep nesting / early-return phobia

Multiple levels of `if` nesting that could be flattened with guard clauses.

```python
# Before (slop)
def process(data: dict | None) -> str:
    if data is not None:
        if "key" in data:
            value = data["key"]
            if isinstance(value, str):
                return value.strip()
    return ""
```

```python
# After (idiomatic)
def process(data: dict | None) -> str:
    if data is None:
        return ""
    if "key" not in data:
        return ""
    value = data["key"]
    if not isinstance(value, str):
        return ""
    return value.strip()
```

**Verification:** Count nesting levels. If ≥3 levels of `if` nesting can be flattened to sequential guard clauses with early returns, it's a finding.

---

## Category 5: Structural Bloat (Weight: 1.5x)

### Delegation chain

A calls B calls C, where B adds no logic — just passes arguments through.

```python
# Before (slop)
def handle_request(request: Request) -> Response:
    return _process_request(request)

def _process_request(request: Request) -> Response:
    return _execute_request(request)

def _execute_request(request: Request) -> Response:
    # actual logic here
    return Response(data=request.body)
```

```python
# After (idiomatic)
def handle_request(request: Request) -> Response:
    return Response(data=request.body)
```

**Verification:** Map the call chain from entry point to actual logic. Count intermediate functions that only delegate. If ≥2 intermediates add no logic, no error handling, and no branching, the chain is a finding. Exception: if intermediate functions are public API entry points with different signatures.

### Symmetry addiction

Parallel code structures maintained for "consistency" when only one path is used.

```python
# Before (slop)
class ExportManager:
    def export_json(self, data: dict) -> str:
        return json.dumps(data)

    def export_xml(self, data: dict) -> str:
        raise NotImplementedError("XML export not supported")

    def export_csv(self, data: dict) -> str:
        raise NotImplementedError("CSV export not supported")
```

```python
# After (idiomatic)
def export_json(data: dict) -> str:
    return json.dumps(data)
```

**Verification:** Check which methods/branches are actually called. If only one variant is used and the others raise `NotImplementedError` or are never called, the symmetry is premature.

### Gratuitous generics

Type parameters or Protocol classes used in exactly one concrete instantiation.

```python
# Before (slop)
T = TypeVar("T")

class Repository(Generic[T]):
    def get(self, id: str) -> T: ...
    def save(self, item: T) -> None: ...

class UserRepository(Repository[User]):
    def get(self, id: str) -> User: ...
    def save(self, item: User) -> None: ...

# UserRepository is the only subclass
```

```python
# After (idiomatic)
class UserRepository:
    def get(self, id: str) -> User: ...
    def save(self, item: User) -> None: ...
```

**Verification:** `Grep` for all subclasses or concrete instantiations of the generic. If exactly one exists, the generic is gratuitous.

### Error message over-engineering

Complex error formatting for errors that are never caught or displayed to users.

```python
# Before (slop)
def parse_config(path: str) -> dict:
    if not os.path.exists(path):
        error_context = {
            "path": path,
            "cwd": os.getcwd(),
            "attempted_at": datetime.now().isoformat(),
        }
        raise FileNotFoundError(
            f"Configuration file not found at '{path}'. "
            f"Current directory: {error_context['cwd']}. "
            f"Attempted at: {error_context['attempted_at']}. "
            f"Please verify the path exists and is accessible."
        )
```

```python
# After (idiomatic)
def parse_config(path: str) -> dict:
    if not os.path.exists(path):
        raise FileNotFoundError(f"Config not found: {path}")
```

**Verification:** Check whether callers catch this specific exception and display/log the message. If the error just propagates to a generic handler or traceback, the elaborate message is wasted effort.

### Repetitive structure

Copy-pasted code blocks with minor variations that should be a loop or mapping.

```python
# Before (slop)
def validate_fields(data: dict) -> list[str]:
    errors = []
    if "name" not in data:
        errors.append("Missing field: name")
    if "email" not in data:
        errors.append("Missing field: email")
    if "phone" not in data:
        errors.append("Missing field: phone")
    if "address" not in data:
        errors.append("Missing field: address")
    return errors
```

```python
# After (idiomatic)
REQUIRED_FIELDS = ["name", "email", "phone", "address"]

def validate_fields(data: dict) -> list[str]:
    return [f"Missing field: {f}" for f in REQUIRED_FIELDS if f not in data]
```

**Verification:** Identify the varying element(s) across the repeated blocks. If the blocks differ only in a single value, they can be collapsed into a loop or mapping.

---

## Category 6: Documentation & Logging Noise (Weight: 1.0x)

### Restating comment

Comment that restates what the code already says.

```python
# Before (slop)
# Increment the counter by one
counter += 1

# Return the result
return result

# Check if the list is empty
if not items:
    return []
```

```python
# After (idiomatic)
counter += 1
return result
if not items:
    return []
```

**Verification:** Read the comment and the code together. If removing the comment loses zero information (the code is self-explanatory), it's a restating comment. Exception: comments that explain *why* something is done, not *what* is done.

### Signature-restating docstring

Docstring that only repeats the function signature without adding useful information.

```python
# Before (slop)
def calculate_total(items: list[Item]) -> float:
    """Calculate the total.

    Args:
        items: The list of items.

    Returns:
        The total as a float.
    """
    return sum(item.price for item in items)
```

```python
# After (idiomatic)
def calculate_total(items: list[Item]) -> float:
    return sum(item.price for item in items)
```

**Verification:** Compare the docstring content with the function signature. If the docstring adds no information beyond what the signature, parameter names, and type hints already convey, it's noise. Exception: docstrings that explain business logic, edge cases, or non-obvious behavior.

### Redundant logging

Log statements that add no diagnostic value.

```python
# Before (slop)
def process(data: dict) -> dict:
    logger.debug("Starting process")
    logger.debug(f"Input data: {data}")
    result = transform(data)
    logger.debug(f"Process completed with result: {result}")
    return result
```

```python
# After (idiomatic)
def process(data: dict) -> dict:
    return transform(data)
```

**Verification:** Check if the log statements provide information useful for debugging in production. Entry/exit logs at every function are noise. Logs that capture decision points, error contexts, or performance data may be valuable. If in doubt, leave logging alone.

### Empty docstring sections

Docstring with empty or placeholder Args/Returns/Raises sections.

```python
# Before (slop)
def fetch(url: str) -> bytes:
    """Fetch content from URL.

    Args:
        url: The URL to fetch.

    Returns:
        The fetched content.

    Raises:
        N/A
    """
    return requests.get(url).content
```

```python
# After (idiomatic)
def fetch(url: str) -> bytes:
    """Fetch content from URL."""
    return requests.get(url).content
```

**Verification:** Check each docstring section. If Args just restates parameter names, Returns just restates the type, or Raises says "N/A" or "None", those sections are empty noise.

---

## Config/YAML Slop (no weight — reported separately)

### Commented-out settings

```yaml
# Before (slop)
database:
  host: localhost
  port: 5432
  # pool_size: 10
  # max_overflow: 20
  # timeout: 30
```

```yaml
# After (idiomatic)
database:
  host: localhost
  port: 5432
```

### Redundant defaults

```toml
# Before (slop)
[tool.ruff]
line-length = 88  # 88 is ruff's default
target-version = "py312"
```

```toml
# After (idiomatic)
[tool.ruff]
target-version = "py312"
```

**Verification:** Check the tool's documentation for default values. If the configured value matches the built-in default, it's redundant.

---

## Verification Methodology Summary

| Technique | When to Apply | What Disqualifies a Finding |
|-----------|---------------|----------------------------|
| **Caller count** | One-caller helpers (Cat 2), zero-caller code (Cat 3), delegation chains (Cat 5), gratuitous generics (Cat 5) | Function has 2+ unique callers; class has 2+ instantiations |
| **Data tracing** | Unused parameters (Cat 3), configuration mirage (Cat 2) | Parameter influences return value or side effect; constant is dynamically read |
| **Parameter tax** | Intermediate data structures (Cat 2), pre-computed state (Cat 3) | Structure used in 2+ distinct contexts; removing it increases total lines |
| **Call chain depth** | Delegation chains (Cat 5) | Any intermediate function adds logic, error handling, or branching |
| **Guard reachability** | Unreachable guards (Cat 1), redundant isinstance (Cat 1) | Type is `Any`, `Optional`, or `Union`; any caller passes `None` |
| **Attribute access count** | Transparent wrappers (Cat 2), intermediate data structures (Cat 2) | 3+ unique attributes are accessed by external code |
