### Code Documentation Standards

This document defines the mandatory standards for documenting functions, data structures, classes, and methods within the OpenCode ecosystem. Every contribution must follow these two layers of documentation: 

1. **High-Level Docstrings:** Structured blocks used to automatically generate public-facing API documentation.
2. **Inline Comments:** Line-by-line or block explanations inside the codebase detailing *how* and *why* complex logic is implemented.

### 1. General Principles

### High-Level Docstrings

Every public class, method, function, and module must include a structured docstring matching the language's native ecosystem. At a minimum, it must capture: 

* **Summary:** A concise, single-line description of purpose.
* **Inputs/Parameters:** Name, type, and brief description.
* **Outputs/Returns:** Type and description of the output.
* **Exceptions/Errors:** Explicit conditions that trigger failures.

### Inline Comments

* **Explain the "Why":** Focus on the rationale behind a design choice, performance trade-off, or workaround.
* **Do Not State the Obvious:** Avoid comments like x = x + 1; // increments x.
* **Placement:** Place inline comments directly above the code line they reference, aligned to the same indentation level.

### 2. Language-Specific Guidelines

### C / C++

* **Docstring Tool:** Doxygen
* **Format:** Use /** ... */ block comments with explicit structural commands.
* **Rules:** Place interface documentation in the header files (.h/.hpp) and complex inline reasoning in the source files (.c/.cpp).

cpp

/**
 * @brief Computes the Euclidean distance between two points.
 * 
 * @param x1 Coordinate X of the first point.
 * @param y1 Coordinate Y of the first point.
 * @return The straight-line distance as a double.
 */
double compute_distance(double x1, double y1, double x2, double y2) {
    // Check for identical points early to bypass square root processing overhead
    if (x1 == x2 && y1 == y2) {
        return 0.0;
    }
    
    // Standard Pythagorean theorem application
    return sqrt(pow(x2 - x1, 2) + pow(y2 - y1, 2));
}

Use code with caution.

### Java / Kotlin

* **Docstring Tool:** Javadoc (Java) / KDoc (Kotlin)
* **Format:** Standard /** ... */ blocks using HTML/Markdown tags appropriately.
* **Rules:** Document nullability explicitly in Java via annotations, and natively via KDoc structure in Kotlin.

kotlin

/**
 * Authenticates a session token against the authorization cluster.
 * 
 * @param token The raw JWT string from the client header.
 * @return True if token is active and signature verified; false otherwise.
 * @throws InvalidTokenException If the token structurally violates JWT standards.
 */
fun verifySession(token: String?): Boolean {
    // Return early if payload missing to avoid parsing performance penalties
    if (token.isNullOrBlank()) return false

    // Split token fragments securely without throwing out-of-bounds exceptions
    val parts = token.split(".")
    if (parts.size != 3) {
        throw InvalidTokenException("Malformed token format structure.")
    }
    return true
}

Use code with caution.

### JavaScript / TypeScript

* **Docstring Tool:** JSDoc / TSDoc
* **Format:** Block comments prefixing parameters with types (JSDoc) or omitting redundant types when implicitly declared by TypeScript compiler context (TSDoc).

typescript

/**
 * Flattens a multi-dimensional array schema.
 * 
 * @param matrix - A double-nested array containing string payloads.
 * @returns A single continuous sequence of strings.
 */
function flattenMatrix(matrix: string[][]): string[] {
  // Use native flat implementation but fallback manually if compatibility matrix demands it
  return matrix.reduce((accumulator, targetRow) => {
    // Merge individual rows sequentially into the base array accumulator
    return accumulator.concat(targetRow);
  }, [] as string[]);
}

Use code with caution.

### Python

* **Docstring Tool:** Sphinx / MkDocs (Google Style)
* **Format:** Triple double quotes """...""" placed immediately below the object definition statement.

python

def parse_payload(raw_data: str) -> dict:
    """Extracts internal dictionary metadata configuration configurations.

    Args:
        raw_data (str): Unparsed structural text package raw string feed.

    Returns:
        dict: Fully mapped clean schema dictionary block structure payload.
    """
    # Guard clause: stop execution to prevent JSON library decoding exceptions on empty spaces
    if not raw_data.strip():
        return {}
        
    return json.loads(raw_data)

Use code with caution.

### Go

* **Docstring Tool:** Godoc
* **Format:** Regular continuous block single-line comments // directly preceding the declared item name.
* **Rules:** The docstring comment sentence must start with the literal name of the defined component.

go

// CalculateTotal adds subtotal balances along with localized sales tax allocations.
func CalculateTotal(subtotal float64, taxRate float64) float64 {
	// Apply exact currency precision rounding constraints using fixed base logic fractions
	total := subtotal * (1.0 + taxRate)
	
	// Convert result directly to standard cent boundaries to prevent floating point drift leakage
	return math.Round(total*100) / 100
}

Use code with caution.

### SQL

* **Docstring Tool:** Embedded standard schema block header patterns.
* **Format:** Dash markers -- or block comments /* ... */.
* **Rules:** Complex window analytical calculations, multi-layered index usage hints, or heavy table joins require explicit functional inline breakdowns.

sql

/*
 * Procedure: RefreshUserActivitySummary
 * Purpose: Aggregates chronological streaming events records into day-level snapshots.
 */
SELECT
    user_id,
    -- Group event tracking sequences into strict 24-hour window ranges
    DATE_TRUNC('day', created_at) AS activity_date,
    -- Extract uniquely isolated device logins avoiding duplicate totals accumulation
    COUNT(DISTINCT device_id) AS hardware_count
FROM telemetry_events
WHERE status = 'SUCCESS' -- Filter failed authorization handshakes out immediately to preserve accuracy
GROUP BY 1, 2;

Use code with caution.

### Common Lisp / Scheme

* **Docstring Tool:** Built-in Lisp Docstrings
* **Format:** Explicit strings inside macro or functional definitions.
* **Rules:** Inline comments use semicolons. Follow standard convention: ; for inline line-ends, ;; for code blocks indentation alignments, and ;;; for file/global structural headers.

lisp

(defun calculate-factorial (n)
  "Computes the factorial value for any positive integer N safely."
  ;; Guard checking for negative parameter domain validation boundaries
  (if (<= n 1)
      1
      ;; Recurse calculation dynamically downward accumulating processing values
      (* n (calculate-factorial (- n 1)))))

Use code with caution.

### Documentation Checklist for Reviewers

Before approving a Pull Request, ensure the following criteria are met: 

* [ ] **Completeness:** Do all new functions and classes have valid language-compliant docstrings?
* [ ] **Type Accuracy:** Do documented signatures match compiler/runtime constraints perfectly?
* [ ] **No Duplications:** Are inline comments adding structural "Why" value rather than repeating simple commands?
* [ ] **Ecosystem Compliance:** Does the file use the designated documentation tool structure for its native language variant?
