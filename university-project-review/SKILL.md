---
name: university-project-review
description: "Review a university student's full-stack web application project. Invoke ONLY when the user explicitly types /university-project-review — never auto-trigger this skill. Performs a comprehensive two-phase review: Phase 1 orients to the project structure and tech stack, Phase 2 evaluates 14 quality categories (component library, styling, data loading, env variables, REST API design, forms, frontend structure, database design, backend layers, auth, testing, logging, error handling, security). Writes evidence-based findings with ✅/⚠️/❌ status and concrete recommendations to REVIEW.md in the project root. Use this skill whenever the user explicitly requests a university project review."
---

# University Project Review

You are reviewing a student full-stack web application for a university software engineering course. The reviewer is a lecturer who needs to understand the project structure quickly and evaluate it across 12 quality categories.

Work through two phases in order. Do not skip categories. Write the complete report to `REVIEW.md` in the current working directory when done.

---

## Phase 1 — Project Orientation

Before evaluating anything, orient yourself to the project. Run these explorations:

**1. Directory structure (depth 2, excluding node_modules and .git):**

```bash
find . -maxdepth 2 -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/.next/*' | sort
```

**2. All package.json files (to detect tech stack):**

```bash
find . -name "package.json" -not -path '*/node_modules/*' -maxdepth 4 | while read f; do echo "=== $f ==="; cat "$f"; done
```

**3. README.md (if present):**

```bash
cat README.md 2>/dev/null
```

**4. Env template files:**

```bash
find . -maxdepth 3 \( -name ".env.example" -o -name ".env.sample" -o -name ".env.template" \) -not -path '*/node_modules/*' | xargs cat 2>/dev/null
```

**5. Monorepo config files:**

```bash
cat turbo.json pnpm-workspace.yaml nx.json 2>/dev/null
```

From this exploration, write the **Orientation section** covering:

- Project structure: how frontend/backend are organized, key directories
- Detected tech stack: framework versions, ORM, auth library, test runner
- Notable architectural observations visible from config/dependencies

The orientation is **descriptive only** — no pass/fail judgments.

---

## Phase 2 — 14 Category Reviews

For each category, run the specified exploration commands, then write a section with:

- **Status:** ✅ Good / ⚠️ Concerns / ❌ Issues
- **Findings:** Evidence-based narrative citing specific file paths and code snippets
- **Recommendations:** Numbered list of concrete, actionable items for the student

If a category cannot be assessed because the technology isn't present (e.g., no component library found), mark it **N/A — [brief reason]** rather than ❌.

Every finding must cite a specific file path or code snippet. Do not write generic observations.

Where applicable, assess component and logic size against these rules of thumb:

- Frontend component/page files should be under ~250 lines if they do not contain significant nested declarative markup. Files over ~400 lines should be clearly composed of smaller sub-components and hooks. Dense logic mixed with markup ("fat files") is a smell.
- Backend service/handlers should be under ~200-300 lines. Complex flows across multiple DB tables should be wrapped in transactions.

These are guidelines, not hard limits — context matters.

---

### 1. Component Library

Is a UI component library present? Is it used consistently, or do students bypass it with raw HTML elements or hand-rolled components?

```bash
# Detect component library in dependencies
grep -E "shadcn|@radix-ui|@mui|chakra-ui|mantine|@headlessui|@nextui" \
  $(find . -name "package.json" -not -path "*/node_modules/*") 2>/dev/null

# Find raw HTML interactive elements (potential library bypasses)
grep -rn "<button\b\|<input\b\|<select\b\|<textarea\b" \
  --include="*.tsx" --include="*.jsx" \
  . 2>/dev/null | grep -v "node_modules" | head -30

# Find where library components are imported
grep -rn "from.*shadcn\|from.*@radix\|from.*@mui\|from.*chakra\|from.*mantine" \
  --include="*.tsx" --include="*.jsx" . 2>/dev/null | grep -v node_modules | head -20
```

Check for:

- Component library present in package.json?
- Raw `<button>`, `<input>`, `<select>` used where library components should be used?
- Custom inline components that duplicate library components?

---

### 2. Styling

