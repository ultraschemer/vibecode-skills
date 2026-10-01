---
name: documentation-standards
description: Use when writing any new function, class, method, module, or SQL query that needs docstrings and inline comments, and when the user explicitly asks to document existing code. Defines the mandatory two-layer documentation standard (high-level docstrings plus inline "why" comments) and the native docstring tool for C/C++, Java/Kotlin, JavaScript/TypeScript, Python, Go, SQL (including Alembic Mako templates), Shell, YAML/Ansible, CMake, Configuration files (INI, dotenv, Nginx), and Common Lisp/Scheme. This standard takes precedence over the general instruction to avoid adding comments, and is mandatory only for newly generated code; pre-existing code is documented or commented only when explicitly requested. Project agent instructions and established local convention take precedence over the defaults documented here. Trigger on "document this", "add docstrings", "code comments", "JSDoc", "Godoc", "Javadoc", "Doxygen", "docstring", "KDoc", or "TSDoc".
license: MIT
metadata:
  audience: maintainers
  workflow: documentation
---

# Code Documentation Standards

Every contribution must follow these two layers of documentation:

1. **High-Level Docstrings:** Structured blocks used to automatically generate public-facing API documentation.
2. **Inline Comments:** Line-by-line or block explanations inside the codebase detailing *how* and *why* complex logic is implemented.

## 0. Scope and Precedence

**Precedence:** This standard overrides the general agent instruction to avoid adding comments. When this document and a core instruction disagree about comments or docstrings, this document wins.

**Scope:** The standard is mandatory **only for newly generated code**: new files, new functions, new classes, new methods, new modules, and new SQL queries.

**Pre-existing code:** Existing code is never retrofitted proactively. Documenting or commenting already-existing code is permitted **only when the user explicitly asks for it**. Code you did not write stays as-is otherwise, even when it does not meet this standard.

**Project Precedence:** These guidelines set the floor, not the ceiling. Where a project has an established convention, or ships its own agent instructions (`AGENTS.md`, `CLAUDE.md`, or equivalent) requiring a different style, follow the project. Concretely: read the instruction files the project ships before writing anything, and match the convention already used in the file you are editing. Apply this skill's stated default only when the project has no established convention for that case.

**Correcting Existing Code:** Pre-existing code stays as it is unless the user explicitly asks for it to be brought into compliance. When they do ask, the full standard applies to the code in scope, including rules that existing comments currently break.

## 1. General Principles

### High-Level Docstrings

Every public class, method, function, and module must include a structured docstring matching the language's native ecosystem. At a minimum, it must capture:

- **Summary:** A concise, single-line description of purpose.
- **Inputs/Parameters:** Name, type, and brief description.
- **Outputs/Returns:** Type and description of the output.
- **Exceptions/Errors:** Explicit conditions that trigger failures.

Formats that are not functions, such as YAML, CMake, INI or Nginx configuration, cannot express parameters, returns or exceptions. Their required elements are declared in their own section of section 2, and this four-part contract does not apply to them.

**Length Limits for Entity Descriptions:** The descriptive prose for any entity (type definitions, structs, classes, function definitions, global variables, and similar) must not exceed **80 words**. This limit applies to the main description only and **excludes** individual parameter descriptions, return value descriptions, member variable descriptions, and subdivision explanations (e.g., `@param`, `@return`, `@throws` blocks), each of which carries its own limit of **16 words**.

### Inline Comments

- **Explain the "Why":** Focus on the rationale behind a design choice, performance trade-off, or workaround.
- **Do Not State the Obvious:** Avoid comments like `x = x + 1; // increments x`.
- **Placement:** Place inline comments directly above the code line they reference, aligned to the same indentation level.
- **Length Limit:** Inline comments must not exceed **20 words**.

## 2. Language-Specific Guidelines

### C / C++

- **Docstring Tool:** Doxygen
- **Format:** Use `/** ... */` block comments with explicit structural commands.
- **Rules:** Place interface documentation in the header files (`.h`/`.hpp`) and complex inline reasoning in the source files (`.c`/`.cpp`).

```cpp
#include <cmath>

/**
 * @brief Computes the Euclidean distance between two points.
 *
 * @param x1 Coordinate X of the first point.
 * @param y1 Coordinate Y of the first point.
 * @param x2 Coordinate X of the second point.
 * @param y2 Coordinate Y of the second point.
 * @return The straight-line distance as a double.
 */
double compute_distance(double x1, double y1, double x2, double y2) {
    // Short-circuiting keeps the result an exact 0.0 for coincident points,
    // which callers compare with == before normalising the vector.
    if (x1 == x2 && y1 == y2) {
        return 0.0;
    }

    return std::sqrt(std::pow(x2 - x1, 2) + std::pow(y2 - y1, 2));
}
```

