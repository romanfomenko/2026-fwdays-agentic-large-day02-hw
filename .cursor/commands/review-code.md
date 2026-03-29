# /review-code

Perform a thorough code review of the currently open file (or the file path provided as an argument).

## Usage

```text
/review-code [file-path]
```

- Without argument: reviews the active editor file.
- With argument: reviews the specified file path.

## What This Command Does

Analyze the target file and produce a structured review covering:

### 1. Correctness
- Logic errors, off-by-one mistakes, incorrect conditionals.
- Edge cases that are not handled (null/undefined, empty arrays, concurrent access).
- Race conditions or async/await anti-patterns.

### 2. Security (rule 03-security)
- `dangerouslySetInnerHTML` with unsanitized input.
- `eval()`, `new Function()`, or string-based `setTimeout`/`setInterval`.
- Open-redirect risks from user-supplied URLs.
- Unvalidated data from clipboard, localStorage, or file imports.
- `<a target="_blank">` without `rel="noopener noreferrer"`.

### 3. TypeScript Quality (rule 01-typescript-best-practices)
- Missing return types on exported functions.
- Use of `any` — suggest precise types or `unknown` with guards.
- Non-null assertions (`!`) that could be replaced with safe alternatives.

### 4. React Patterns (rule 02-react-components — .tsx files only)
- Multiple components in one file.
- Missing prop interface or incorrect naming convention.
- Hooks called conditionally or inside loops.
- Missing `aria-label` / accessibility attributes on interactive elements.

### 5. Performance (rule 06-performance)
- New object/array literals created inside hot loops or render paths.
- Missing `useCallback`/`useMemo` on props passed to memoized children.
- Full lodash package imports.

### 6. Import Order & Module Boundaries (rule 05-imports)
- Import groups out of order.
- Cross-package relative imports.
- Missing `import type` for type-only imports.

### 7. Test Coverage
- If the file is a source file, are there corresponding tests?
- Untested branches or exported functions without a test.

## Output Format

Return the review as a Markdown checklist grouped by the categories above.
For each finding include:
- **Severity**: `critical` | `major` | `minor` | `suggestion`
- **Location**: file path + line number
- **Issue**: what is wrong
- **Fix**: concrete recommended change (code snippet if helpful)

End the review with a one-paragraph **Summary** and an overall rating: `Approved` | `Approved with suggestions` | `Changes requested`.

## Example

```
/review-code packages/excalidraw/src/renderer/renderScene.ts
```