Are inline styles avoided? Is the chosen styling approach (CSS/BEM or Tailwind) used properly?

```bash
# Find inline style attributes
grep -rn 'style={{' --include="*.tsx" --include="*.jsx" --include="*.html" \
  . 2>/dev/null | grep -v node_modules | head -20

# Find Tailwind arbitrary value abuse
grep -rn '\[.*px\]\|\[.*rem\]\|\[.*em\]\|\[#[0-9a-fA-F]\{3,6\}\]' \
  --include="*.tsx" --include="*.jsx" . 2>/dev/null | grep -v node_modules | head -20

# Check CSS files for BEM usage
find . -name "*.css" -o -name "*.module.css" | grep -v node_modules | head -10 | xargs grep -l "__\|--" 2>/dev/null

# Check for !important overrides
grep -rn "important" --include="*.css" --include="*.tsx" . 2>/dev/null | grep -v node_modules | head -10
```

Check for:

- Inline `style={}` attributes in JSX?
- If CSS: BEM naming (`.block__element--modifier`)?
- If Tailwind: arbitrary value abuse, `!important` hacks?

---

### 3. Loading Data

Is TanStack Query or SWR used correctly, or do students fetch data directly in components?

```bash
# Find data fetching library usage
grep -rn "useQuery\|useSWR\|useInfiniteQuery\|useMutation\|queryClient" \
  --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -30

# Find direct fetch/axios calls in component files (potential bypasses)
grep -rn "await fetch(\|axios\.\|\.then(" \
  --include="*.tsx" --include="*.jsx" . 2>/dev/null | grep -v node_modules | head -20

# Check query key patterns
grep -rn "queryKey:" --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20

# Check for direct query/mutation instantiation without hooks
grep -rn "new QueryClient\|QueryClient()" \
  --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -10
```

Check for:

- TanStack Query or SWR present and used?
- `fetch`/`axios` called directly inside React components?
- Cache keys: are they stable arrays (e.g., `['users', userId]`) or fragile strings?
- Mutations and queries invoked through proper hooks?

---

### 4. Environment Variables

Are hardcoded values avoided? Is an env template committed?

```bash
# Check .gitignore protects .env files
grep -i "\.env" .gitignore 2>/dev/null

# Find hardcoded URLs and ports in source (not in tests or examples)
grep -rn "localhost:[0-9]\+\|http://[a-z]\|https://[a-z]" \
  --include="*.ts" --include="*.tsx" . 2>/dev/null \
  | grep -v "node_modules\|test\|spec\|example\|README\|\.env" | head -20

# Find hardcoded DB connection strings
grep -rn "postgresql://\|mysql://\|mongodb://" \
  --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|env\|example\|template" | head -10

# List .env* files in project root
find . -maxdepth 1 -name ".env*" 2>/dev/null

# Check env access patterns
grep -rn "process\.env\.\|Bun\.env\." \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20
```

Check for:

- `.env` in `.gitignore`?
- `.env.example` or `.env.sample` with placeholder values?
- Hardcoded `localhost:PORT`, connection strings, or API keys in source?

---

### 5. REST API Design

Are API routes resource-oriented with correct HTTP verbs, response codes, and well-defined contracts?

```bash
# Find route definitions (Hono, Elysia, Express, Fastify patterns)
grep -rn "\.get(\|\.post(\|\.put(\|\.patch(\|\.delete(\|\.route(" \
  --include="*.ts" . 2>/dev/null | grep -v "node_modules\|test\|spec" | head -40

# Find route files
find . \( -path "*/routes/*" -o -path "*/api/*" -o -path "*/controllers/*" \) \
  -name "*.ts" -not -path "*/node_modules/*" | head -20

# Check for verb-in-path anti-patterns
grep -rn '"/.*get\|"/.*create\|"/.*delete\|"/.*update\|"/.*fetch\|"/.*do' \
  --include="*.ts" . 2>/dev/null | grep -v "node_modules" | head -20

# Check for validation schemas (Zod, TypeBox, Valibot)
grep -rn "z\.object\|z\.string\|Type\.Object\|v\.object" \
  --include="*.ts" . 2>/dev/null | grep -v "node_modules" | head -20

# Check HTTP status codes returned
grep -rn "\.status(\|c\.status\|statusCode\|status:" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -30

# Check PUT handler bodies — should send/accept whole entity
grep -rn -A10 "\.put(" --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|test\|spec" | head -60

# Check PATCH handler bodies — should use partial update pattern
grep -rn -A10 "\.patch(" --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|test\|spec" | head -60

# Check for query/search parameter usage
grep -rn "query\.\|searchParams\.\|c\.req\.query\|req\.query" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20
```

