# Six Failure Modes of AI-Assisted Development

This reference is used during Audit Mode. For each failure mode, the skill runs the listed detection checks and adds confirmed findings to the remediation register.

These are not hypothetical. They are the patterns that cause founders to spend 18 days building in isolation only to discover the codebase cannot be continued — and to pay $5,000 to $50,000 for a rescue engagement.

---

## Failure Mode 1: Disconnected Schema

**What it looks like:** The database schema exists and tables were created, but the application never reads from or writes to them. The schema was generated in one session and the application was built in another, with no shared architectural context.

**Detection:**
```bash
# Find all table names in migration files
grep -rh "CREATE TABLE" ./src/database/migrations/ | sed 's/.*CREATE TABLE \([a-z_]*\).*/\1/'

# Find all table names referenced in repository files
grep -rh "FROM\|INTO\|UPDATE\|JOIN" ./src/ --include="*.ts" | grep -oP '(?<=FROM |INTO |UPDATE |JOIN )\w+' | sort -u

# Tables in migrations not found in application code = disconnected schema
# Manual review required: diff the two lists
```

**Remediation path:**
1. Audit Mode: list every disconnected table in the finding register with severity HIGH.
2. Architecture Phase: add each disconnected table to the architecture document with its data flow.
3. Implementation Phase: wire each table to its appropriate repository and service layer.

---

## Failure Mode 2: Unwired Frontend Pages

**What it looks like:** UI components render correctly with hardcoded or mocked data. The components exist, they look finished, but they call no API routes. Or they call routes that do not exist in the backend.

**Detection:**
```bash
# Find all API calls in frontend code
grep -rn "fetch\|axios\|useQuery\|useMutation\|api\." ./src --include="*.tsx" --include="*.ts" | grep -v "node_modules"

# Find all defined API routes in backend
grep -rn "@Get\|@Post\|@Put\|@Delete\|@Patch\|router\." ./src --include="*.ts" | grep -v "node_modules"

# Find hardcoded mock data that should be dynamic
grep -rn "const mock\|const fake\|TODO.*api\|FIXME.*backend\|hardcoded" ./src --include="*.ts" --include="*.tsx"
```

**Remediation path:**
1. Audit Mode: for each UI component with mocked data, record the missing endpoint in the finding register with severity HIGH.
2. Architecture Phase: define the API contract for each missing route.
3. Implementation Phase: build the route as an atomic task, then update the frontend as a separate atomic task.

---

## Failure Mode 3: Incomplete Backend Wiring

**What it looks like:** Service classes exist but are not injected anywhere. Controllers exist but call services directly with no dependency injection. Modules are defined but not imported into the application module. The backend compiles and runs — but entire feature domains are unreachable.

**Detection:**
```bash
# Find service classes that are never injected (NestJS example)
grep -rn "export class.*Service" ./src --include="*.ts" | awk -F: '{print $1}' | while read file; do
  classname=$(grep "export class.*Service" "$file" | head -1 | sed 's/.*export class \([A-Za-z]*Service\).*/\1/')
  grep -rn "$classname" ./src --include="*.module.ts" > /dev/null || echo "UNWIRED SERVICE: $classname in $file"
done

# Find modules not imported in AppModule
grep -n "imports:" ./src/app.module.ts

# Find controllers not registered in any module
grep -rn "export class.*Controller" ./src --include="*.ts"
```

**Remediation path:**
1. Audit Mode: list every unregistered service and controller in the finding register.
2. Architecture Phase: confirm the module dependency map.
3. Implementation Phase: wire each service via atomic tasks, each with a VERIFY command that confirms registration.

---

## Failure Mode 4: Missing Row Level Security

**What it looks like:** Supabase or Postgres tables contain user data with no RLS policies. The application uses an anonymous public key in client-side code. Any user, authenticated or not, can query any row.

**Documented consequence:** 88% of AI-generated applications using Supabase had RLS disabled or misconfigured as of early 2026. One researcher found 170 critical security failures in 1,645 publicly listed apps.

**Detection:**
```bash
# Check RLS status for all tables (Supabase / Postgres)
psql $DATABASE_URL -c "
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;"

# Any table with rowsecurity = false = CRITICAL finding

# Check for policies on RLS-enabled tables
psql $DATABASE_URL -c "
SELECT schemaname, tablename, policyname, permissive, roles, cmd
FROM pg_policies
WHERE schemaname = 'public'
ORDER BY tablename, policyname;"

# Check for anon key usage in client-side code (should only be in server-side)
grep -rn "SUPABASE_ANON_KEY\|supabaseAnonKey\|anon" ./src --include="*.tsx" --include="*.ts"
```

