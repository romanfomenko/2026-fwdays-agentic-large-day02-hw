# /create-component

Scaffold a new React component that follows Excalidraw's coding standards and all project rules.

## Usage

```
/create-component <ComponentName> [--dir <relative-path>] [--with-test] [--with-story]
```

| Flag | Default | Description |
|---|---|---|
| `<ComponentName>` | required | PascalCase component name |
| `--dir` | `excalidraw-app/src/components` | Directory to create the component in |
| `--with-test` | off | Also create a `<ComponentName>.test.tsx` file |
| `--with-story` | off | Also create a `<ComponentName>.stories.tsx` file |

## What This Command Does

Generate the following files (example: `Button`):

### `Button.tsx`

```tsx
import React from "react";

interface ButtonProps {
  /** Visible label text (also used as accessible name). */
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: "primary" | "secondary" | "ghost";
}

export const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  disabled = false,
  variant = "primary",
}) => {
  return (
    <button
      type="button"
      aria-label={label}
      disabled={disabled}
      className={`btn btn--${variant}`}
      onClick={onClick}
    >
      {label}
    </button>
  );
};
```

### `Button.test.tsx` (when `--with-test`)

```tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { Button } from "./Button";

describe("Button", () => {
  it("renders the label", () => {
    render(<Button label="Submit" onClick={vi.fn()} />);
    expect(screen.getByRole("button", { name: "Submit" })).toBeInTheDocument();
  });

  it("calls onClick when clicked", () => {
    const handleClick = vi.fn();
    render(<Button label="Click me" onClick={handleClick} />);
    fireEvent.click(screen.getByRole("button"));
    expect(handleClick).toHaveBeenCalledOnce();
  });

  it("does not call onClick when disabled", () => {
    const handleClick = vi.fn();
    render(<Button label="Disabled" onClick={handleClick} disabled />);
    fireEvent.click(screen.getByRole("button"));
    expect(handleClick).not.toHaveBeenCalled();
  });
});
```

## Rules Applied During Generation

- **Rule 02** (react-components): Named export, one component per file, `<ComponentName>Props` interface, accessible `aria-label`.
- **Rule 01** (typescript): Explicit prop types, no `any`, boolean props prefixed with `is`/`has`/`disabled`.
- **Rule 03** (security): No `dangerouslySetInnerHTML`; no inline event-handler strings.
- **Rule 04** (testing): AAA pattern, accessible queries (`getByRole`), mocks reset after tests.
- **Rule 05** (imports): Correct import order — React first, then libraries, then relative.

## Post-Generation Checklist

After the files are created, confirm:

- [ ] `yarn test:typecheck` passes with the new file.
- [ ] `yarn test:code` passes (ESLint zero warnings).
- [ ] If `--with-test` was used, `yarn test:app --watch=false` passes.
- [ ] Component is exported from its parent `index.ts` if applicable.

## Example

```
/create-component ColorPicker --dir packages/excalidraw/src/components --with-test
```