Check for:

- Resource-oriented URLs (nouns, plural): `/users`, `/posts/:id`?
- Verbs in paths: `/getUser`, `/createPost`, `/doAction`?
- Correct HTTP verbs: GET for reads, POST for create, PUT/PATCH for update, DELETE for delete?
- **Response status codes** match semantics: 201 for created, 204 for no-content deletes, 400 for bad input, 404 for not found, 409 for conflict — not everything returning 200?
- **PUT idempotency**: PUT handlers should accept and store the complete resource representation, not partial fields. A PUT called twice with the same body should leave the resource unchanged (idempotent). Students often use PUT for partial updates — flag this.
- **PATCH semantics**: PATCH should apply a partial update. Acceptable patterns are a partial body (only fields to change) or JSON Merge Patch (RFC 7396). Flag any PATCH that replaces the whole resource (that's what PUT is for), or that uses PUT-style semantics.
- **Query parameters**: used for filtering, sorting, pagination (`?status=active&page=2`), not for actions or IDs that belong in the path?
- Request/response validation with Zod or similar?

---

### 6. Database

Is the schema sensible, ORM used properly, migrations managed, and blobs kept out of the DB?

```bash
# Find schema files
find . \( -name "schema.ts" -o -name "*.schema.ts" -o -name "drizzle.config.*" \) \
  -not -path "*/node_modules/*" | head -10

# Read the main schema file
find . -name "schema.ts" -not -path "*/node_modules/*" | head -1 | xargs cat 2>/dev/null

# Count migration files
find . \( -path "*/migrations/*.sql" -o -path "*/migrations/*.ts" \) \
  -not -path "*/node_modules/*" | wc -l

# List migration files (to check if managed properly)
find . \( -path "*/migrations/*.sql" -o -path "*/migrations/*.ts" \) \
  -not -path "*/node_modules/*" | sort | head -20

# Check for blob/image columns
grep -rn "bytea\|blob\|binary\|BYTEA\|BLOB\|BINARY\|base64\|photoData\|imageData" \
  --include="*.ts" --include="*.sql" . 2>/dev/null | grep -v node_modules | head -10

# Check for raw SQL (potential ORM bypass)
grep -rn "db\.execute\|sql\`\|\.query(" \
  --include="*.ts" . 2>/dev/null | grep -v "node_modules\|migration\|schema" | head -20
```

Check for:

- Schema has sensible relational structure with foreign keys?
- No `bytea`/`blob` columns storing images or files directly?
- Migration files exist and are versioned (not a single schema dump)?
- UUID vs integer IDs used consistently?
- Drizzle (or detected ORM) used for queries, not raw SQL strings?

---

### 7. Backend Design Patterns

Is there a clear separation between controller, service, and repository layers?

```bash
# Find layer-named files
find . \( -name "*.controller.ts" -o -name "*.service.ts" \
  -o -name "*.repository.ts" -o -name "*.repo.ts" \) \
  -not -path "*/node_modules/*" | sort

# Check directory structure for layer separation
find . -type d -not -path "*/node_modules/*" -not -path "*/.git/*" | sort

# Detect DB calls in route handlers (business logic leak)
# First: show all DB usage locations and their file paths so you can judge layer separation
grep -rn "db\.\|drizzle\." --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|schema\|migration\|config" | head -30

# Detect direct DB imports in controller/route files
grep -rn "from.*db\|from.*database\|from.*schema" \
  --include="*.controller.ts" --include="*.router.ts" . 2>/dev/null \
  | grep -v node_modules | head -10
```

Check for:

- Files named `*.controller.ts`, `*.service.ts`, `*.repository.ts` (or equivalent folders)?
- DB calls appearing directly in route handlers?
- Business logic (non-trivial computation, validation, rules) in controllers?

---

### 8. Auth

Is authentication using a known library? Is authorization (not just authentication) present?

```bash
# Detect known auth libraries
grep -E "better-auth|lucia|next-auth|@auth/|clerk|supabase|@hono/auth|kinde" \
  $(find . -name "package.json" -not -path "*/node_modules/*") 2>/dev/null

# Red flags: custom JWT/bcrypt without a framework
grep -rn "jsonwebtoken\|jwt\.sign\|jwt\.verify\|bcrypt\.hash\|bcrypt\.compare\|crypto\.createHash" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -10

# Find auth middleware / session checks
grep -rn "session\|getSession\|currentUser\|auth()\|requireAuth\|isAuthenticated" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20

# Check for authorization (not just authentication): ownership checks, role checks
grep -rn "userId.*===\|\.userId\|isAdmin\|hasPermission\|role.*===\|ownedBy\|forbidden\|403" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20
```

Check for:

- Known auth library in dependencies (BetterAuth, Lucia, Auth.js, Clerk, Supabase Auth)?
- `jsonwebtoken` + manual `sign`/`verify` without a framework (roll-your-own auth red flag)?
- Raw `bcrypt.hash`/`bcrypt.compare` wiring without a library?
- Authorization checks: does the code verify users can only access their own resources?

---

### 9. Testing

Are tests present, meaningful, and not all skipped?

```bash
# Count test files
find . \( -name "*.test.ts" -o -name "*.spec.ts" -o -name "*.test.tsx" \) \
  -not -path "*/node_modules/*" | wc -l

# List test files
find . \( -name "*.test.ts" -o -name "*.spec.ts" -o -name "*.test.tsx" \) \
  -not -path "*/node_modules/*" | sort

# Check for skipped or empty tests
grep -rn "test\.skip\|it\.skip\|xit\|xdescribe\|\.todo(" \
  --include="*.test.ts" --include="*.spec.ts" . 2>/dev/null | head -10

# Check test runner config (vitest, jest, or bun's built-in test runner)
find . \( -name "vitest.config.*" -o -name "jest.config.*" \) \
  -not -path "*/node_modules/*" | head -5 | xargs cat 2>/dev/null
# Check for bun test (no config file needed — look for test script in package.json)
grep -rn '"test"' --include="package.json" . 2>/dev/null | grep -v node_modules | head -5

# Check what's being tested (service vs trivial)
grep -rn "describe(" --include="*.test.ts" --include="*.spec.ts" . 2>/dev/null \
  | grep -v node_modules | head -20
```

Check for:

- At least some test files present?
- Tests cover service/business logic or integration (not just empty/snapshot tests)?
- Tests not all marked as `skip` or `todo`?
- A test runner configured (vitest, bun test, jest)? Note: bun test needs no config file — check package.json "test" script.

---

### 10. Logging & Monitoring

Is structured logging used? Are `console.log` calls avoided in production paths?

```bash
# Detect logging library
grep -E '"pino"|"winston"|"consola"|"@logtail"|"pino-http"' \
  $(find . -name "package.json" -not -path "*/node_modules/*") 2>/dev/null

# Count console.log in source (excluding tests)
grep -rn "console\.log\|console\.debug" --include="*.ts" --include="*.tsx" \
  . 2>/dev/null | grep -v "node_modules\|test\|spec\|\.test\.\|\.spec\." | wc -l

# Show console.log locations
grep -rn "console\.log\|console\.debug" --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|test\|spec" | head -20

# Check for sensitive data in logs
grep -rn "console\.log.*password\|console\.log.*token\|console\.log.*secret\|logger.*password" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -10
```

Check for:

- Structured logging library in dependencies (pino, winston, consola)?
- How many `console.log` calls in non-test source code?
- Any `console.log` of passwords, tokens, or other sensitive values?
- Errors caught and logged (not silently swallowed)?

---

### 11. Error Handling

Are HTTP status codes correct? Are frontend errors surfaced? Are catch blocks meaningful?

```bash
# Check for explicit HTTP status codes in route handlers
grep -rn "\.status(\|c\.status\|status:" --include="*.ts" . 2>/dev/null \
  | grep -v node_modules | head -20

# Find empty or near-empty catch blocks
grep -rn -A3 "} catch" --include="*.ts" --include="*.tsx" . 2>/dev/null \
  | grep -v "node_modules" | head -40

# Find error boundaries or error states in frontend
grep -rn "ErrorBoundary\|isError\|onError\|error:" --include="*.tsx" . 2>/dev/null \
  | grep -v node_modules | head -20

# Check for global error handler in backend
grep -rn "onError\|app\.use.*error\|errorHandler\|notFound" \
  --include="*.ts" . 2>/dev/null | grep -v node_modules | head -10
```

Check for:

- Routes return correct 4xx/5xx status codes, not always 200?
- Frontend shows error states (not just loading spinner that never resolves)?
- `catch(e) {}` or `catch(e) { return null }` silent swallows?
- Global error handler configured in backend framework?

---

### 12. Security

Is CORS configured? No secrets in repo? DB queries safe from injection?

```bash
# Check CORS configuration
grep -rn "cors\|CORS\|origin:" --include="*.ts" . 2>/dev/null \
  | grep -v node_modules | head -20

# Check for wildcard CORS (potential issue)
grep -rn "origin.*['\"]\\*['\"]" --include="*.ts" . 2>/dev/null \
  | grep -v node_modules | head -10

# Check for potential secrets in source code
grep -rn "api_key\s*=\|apiKey\s*=\|secret\s*=.*['\"][A-Za-z0-9+/]\{20,\}" \
  --include="*.ts" --include="*.tsx" . 2>/dev/null \
  | grep -v "node_modules\|test\|example\|placeholder\|\.env" | head -10

# Check for raw SQL string interpolation (injection risk)
grep -rn "sql.*\${.*}\|execute.*\`.*\${" --include="*.ts" . 2>/dev/null \
  | grep -v "node_modules\|migration" | head -10

# Check .gitignore covers sensitive files
grep -E "\.env|secret|credential|key" .gitignore 2>/dev/null
```

Check for:

- CORS explicitly configured (not defaulting to wildcard `*`)?
- No API keys, passwords, or tokens hardcoded in source?
- Database queries through ORM, not raw string interpolation?
- `.gitignore` covers `.env` files?

---

### 13. Forms

Are forms handled with a modern library and schema validation, rather than a tangle of `useState` calls?

```bash
# Detect form libraries
grep -E '"react-hook-form"|"@tanstack/react-form"|"formik"|"final-form"' \
  $(find . -name "package.json" -not -path "*/node_modules/*") 2>/dev/null

# Detect validation libraries
grep -E '"zod"|"yup"|"valibot"|"@hookform/resolvers"' \
  $(find . -name "package.json" -not -path "*/node_modules/*") 2>/dev/null

# Find useForm / useField usage
grep -rn "useForm\|useField\|useFormContext\|Controller\b" \
  --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20

# Find form elements — do they tie back to a form library?
grep -rn "<form\b\|onSubmit" \
  --include="*.tsx" --include="*.jsx" . 2>/dev/null | grep -v node_modules | head -20

# Red flag: many useStates that look like form fields
grep -rn "useState" --include="*.tsx" --include="*.jsx" . 2>/dev/null \
  | grep -v node_modules \
  | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -10

# Check for schema-based validation (Zod/Yup schemas for forms)
grep -rn "z\.object\|yup\.object\|v\.object" \
  --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20

# Check for manual validation logic (bad pattern)
grep -rn "if.*\.length.*===.*0\|if.*===.*''\|setError\b" \
  --include="*.tsx" . 2>/dev/null | grep -v node_modules | head -20
```

Check for:

- A form library (React Hook Form, TanStack Form, Formik) present and actually used?
- Schema validation (Zod, Yup, Valibot) wired to the form library — not manual `if` checks?
- **Anti-pattern: form-as-useState** — a component with 4+ `useState` calls that correspond to form fields, a manual `handleChange`, and a manual validation block is effectively a hand-rolled form library. This is a significant red flag — students should use a proper library.
- Error messages rendered for the user (not just logged to the console)?
- Controlled vs uncontrolled inputs: mixed usage without clear intent is a smell.

---

### 14. Frontend Structure

Are route-level components lean orchestrators, or do they contain business logic, fetching, and rendering all tangled together?

```bash
# Find route/page component files
find . \( -path "*/routes/*" -o -path "*/pages/*" -o -path "*/app/*" \) \
  -name "*.tsx" -not -path "*/node_modules/*" | sort

# Count lines in route/page files — large files are a god-component smell
find . \( -path "*/routes/*" -o -path "*/pages/*" \) \
  -name "*.tsx" -not -path "*/node_modules/*" \
  | xargs wc -l 2>/dev/null | sort -rn | head -15

# Detect TanStack Router loader usage (preferred fetch location)
grep -rn "loader\b\|useLoaderData\|beforeLoad\|Route\.createFileRoute\|createRootRoute" \
  --include="*.tsx" --include="*.ts" . 2>/dev/null | grep -v node_modules | head -20

# Detect data fetching directly inside component bodies (outside hooks)
grep -rn "useQuery\|useSWR\|fetch(\|axios\." \
  --include="*.tsx" . 2>/dev/null | grep -v node_modules | head -30

# Check for heavy business logic in route files
grep -rn "\.filter(\|\.map(\|\.reduce(\|\.sort(" \
  --include="*.tsx" . 2>/dev/null \
  | grep -v "node_modules\|components\|hooks" \
  | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -10

# Look for custom hooks that extract logic from components
find . -name "use*.ts" -o -name "use*.tsx" | grep -v node_modules | sort
```

Check for:

- **God components**: route-level files over ~300 lines that mix fetching, transformation, and rendering are a significant problem. Read the largest files found above and assess whether they should be decomposed.
- **Fetch placement** (when TanStack Router is used): critical data fetches should live in route `loader` functions or `beforeLoad`, not inside component bodies. This enables parallel loading and avoids loading waterfalls. Flag any `useQuery` inside a route component that could move to a loader.
- **Page as orchestrator**: pages/routes should delegate to purpose-built sub-components and custom hooks. A route file that contains inline filter logic, sort comparisons, or multi-step data transformation is carrying logic that belongs in a service or hook.
- **Logic/rendering separation**: are there custom `use*.ts` hooks extracting non-trivial logic out of components? A project with zero custom hooks and large page files almost certainly has mixed concerns.
- **Component cohesion**: sub-components should be responsible for a single visual/functional unit. A component called `Dashboard` that renders a table, a chart, a form, and a sidebar is too broad.

---

## Output: Write REVIEW.md

After completing all 14 categories, write the full report to `REVIEW.md` in the current working directory using the Write tool. Use this exact structure:

```markdown
# Project Review: [project name from package.json "name" field or directory name]

**Date:** [today's date in YYYY-MM-DD]

---

## Project Orientation

[3-6 sentences: project structure, detected tech stack, architectural observations. Descriptive only.]

---

## Review Summary

| Category              | Status         |
| --------------------- | -------------- |
| Component Library     | [emoji status] |
| Styling               | [emoji status] |
| Loading Data          | [emoji status] |
| Environment Variables | [emoji status] |
| REST API Design       | [emoji status] |
| Database              | [emoji status] |
| BE Design Patterns    | [emoji status] |
| Auth                  | [emoji status] |
| Testing               | [emoji status] |
| Logging & Monitoring  | [emoji status] |
| Error Handling        | [emoji status] |
| Security              | [emoji status] |
| Forms                 | [emoji status] |
| Frontend Structure    | [emoji status] |

Status legend: ✅ Good | ⚠️ Concerns | ❌ Issues | N/A

---

## Detailed Review

### 1. Component Library

**Status:** [emoji] [Good/Concerns/Issues/N/A — reason]

[Findings narrative with specific file:line evidence and code snippets]

**Recommendations:**

1. [Specific actionable item with file reference if applicable]

[Repeat structure for all 14 categories]
```

After writing the file, tell the user:

> "Review complete. REVIEW.md written to [absolute path]."

Do not output the full report content to the conversation — it belongs in the file.