### Java / Kotlin

- **Docstring Tool:** Javadoc (Java) / KDoc (Kotlin)
- **Format:** Standard `/** ... */` blocks using HTML/Markdown tags appropriately.
- **Rules:** Document nullability explicitly in Java via annotations, and natively via KDoc structure in Kotlin.

```kotlin
/**
 * Validates the structural shape of a JWT session token.
 *
 * The signature is not checked here; callers must verify it separately.
 *
 * @param token The raw compact JWT from the request header, or null.
 * @return True if the token carries the three dot-separated JWT segments.
 * @throws InvalidTokenException If the token is non-blank but not a 3-part JWT.
 */
fun verifySessionStructure(token: String?): Boolean {
    // A missing header is an unauthenticated request rather than a malformed
    // token, so it returns false instead of raising and forcing a 500.
    if (token.isNullOrBlank()) return false

    val parts = token.split(".")
    if (parts.size != 3) {
        throw InvalidTokenException("Malformed token format structure.")
    }
    return true
}
```

### JavaScript / TypeScript

- **Docstring Tool:** JSDoc / TSDoc
- **Format:** Block comments prefixing parameters with types (JSDoc) or omitting redundant types when implicitly declared by TypeScript compiler context (TSDoc).

```typescript
/**
 * Flattens a double-nested array of strings into a single sequence.
 *
 * @param matrix - Two-dimensional array of string values.
 * @returns Every string from every row, in row-major order.
 * @throws TypeError If a row is not an array.
 */
function flattenMatrix(matrix: string[][]): string[] {
  // `flat(1)` is depth-limited, so a stray third level raises loudly instead of
  // silently flattening a shape the caller never validated.
  return matrix.flat(1);
}
```

### Python

- **Docstring Tool:** Sphinx / MkDocs (Google Style)
- **Format:** Triple double quotes `"""..."""` placed immediately below the object definition statement.

```python
import json


def parse_payload(raw_data: str) -> dict:
    """Parses a JSON object out of a raw request body.

    Args:
        raw_data (str): Unparsed request body expected to hold a JSON object.

    Returns:
        dict: The decoded object. An empty or whitespace-only body yields {}.

    Raises:
        json.JSONDecodeError: If the body is non-empty but is not valid JSON.
    """
    # An absent body means "no filters applied" on this endpoint, so it maps to
    # an empty result set rather than a 400 that would break older clients.
    if not raw_data.strip():
        return {}

    return json.loads(raw_data)
```

### Go

- **Docstring Tool:** Godoc
- **Format:** Regular continuous block single-line comments `//` directly preceding the declared item name.
- **Rules:** The docstring comment sentence must start with the literal name of the defined component. This applies to new code. Existing comments that do not follow it are left alone unless the user asks for them to be brought into compliance, in which case the rule applies to those comments as well.

```go
// CalculateTotal applies a tax rate to a subtotal and rounds the result to cents.
func CalculateTotal(subtotal float64, taxRate float64) float64 {
	total := subtotal * (1.0 + taxRate)

	// Rounding is deferred to the end so intermediate values keep full precision.
	return math.Round(total*100) / 100
}
```

### SQL

- **Docstring Tool:** Embedded standard schema block header patterns.
- **Format:** Dash markers `--` or block comments `/* ... */`.
- **Rules:** Complex window analytical calculations, multi-layered index usage hints, or heavy table joins require explicit functional inline breakdowns. When a query is generated from an Alembic Mako template such as `script.py.mako`, comment the template rather than its generated output, and keep revision identifiers out of the prose.

```sql
/*
 * Procedure: RefreshUserActivitySummary
 * Purpose: Aggregates streaming events into day-level per-user snapshots.
 */
SELECT
    user_id,
    DATE_TRUNC('day', created_at) AS activity_date,
    COUNT(DISTINCT device_id) AS hardware_count
FROM telemetry_events
-- Filtering before the aggregate keeps retried sign-in attempts from
-- registering one device several times inside the same day bucket.
WHERE status = 'SUCCESS'
GROUP BY 1, 2;
```

### Common Lisp / Scheme

- **Docstring Tool:** Built-in Lisp Docstrings
- **Format:** Explicit strings inside macro or functional definitions.
- **Rules:** Inline comments use semicolons. Follow standard convention: `;` for inline line-ends, `;;` for code block indentation alignments, and `;;;` for file/global structural headers.

