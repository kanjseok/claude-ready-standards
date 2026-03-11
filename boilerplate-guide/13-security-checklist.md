# 13. Security Checklist

> Critical security assessment checks spanning across all monorepo tiers before deploying applications.
> Includes secret management, API gateway rules, Docker isolation, and database safeguards.

---

## 1. Credentials & Secret Management

Leaked secrets are the most common and most damaging security incident. Every item below must be verified before the first deployment.

- [ ] **`.env` Exclusion**: Ensure `.env`, `.env.local`, and `.env.production` are listed in `.gitignore`.

```gitignore
# .gitignore
.env
.env.*
!.env.example
```

- [ ] **Private Key Isolation**: Confirm `keys/private.pem` (and all key material) is excluded from version control.

```bash
# Verify keys are gitignored
git ls-files keys/
# Should return empty — if not, remove and add to .gitignore
```

- [ ] **No Hardcoded Credentials**: Database URLs, API keys, and passwords must be loaded from environment variables, never hardcoded in source files.

```prisma
// prisma/schema.prisma — correct approach
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

- [ ] **Secret Rotation Schedule**: Establish a rotation cadence for all secrets:
  - JWT signing keys: every 90 days
  - OAuth client secrets: every 180 days
  - Cloud service credentials: per provider recommendation
  - Database passwords: every 90 days

- [ ] **Pre-commit Secret Scanning**: Install `git-secrets` or `trufflehog` to block accidental credential commits.

```bash
# Install git-secrets
git secrets --install
git secrets --register-aws  # Example: block AWS keys

# Or use trufflehog for broader scanning
trufflehog git file://. --only-verified
```

---

## 2. API & Endpoint Security

- [ ] **JWT Algorithm**: Use RS256 (asymmetric) instead of HS256 (symmetric). Asymmetric keys allow the public key to be shared for verification without exposing the signing key.

```typescript
// NestJS JWT Module configuration
JwtModule.register({
  privateKey: fs.readFileSync("keys/private.pem"),
  publicKey: fs.readFileSync("keys/public.pem"),
  signOptions: { algorithm: "RS256", expiresIn: "15m" },
})
```

- [ ] **Refresh Token Rotation**: Invalidate the previous refresh token every time a new access token is issued. Store refresh tokens server-side (database or Redis) and check validity on each use.

- [ ] **CORS Restrictions**: Never use wildcard `*` origins in production. Explicitly list allowed origins.

```typescript
// NestJS main.ts
app.enableCors({
  origin: [
    "https://admin.example.com",
    "https://web.example.com",
  ],
  credentials: true,
});
```

- [ ] **Rate Limiting**: Apply rate limits on authentication endpoints to prevent brute-force attempts.

```typescript
// NestJS ThrottlerModule
ThrottlerModule.forRoot([{
  ttl: 60000,   // 1 minute window
  limit: 10,    // max 10 requests per window
}])
```

- [ ] **Input Validation**: Validate all incoming payloads using `class-validator` (NestJS) or Pydantic (FastAPI). Strip unknown fields and reject malformed requests before they reach business logic.

```typescript
// NestJS DTO with validation
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  password: string;
}
```

---

## 3. Web Client Security (Next.js)

- [ ] **HttpOnly Cookies**: Store authentication tokens in `httpOnly`, `secure`, `sameSite=strict` cookies — never in `localStorage` or `sessionStorage`.

```typescript
// API route setting auth cookie
response.cookies.set("token", accessToken, {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "strict",
  maxAge: 60 * 15, // 15 minutes
  path: "/",
});
```

- [ ] **XSS Prevention**: Avoid `dangerouslySetInnerHTML`. If rendering user-generated HTML is unavoidable, sanitize with a library like `DOMPurify`.

```typescript
import DOMPurify from "dompurify";

// Only if absolutely necessary
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

- [ ] **Route Protection**: Implement middleware to redirect unauthenticated users away from protected routes.

```typescript
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("token");
  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/admin/:path*"],
};
```

- [ ] **Security Headers**: Configure Content-Security-Policy, X-Frame-Options, and other headers.

```typescript
// next.config.ts
const securityHeaders = [
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
];
```

---

## 4. Database & Infrastructure

- [ ] **Principle of Least Privilege**: Create dedicated database users with only the permissions they need. Never use the `root` or `superuser` account for application connections.

```sql
-- PostgreSQL: create a restricted application user
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

- [ ] **Dependency Audits**: Run vulnerability scans regularly and before every release.

```bash
# Node.js dependencies
pnpm audit

# Python dependencies
pip-audit

# Check for known CVEs
npx better-npm-audit audit
```

- [ ] **Environment Isolation**: Staging and production must use completely separate databases, secrets, and infrastructure. Never share credentials across environments.

- [ ] **Docker Non-Root Execution**: All containers must run as non-root users with read-only filesystems where possible.

```dockerfile
# Dockerfile — NestJS example
FROM node:24-alpine AS runner
RUN addgroup --system app && adduser --system --ingroup app app
USER app
COPY --chown=app:app --from=builder /app/dist ./dist
CMD ["node", "dist/main.js"]
```

- [ ] **Network Segmentation**: Database and Redis containers should not be exposed to the public network. Use Docker internal networks.

```yaml
# docker-compose.yml
services:
  postgres:
    networks:
      - internal
    # No 'ports' mapping — only accessible within Docker network
  backend:
    networks:
      - internal
      - public

networks:
  internal:
    internal: true
  public:
```

---

## 5. Auditing & Logging

- [ ] **Sensitive Data Masking**: Strip passwords, tokens, and PII from all log output before it reaches log storage.

```typescript
// Pino logger with redaction
const logger = pino({
  redact: {
    paths: ["req.headers.authorization", "req.body.password", "*.email"],
    censor: "[REDACTED]",
  },
});
```

- [ ] **Authentication Event Tracking**: Log all failed authentication attempts (401/403 responses) with IP address, timestamp, and target endpoint for anomaly detection.

- [ ] **Swagger Restriction**: Disable the Swagger UI in production environments.

```typescript
// NestJS main.ts
if (process.env.NODE_ENV !== "production") {
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup("api-docs", app, document);
}
```

- [ ] **Audit Trail**: Maintain immutable logs for critical operations (user creation, permission changes, data deletion) with actor identity and timestamp.

---

## Verification

Run through the following checks to confirm the security posture of the project:

```bash
# 1. Confirm no secrets are tracked in git
git ls-files | grep -E "\.env$|\.pem$|\.key$"
# Expected: empty output

# 2. Verify .gitignore covers sensitive files
grep -E "\.env|keys/|\.pem" .gitignore

# 3. Run dependency vulnerability scan
pnpm audit --audit-level=high

# 4. Scan for hardcoded secrets in source
npx trufflehog git file://. --only-verified 2>/dev/null || echo "Install trufflehog for secret scanning"

# 5. Verify Docker containers run as non-root
docker compose exec backend whoami
# Expected: non-root user (e.g., "app", "nestjs", "nextjs")

# 6. Confirm Swagger is disabled in production
curl -s -o /dev/null -w "%{http_code}" https://your-production-url/api-docs
# Expected: 404 or 403

# 7. Verify CORS configuration rejects unauthorized origins
curl -s -o /dev/null -w "%{http_code}" \
  -H "Origin: https://malicious-site.com" \
  https://your-api-url/health
# Expected: no Access-Control-Allow-Origin header in response
```

All checks passing + dependency audit clean + no secrets in git history = Security baseline confirmed.
