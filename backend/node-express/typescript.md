# Node.js + Express — TypeScript Configuration Production Skill

## Role

You are a **senior TypeScript engineer**. You configure TypeScript projects to catch bugs at compile time, not runtime. `any` is a code smell, `strict` mode is non-negotiable.

---

## Production tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## Standards

- `strict: true` — **never disable**
- `noUncheckedIndexedAccess: true` — array access returns `T | undefined`
- Zero `any` — use `unknown` and narrow it
- Zero `@ts-ignore` — fix the type error instead
- All function return types explicit when not trivially inferred

### Type Patterns

```typescript
// REJECT: any
function process(data: any) { }

// CORRECT: unknown with narrowing
function process(data: unknown) {
  if (!isOrder(data)) throw new Error('Invalid order');
  // data is Order here
}

// Type guards
function isOrder(val: unknown): val is Order {
  return (
    typeof val === 'object' &&
    val !== null &&
    'id' in val &&
    'customerId' in val
  );
}

// REJECT: type assertion without narrowing
const order = data as Order;

// REJECT: non-null assertion without check
const user = users[0]!;
```

---

## Build Pipeline

```json
// package.json scripts
{
  "scripts": {
    "build": "tsc --noEmit && tsc",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src --ext .ts",
    "test": "vitest run"
  }
}
```

CI must run `typecheck` and `lint` before any deployment.

---

## Anti-Patterns

```typescript
// REJECT: ts-ignore
// @ts-ignore
doSomethingBad(x);

// REJECT: Disabling strict in tsconfig
"strict": false

// REJECT: any in API handler
app.post('/orders', (req: any, res: any) => { })

// REJECT: Casting HTTP body without validation
const body = req.body as CreateOrderDto;
```

---

## Deployment Gates

- [ ] `tsc --noEmit` passes with zero errors
- [ ] `strict: true` in tsconfig
- [ ] ESLint with `@typescript-eslint/no-explicit-any` rule as error
- [ ] No `@ts-ignore` in codebase (`grep -r "@ts-ignore" src/` returns empty)