**Remediation path:**
1. Audit Mode: record every table with `rowsecurity = false` as severity CRITICAL. Do not remediate inline.
2. Security Phase: write RLS policies for each table as atomic tasks.
3. VERIFY: re-run the detection queries to confirm policies are active.

---

## Failure Mode 5: N+1 Query Explosions

**What it looks like:** A list endpoint fetches N parent records, then issues N individual queries for child records inside a loop. At prototype scale (10 records), imperceptible. At production scale (10,000 records), it exhausts the database connection pool in seconds.

**Documented consequence:** Unoptimized AI-generated queries have inflated cloud infrastructure bills by up to 400% at production scale. A single AI-generated admin dashboard caused $12,000 in database costs in one month.

**Detection:**
```bash
# Find query-in-loop patterns (ORM-agnostic)
grep -rn "for.*await\|\.map.*await\|forEach.*await" ./src --include="*.ts" | grep -i "find\|query\|select\|fetch\|get"

# Find missing eager loading in TypeORM / Prisma
grep -rn "findMany\|findAll\|find({" ./src --include="*.ts" | grep -v "include:\|relations:\|select:"

# Find missing pagination on list endpoints
grep -rn "findAll\|findMany" ./src --include="*.ts" | grep -v "take:\|limit:\|skip:\|offset:"
```

**Remediation path:**
1. Audit Mode: record every query-in-loop as severity HIGH. Estimate the cost impact at expected production scale.
2. Architecture Phase: define eager loading strategy, batch query patterns, and pagination for every list operation.
3. Implementation Phase: rewrite each query as an atomic task with a VERIFY that confirms no loop-inside-query pattern.

---

## Failure Mode 6: Exposed Secrets and Broken Authentication

**What it looks like:** API keys committed to source control. Credentials hardcoded in client-side bundles. JWT signing secrets in environment files that are not gitignored. Authentication checks implemented in the UI layer only, bypassed by anyone with browser dev tools.

**Detection:**
```bash
# Scan for hardcoded credentials
grep -rn "sk_live\|sk_test\|AKIA\|ghp_\|xoxb-\|password.*=.*['\"]" ./src --include="*.ts" --include="*.tsx" --include="*.js" --include="*.env"

# Check .gitignore covers all env files
cat .gitignore | grep "\.env"
git ls-files | grep "\.env"  # any .env files tracked = CRITICAL

# Find client-side auth checks (these are bypassable)
grep -rn "isAdmin\|hasRole\|isAuthenticated" ./src --include="*.tsx" | grep -v "server\|api\|guard\|middleware"

# Find auth checks missing from API routes
grep -rn "@Get\|@Post\|@Put\|@Delete" ./src --include="*.controller.ts" | while read line; do
  file=$(echo $line | cut -d: -f1)
  echo "$file" # check manually for @UseGuards decorator
done
```

**Remediation path:**
1. Audit Mode: any committed secret is severity CRITICAL. Stop all other work. Rotate the secret immediately. Rewrite git history.
2. Architecture Phase: define authentication guards for every route. No route is unguarded by default.
3. Implementation Phase: add guards as atomic tasks with VERIFY commands that confirm `@UseGuards` is present on every controller.

---

## Remediation Register Template

When running in Audit Mode, populate this register for every finding before proposing any fixes.

```markdown
# Remediation Register: [Project Name]

Audit date: [date]
Auditor: [name]
Codebase: [repo]

## Findings

| ID | Failure Mode | Location | Severity | Description | Estimated Remediation |
|----|-------------|----------|----------|-------------|----------------------|
| F1 | Disconnected Schema | src/database/migrations/... | HIGH | users table never queried in application | 4–8 hours |
| F2 | Missing RLS | public.orders | CRITICAL | Orders table has no row-level security — all rows publicly readable | 1–2 hours |
| F3 | N+1 Query | src/modules/products/products.service.ts:47 | HIGH | findAll loops N image queries per product | 2–4 hours |

## Remediation Sequence

Severity order: CRITICAL → HIGH → MEDIUM → LOW.
Do not begin HIGH remediation until all CRITICAL findings are resolved and re-verified.

## Re-Verification Checklist

- [ ] All CRITICAL findings resolved and VERIFY commands passing
- [ ] All HIGH findings resolved and VERIFY commands passing
- [ ] Architecture document updated to reflect actual state
- [ ] CLAUDE.md updated with security and architectural constraints
```
