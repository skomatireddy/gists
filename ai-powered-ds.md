# Making Design System AI-Compatible

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [Documentation Architecture](#2-documentation-architecture)
3. [Writing Component Docs That Work](#3-writing-component-docs-that-work)
4. [Handling Compound Components](#4-handling-compound-components)
5. [Design Tokens](#5-design-tokens)
6. [Anti-Pattern Reference](#6-anti-pattern-reference)
7. [Copilot-Specific Configuration](#8-copilot-specific-configuration)

---

## 1. The Problem

When developers use Copilot to build with our design system, three things go wrong repeatedly:

**Hallucinated APIs.** Copilot invents props that don't exist. A developer types `<Button` and Copilot suggests `loading={true}` — a prop we've never shipped. The developer accepts it, the app compiles (it's just an unknown HTML attribute), and nobody notices until QA.

**Made-up tokens.** Copilot reaches for raw Tailwind (`bg-blue-500`, `text-gray-700`) instead of our semantic tokens (`bg-primary`, `text-muted-foreground`). The result looks right but breaks when we change themes.

**Wrong composition.** Copilot doesn't know that our `<Dialog>` requires `DialogContent` inside `DialogOverlay`, or that `DialogTitle` is mandatory for accessibility. It assembles the pieces wrong, or skips required children.

The root cause is that Copilot works from two sources: its training data (generic shadcn patterns from the internet) and the files open in your editor. If our DS documentation isn't in the repo in a format Copilot can use, it falls back to guessing.

---

## 2. Documentation Architecture

### Where the docs live

Create a `.github/` instruction file and an `.ai/` directory at the repo root. The `.github/` file is what Copilot reads automatically. The `.ai/` directory is structured reference that developers (and Copilot, via `#file` references) can pull from.

```
design-system/
├── .github/
│   └── copilot-instructions.md      # Copilot reads this automatically
├── .ai/
│   ├── overview.md                   # DS principles, 1 page
│   ├── tokens/
│   │   ├── colors.md
│   │   ├── spacing.md
│   │   ├── typography.md
│   │   └── tokens.json               # W3C DTCG format
│   ├── components/
│   │   ├── index.md                  # Component catalog
│   │   ├── button.md
│   │   ├── dialog.md                 # Compound component example
│   │   ├── form.md
│   │   └── ... (one per component)
│   ├── patterns/
│   │   ├── forms.md                  # Composed multi-component patterns
│   │   ├── layouts.md
│   │   └── data-tables.md
│   └── rules/
│       └── do-not.md                 # Anti-patterns with real examples
```

### Why this structure

Copilot in VS Code gets context from three places: the file you're editing, other open tabs, and `.github/copilot-instructions.md`.

- `.github/copilot-instructions.md` must be concise and self-contained — it's always loaded but has limited space.
- `.ai/` component docs are useful when a developer explicitly opens them in a tab, or references them with `#file` in Copilot Chat. They're also useful for human onboarding, code review, and any future tool migration.
- Don't put everything in one giant file. Copilot handles focused, relevant context better than a wall of text.

---

## 3. Writing Component Docs That Work

Each component gets one markdown file in `.ai/components/`. The format must be rigid and consistent across every component — Copilot pattern-matches, so consistency matters more than cleverness.

### Template for atomic components

````markdown
# Button

## Import
```tsx
import { Button } from "@/components/ui/button"
```

## Props
| Prop | Type | Default | Required |
|------|------|---------|----------|
| variant | "default" \| "destructive" \| "outline" \| "secondary" \| "ghost" \| "link" | "default" | no |
| size | "default" \| "sm" \| "lg" \| "icon" | "default" | no |
| asChild | boolean | false | no |
| disabled | boolean | false | no |

**These are the only props.** Do not use any props not listed above.

## When to use which variant
- `default` — primary actions: Save, Submit, Continue
- `destructive` — irreversible actions: Delete, Remove (always pair with confirmation)
- `outline` — secondary actions alongside a primary button
- `ghost` — tertiary actions, toolbar buttons, navigation
- `link` — inline text-level actions

## Examples
```tsx
// Primary action
<Button>Save changes</Button>

// Destructive with confirmation pattern
<AlertDialog>
  <AlertDialogTrigger asChild>
    <Button variant="destructive">Delete account</Button>
  </AlertDialogTrigger>
  {/* ... confirmation dialog */}
</AlertDialog>

// Icon button (accessible)
<Button variant="ghost" size="icon">
  <Settings className="h-4 w-4" />
  <span className="sr-only">Settings</span>
</Button>

// Loading state — no "loading" prop exists, use this pattern
<Button disabled>
  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
  Saving...
</Button>

// As a link
<Button asChild>
  <Link href="/dashboard">Go to dashboard</Link>
</Button>
```

## Do NOT
```tsx
// ❌ There is no "loading" prop
<Button loading>Save</Button>

// ❌ Don't use raw Tailwind colors — use variant
<Button className="bg-blue-500 text-white">Save</Button>

// ❌ Don't wrap in <a> — use asChild
<a href="/home"><Button>Home</Button></a>

// ❌ Don't make icon buttons without accessible labels
<Button size="icon"><Trash className="h-4 w-4" /></Button>
```

## Accessibility
- Icon-only buttons: always add `<span className="sr-only">Label</span>`
- Destructive actions: always pair with a confirmation dialog
- Disabled buttons: include visible explanation of why the action is unavailable
````

---

## 4. Handling Compound Components

Compound components (Dialog, DropdownMenu, Sheet, AlertDialog, Form) are where Copilot fails most often with shadcn. It doesn't know which sub-components are required, what the nesting order is, or which parts are mandatory for accessibility.

### Template for compound components

````markdown
# Dialog

## Import
```tsx
import {
  Dialog,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogDescription,
  DialogFooter,
  DialogClose,
} from "@/components/ui/dialog"
```

## Required structure
The assembly order below is mandatory. Omitting `DialogTitle` breaks screen readers. `DialogDescription` is strongly recommended.

```tsx
<Dialog>
  <DialogTrigger asChild>
    {/* The element that opens the dialog */}
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Required — always include</DialogTitle>
      <DialogDescription>Recommended — describe the purpose</DialogDescription>
    </DialogHeader>

    {/* Your content here */}

    <DialogFooter>
      <DialogClose asChild>
        <Button variant="outline">Cancel</Button>
      </DialogClose>
      <Button>Confirm</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

## Sub-component reference
| Component | Required | Purpose |
|-----------|----------|---------|
| Dialog | yes | Root — manages open/close state |
| DialogTrigger | yes (unless controlled) | Element that opens the dialog |
| DialogContent | yes | The overlay + panel |
| DialogHeader | yes | Container for title and description |
| DialogTitle | yes (a11y) | Accessible name — screen readers read this |
| DialogDescription | recommended | Accessible description |
| DialogFooter | no | Action buttons container, aligns right |
| DialogClose | no | Wraps a button to close the dialog |

## Controlled usage
```tsx
const [open, setOpen] = useState(false);

<Dialog open={open} onOpenChange={setOpen}>
  {/* DialogTrigger is optional when controlled */}
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Edit profile</DialogTitle>
    </DialogHeader>
    {/* form content */}
  </DialogContent>
</Dialog>
```

## Do NOT
```tsx
// ❌ Missing DialogTitle — breaks screen readers
<Dialog>
  <DialogContent>
    <p>Are you sure?</p>
  </DialogContent>
</Dialog>

// ❌ DialogContent outside Dialog root
<Dialog>
  <DialogTrigger>Open</DialogTrigger>
</Dialog>
<DialogContent>...</DialogContent>

// ❌ Direct onClick to close — use DialogClose or onOpenChange
<DialogContent>
  <Button onClick={() => setIsOpen(false)}>Cancel</Button>
</DialogContent>
```
````

### Key difference from atomic docs

The **"Required structure"** section with the full skeleton is critical. This is the snippet Copilot will extrapolate from. Make it complete, realistic, and correct — developers will scaffold from this pattern for every dialog they build.

---

## 5. Design Tokens

### Use the W3C DTCG format

Don't invent a custom token JSON schema. The W3C Design Tokens Community Group spec is supported by Style Dictionary, Tokens Studio, Figma Variables, and an increasing number of tools. Use it so the file is useful beyond just AI:

```json
{
  "color": {
    "primary": {
      "$value": "hsl(222.2 47.4% 11.2%)",
      "$type": "color",
      "$description": "Primary actions, active states, focus rings"
    },
    "primary-foreground": {
      "$value": "hsl(210 40% 98%)",
      "$type": "color",
      "$description": "Text on primary-colored backgrounds"
    },
    "destructive": {
      "$value": "hsl(0 84.2% 60.2%)",
      "$type": "color",
      "$description": "Errors, destructive actions, danger states"
    },
    "muted": {
      "$value": "hsl(210 40% 96.1%)",
      "$type": "color",
      "$description": "Subtle backgrounds, disabled states"
    },
    "muted-foreground": {
      "$value": "hsl(215.4 16.3% 46.9%)",
      "$type": "color",
      "$description": "Secondary text, placeholder text, subtle labels"
    }
  },
  "spacing": {
    "$description": "Based on a 4px (0.25rem) scale. Use Tailwind classes: p-1 = 4px, p-2 = 8px, etc.",
    "unit": {
      "$value": "0.25rem",
      "$type": "dimension"
    }
  },
  "radius": {
    "default": {
      "$value": "0.5rem",
      "$type": "dimension",
      "$description": "Default border radius. Use Tailwind: rounded-md"
    },
    "lg": {
      "$value": "0.75rem",
      "$type": "dimension",
      "$description": "Cards, dialogs. Use Tailwind: rounded-lg"
    }
  }
}
```

### The markdown mirror

Ship `tokens/colors.md` for the human-readable version (and for Copilot context when a developer opens it):

```markdown
# Color Tokens

Use CSS variables via Tailwind. Never use raw Tailwind colors like bg-blue-500.

## Usage mapping

| Need | CSS Variable | Tailwind Class | When to use |
|------|-------------|----------------|-------------|
| Primary action | --primary | bg-primary text-primary | Buttons, links, focus rings |
| Primary text-on-color | --primary-foreground | text-primary-foreground | Text on bg-primary |
| Danger/error | --destructive | bg-destructive text-destructive | Delete, error states |
| Subtle background | --muted | bg-muted | Hover states, code blocks |
| Secondary text | --muted-foreground | text-muted-foreground | Descriptions, timestamps |
| Card surface | --card | bg-card | Card backgrounds |
| Borders | --border | border-border | All borders, dividers |
| User input bg | --input | bg-input | Input field backgrounds |

**These are the only color tokens.** If you need a color not listed here, propose it — do not use raw Tailwind colors as a workaround.
```
---

### Component manifest

Add a `components.json` manifest (shadcn already scaffolds one — extend it):

```json
{
  "components": [
    {
      "name": "Button",
      "import": "@/components/ui/button",
      "doc": ".ai/components/button.md",
      "type": "atomic"
    },
    {
      "name": "Dialog",
      "import": "@/components/ui/dialog",
      "doc": ".ai/components/dialog.md",
      "type": "compound",
      "subcomponents": [
        "DialogTrigger", "DialogContent", "DialogHeader",
        "DialogTitle", "DialogDescription", "DialogFooter", "DialogClose"
      ]
    }
  ]
}
```

This doesn't directly affect Copilot, but it's the source of truth for generating docs, validating imports, and onboarding new tools in the future.

---

## 6. Anti-Pattern Reference

Create `.ai/rules/do-not.md` as a centralized reference of common AI mistakes. This file is useful both as Copilot context and as a code review checklist.

### Real examples of Copilot hallucinations

````markdown
# Design System Anti-Patterns

Common mistakes that AI tools generate. Watch for these in code review.

## Invented props

Copilot frequently suggests props that don't exist in our components:

```tsx
// ❌ WRONG — none of these props exist
<Button loading>Save</Button>
<Button color="blue">Save</Button>
<Button rounded>Save</Button>
<Input error="Name is required" />
<Card hoverable>Content</Card>
<Badge count={5}>Notifications</Badge>

// ✅ CORRECT equivalents
<Button disabled>
  <Loader2 className="mr-2 h-4 w-4 animate-spin" /> Save
</Button>
<Button variant="default">Save</Button>    {/* variant, not color */}
<Input className="border-destructive" />     {/* class, not error prop */}
<Card className="hover:shadow-md">Content</Card>
<Badge>5</Badge>
```

## Raw Tailwind instead of tokens

```tsx
// ❌ WRONG — raw colors bypass theming
<div className="bg-white text-gray-900 border-gray-200">
<p className="text-blue-600">Click here</p>
<span className="text-gray-500">Secondary info</span>

// ✅ CORRECT — semantic tokens
<div className="bg-background text-foreground border-border">
<p className="text-primary">Click here</p>
<span className="text-muted-foreground">Secondary info</span>
```

## Broken compound component assembly

```tsx
// ❌ WRONG — missing DialogTitle (a11y violation)
<Dialog>
  <DialogContent>
    <p>Are you sure?</p>
  </DialogContent>
</Dialog>

// ❌ WRONG — Select items outside SelectContent
<Select>
  <SelectTrigger />
  <SelectItem value="a">A</SelectItem>   {/* Must be inside SelectContent */}
</Select>

// ❌ WRONG — Form fields without FormField wrapper
<form>
  <Label>Name</Label>
  <Input />                              {/* Won't connect to react-hook-form */}
</form>
```

## Direct Radix imports

```tsx
// ❌ WRONG — bypasses our wrappers and styling
import * as Dialog from "@radix-ui/react-dialog"

// ✅ CORRECT — use our components
import { Dialog, DialogContent } from "@/components/ui/dialog"
```
````

---


```

### TypeScript strictness

Use strict `VariantProps` from CVA and narrow union types. The tighter the types, the less Copilot can hallucinate successfully — if `variant="blue"` won't compile, Copilot learns to stop suggesting it.

Make sure `tsconfig.json` has `"strict": true`. Copilot respects TypeScript errors in the current file and adjusts its suggestions accordingly.
```


---

## 7. Copilot-Specific Configuration

### `.github/copilot-instructions.md`

This is the one file Copilot reads automatically for every prompt and completion in the repo. It has a limited context budget, so keep it focused on rules rather than reference material.

```markdown
# Copilot Instructions

This project uses a shadcn-based design system. Follow these rules strictly.

## Import rules
- Import UI components from `@/components/ui/`, never from `@radix-ui/*`
- Available components are listed in `.ai/components/index.md`

## Styling rules
- Use semantic color tokens only: bg-primary, text-muted-foreground, border-border, etc.
- Never use raw Tailwind colors: bg-blue-500, text-gray-700, border-red-300, etc.
- Spacing follows a 4px scale via Tailwind: p-1 (4px), p-2 (8px), p-4 (16px), etc.
- Border radius: rounded-md (default), rounded-lg (cards/dialogs)

## Component rules
- Button variants: "default", "destructive", "outline", "secondary", "ghost", "link"
- Button sizes: "default", "sm", "lg", "icon"
- There is no loading prop on Button — use disabled + Loader2 spinner
- Icon-only buttons must include <span className="sr-only">Label</span>
- Dialog must always include DialogTitle for accessibility
- Form fields must use FormField, FormItem, FormLabel, FormControl, FormMessage

## When unsure
- Check `.ai/components/<component>.md` for the full API
- Check `.ai/rules/do-not.md` for common mistakes to avoid
- If a prop isn't documented, it doesn't exist — do not invent props
```
