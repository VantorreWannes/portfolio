---
icon: simple/python
---

# PythonWrangler

**A streamlined, AST-aware unit testing library.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/PythonWrangler){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    PythonWrangler is a lightweight, down-to-earth testing framework tailored for developers who need robust assertions without the heavy boilerplate of standard libraries like `unittest` or `pytest`. It introduces a powerful `@test` decorator that applies cascading configurations to classes, methods, and functions dynamically.

## Core Mechanics & Impact

Standard testing frameworks often overwhelm newer developers with deep, confusing error tracebacks that point to internal library code rather than the actual failing test. 

PythonWrangler solves this by manipulating the Python traceback directly. When a test fails, the framework intercepts the error, truncates the stack trace, and points the developer exactly to the line of code that failed. This drastically improves the developer experience (DX) and reduces debugging time.

---

## Technical Architecture

=== "Cascading Settings"

    The `@test` decorator supports both explicit and implicit configurations (e.g., `crash_on_false`, `verbose`). 
    
    Using custom `TestableClass` and `TestableFunction` wrapper objects, settings cascade hierarchically. A higher-level explicit setting defined on a `class` will properly override or inherit down to its inner methods, creating a highly predictable and flexible test environment.

=== "Traceback Manipulation"

    When an `AffirmError` is raised, PythonWrangler does not just throw a standard exception. It utilizes Python's internal `sys.exc_info()` and the `types.TracebackType` module to construct a completely new, truncated stack trace. 
    
    It physically removes the topmost frames (the framework's own execution logic) before passing the exception back to the user, ensuring the output is perfectly clean.

---

## Implementation Highlight

Below is the internal mechanism used to truncate tracebacks. Interacting with raw frame objects (`tb_frame`) requires a deep understanding of how the Python interpreter tracks execution context in memory.

```python title="src/PythonWrangler/__internals/affirm_error.py" hl_lines="4 7"
def __trunicate_traceback(self, traceback: types.TracebackType, amount: int):
    back_frame = traceback.tb_frame
    
    # Step back through the execution frames to hide internal framework logic
    for _ in range(0, amount):
        back_frame = back_frame.f_back
        
    # Reconstruct a clean traceback pointing only to the user's code
    return types.TracebackType(tb_next=None,
                               tb_frame=back_frame,
                               tb_lasti=back_frame.f_lasti,
                               tb_lineno=back_frame.f_lineno)
```

## Usage Example

!!! info "Clean Assertions"
    The API exposes highly readable `affirm()`, `affirm_eq()`, and `raises()` context managers.
    
```python title="test_example.py"
from python_wrangler import affirm, affirm_eq, raises, test

@test(crash_on_false=False, verbose=True)
def test_logic():
    affirm(1 < 2)
    affirm_eq("Production", "Production")
    
    with raises(Exception):
        raise Exception("This is safely caught")
```