```lisp
(defun calculate-factorial (n)
  "Returns the factorial of N, treating any N below 1 as 1.

The 0! = 1 convention lets callers pass a count of zero without special-casing."
  (if (<= n 1)
      1
      (* n (calculate-factorial (- n 1)))))
```

### Shell / Bash

- **Docstring Tool:** Header comment block beneath the shebang line.
- **Format:** A block stating the purpose, the required environment variables, and the exit codes the script may return.
- **Rules:** Comment only what the code cannot express: terminal escape sequences, `exec` semantics, masking behaviour, and deliberate deviations from strict mode. Never restate a command, and never explain what a standard test or assignment already says.

```bash
#!/usr/bin/env bash
# Runs the behave suite against already-built services.
# Exit codes: 0 pass, 2 invalid password, 1 invalid project directory.
# Requires: TEST_ROOT_DATABASE_PASSWORD, TEST_PROJECTS_DIRECTORY_PATH.

# `read -s` suppresses the terminal echo, so the mask below is all the operator sees.
password=""
while IFS= read -r -s -n 1 char; do
    # 0177 octal is DEL, the byte a terminal sends for Backspace.
    if [[ "$char" == $'\177' ]]; then
        password="${password%?}"
        printf "\b \b"
    else
        password+="$char"
        printf "*"
    fi
done
```

### YAML / Ansible

- **Docstring Tool:** File header comment block using `#`.
- **Format:** A header block separated from the content by a row of `# ===` characters, stating the purpose, the inventory to use, and any prerequisite playbook.
- **Rules:** Document play ordering and idempotency decisions, because Ansible replays these files on every run. Never restate a module name or a variable assignment.

```yaml
# Install the ledger documentation gateway.
# Usage:
#   ANSIBLE_STRATEGY=debug ansible-playbook 0013-documentation_install.yml -i inventory_test.yml
# Idempotent: the imported playbook re-applies the same nginx and file state.
# =============================================================================
- name: Install ledger documentation
  ansible.builtin.import_playbook: 1016-gateway_documentation_install.yml
  vars:
    project: multiplos-ledger-docs
    delivery_path: /docs-cartorario-api
```

### CMake

- **Docstring Tool:** `#` comments in `CMakeLists.txt`. `CMakePresets.json` accepts no comments, so document preset intent in `CMakeLists.txt` instead.
- **Format:** A comment above each target, option or `find_package` call.
- **Rules:** State why the target exists and which toolchain, standard or platform constraint it encodes. Never restate the command, so never write "adds an executable" above `add_executable`.

```cmake
# C++20 is required by the yaml-cpp and fmt versions pinned in vcpkg.json.
set(CMAKE_CXX_STANDARD 20)

# MUSL_FULL_STATIC is opt-in because a static glibc build breaks the GDB
# workflow described in README.md, so only release builds set it.
if("$ENV{MUSL_FULL_STATIC}" STREQUAL "true")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a" ".lib")
endif()
```

### Configuration Files (INI, dotenv, Nginx)

- **Docstring Tool:** Native comment syntax per format: `;` or `#` for INI and dotenv, `#` for Nginx.
- **Format:** A comment on the line above each non-obvious key or directive.
- **Rules:** State the effect of the key and the consequence of changing it. Never restate the key name, and never document a secret value or a live credential.

```nginx
server {
        # The document root is created empty by the deploy playbook and must
        # hold an index.html, otherwise nginx answers 403 on this vhost.
        root /var/www/ledger.multiplos.cards;

        index index.html index.htm;

        # Strips the gateway prefix because the Go service registers its own
        # /api/... routes and must not receive it.
        location ~* /user-management {
            rewrite /user-management/(.*) /$1 break;
            proxy_pass http://127.0.0.1:8080;
        }
}
```

## 3. Documentation Checklist for Reviewers

Before approving a pull request, and after editing this document, ensure the following criteria are met:

- [ ] **Completeness:** Do all new functions and classes have valid language-compliant docstrings?
- [ ] **Type Accuracy:** Do documented signatures match compiler/runtime constraints perfectly?
- [ ] **No Duplications:** Are inline comments adding structural "Why" value rather than repeating simple commands?
- [ ] **Ecosystem Compliance:** Does the file use the designated documentation tool structure for its native language variant?
- [ ] **Example Integrity:** Do the examples in section 2 satisfy section 1 and the inline comment rule, with no comment restating its adjacent line, no rationale the code does not implement, and no documented parameter the signature does not declare?
