# 14 — Input/Output Processing

> **In plain English:** Before the agent starts working, the input (question + files) needs to be prepared. After the agent finishes, the output (answer) needs to be cleaned up and extracted. This page covers both sides.

---

## Input Processing

**File:** `src/utils/io_utils.py` — `process_input()`

When a task includes a file, MiroFlow detects the file type and adds helpful context to the task description:

| File Type | Extensions | What the agent is told |
|-----------|-----------|----------------------|
| Image | `.jpg`, `.png`, `.gif`, `.webp` | "An image file 'photo.jpg' is associated with this task." |
| Document | `.pdf`, `.docx`, `.pptx` | "A document file 'report.pdf' is associated with this task." |
| Spreadsheet | `.xlsx`, `.xls` | "A spreadsheet file 'data.xlsx' is associated with this task." |
| Audio | `.wav`, `.mp3`, `.m4a` | "An audio file 'recording.mp3' is associated with this task." |
| Archive | `.zip` | "An archive file 'bundle.zip' is associated with this task." |
| Text/JSON/HTML | `.txt`, `.json`, `.html` | "A text file 'config.json' is associated with this task." |

The file path is also converted to an absolute path so tools can find it.

> **Design rationale — Automatic file type detection:** The agent shouldn't have to guess what kind of file it's dealing with. By detecting the type upfront and telling the agent, we reduce the chance of the agent trying to read an image as text or vice versa. This is a small preprocessing step that prevents a whole class of errors.

---

## Hint Generation (Pre-Processing)

**File:** `src/utils/summary_utils.py` — `extract_hints()`

Before the main agent starts working, a separate LLM call can pre-analyze the question to identify:

- **Potential pitfalls** — "Be careful, 'starting with Co' could match multiple countries."
- **Multiple interpretations** — "Does 'first country' mean alphabetically or by row order?"
- **Formatting requirements** — "The answer should be a country name, not a code."
- **Possible mistakes** — "The question says 'XLSX' but the file might actually be CSV."

These hints are added to the agent's initial context.

> **Design rationale — "Think before you do":** Benchmark questions (especially GAIA) are deliberately tricky. By having a separate LLM call just for identifying pitfalls, the agent is better prepared before it starts its multi-step work. This is much cheaper than having the agent fail, realize the pitfall, and start over.

---

## Output Processing

### `\boxed{}` Extraction

**File:** `src/utils/io_utils.py` — `OutputFormatter`

The agent's final answer must be wrapped in `\boxed{}` (LaTeX notation). The `OutputFormatter` extracts this answer using a **balanced-brace counting algorithm** (not regex):

```python
def _extract_boxed_content(self, text: str) -> str:
    # Finds the LAST \boxed{...} in the text
    # Handles arbitrary nesting: \boxed{f(x) = \frac{a}{b+c}}
```

**Why not regex?** A regex like `\\boxed\{(.*?)\}` would fail on nested braces:
- Input: `\boxed{f(x) = \frac{a}{b+c}}`
- Regex would extract: `f(x) = \frac{a}{b+c` (wrong — stops at first `}`)
- Brace counting extracts: `f(x) = \frac{a}{b+c}` (correct — matches balanced braces)

> **Design rationale — Balanced-brace counting:** Mathematical and formatted answers often contain nested braces. A character-by-character brace counter is the only reliable way to extract the content. This is a small but critical detail that prevents answer extraction bugs across hundreds of benchmark evaluations.

### Tool Result Formatting

Tool results are truncated to **100,000 characters** (~25k tokens) to prevent context overflow. Error context is preserved to help the agent understand what went wrong.

### Final Answer Extraction

**File:** `src/utils/summary_utils.py` — `extract_gaia_final_answer()`

An optional post-processing step where a separate LLM call validates and cleans up the boxed answer. Useful for benchmarks where the answer format is strict (e.g., "just a number" or "just a country name").

> **Design rationale — LLM-based answer extraction:** Sometimes the agent wraps the answer in `\boxed{}` but includes extra formatting or explanation. A separate, focused LLM call that just extracts the clean answer is more reliable than regex-based cleanup.

---

**Next:** [15 — Design Decisions & Rationale](15-design-decisions.md) · **Previous:** [13 — Benchmark Evaluation](13-benchmarks.md)
