# 06. Authentication + Security

> JWT RS256 + Google OAuth + Guard Chain + AES-256-GCM encryption guide.
> Covers the overall implementation pattern of the authentication module and security utilities.

---

## 1. Generating RSA Keypair

Generate a 4096-bit RSA keypair for JWT RS256 signing:

```typescript
// tooling/scripts/generate-keys.ts
import { generateKeyPairSync, createHash } from "node:crypto";
import { writeFileSync, mkdirSync } from "node:fs";
import { join } from "node:path";

function generateKeys() {
  const { privateKey, publicKey } = generateKeyPairSync("rsa", {
    modulusLength: 4096,
    publicKeyEncoding: { type: "spki", format: "pem" },
    privateKeyEncoding: { type: "pkcs8", format: "pem" },
  });

  const keysDir = join(process.cwd(), "..", "..", "keys");
  mkdirSync(keysDir, { recursive: true });

  writeFileSync(join(keysDir, "private.pem"), privateKey, { mode: 0o600 });
  writeFileSync(join(keysDir, "public.pem"), publicKey);

  const fingerprint = createHash("sha256")
    .update(publicKey)
    .digest("hex")
    .substring(0, 16);

  console.log(`Keys generated — Fingerprint: ${fingerprint}`);
}

generateKeys();
```

```bash
# Execute
cd tooling/scripts && npx tsx generate-keys.ts

# Result
# keys/private.pem  (mode 0600 — owner read only)
# keys/public.pem
```

**Important**: The `keys/` directory must be included in `.gitignore`.

---

## 2. Environment Variables

```bash
# .env
JWT_PRIVATE_KEY=./keys/private.pem
JWT_PUBLIC_KEY=./keys/public.pem
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
ALLOWED_DOMAIN=example.com    # comma-separated for multiple domains
ENCRYPTION_KEY=change-me      # openssl rand -hex 32
```

---

## 3. AuthModule

```typescript
// src/modules/auth/auth.module.ts
import { Module } from "@nestjs/common";
import { PassportModule } from "@nestjs/passport";
import { JwtModule } from "@nestjs/jwt";
import { ConfigModule, ConfigService } from "@nestjs/config";
import * as fs from "fs";
import * as path from "path";
import { AuthController } from "./auth.controller";
import { AuthService } from "./auth.service";
import { JwtStrategy } from "./strategies/jwt.strategy";
import { UsersModule } from "../users/users.module";

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => {
        const privateKeyPath = configService.get<string>("auth.jwtPrivateKey");
        const publicKeyPath = configService.get<string>("auth.jwtPublicKey");

        let privateKey: string | undefined;
        let publicKey: string | undefined;

        if (privateKeyPath) {
          try {
            privateKey = fs.readFileSync(path.resolve(privateKeyPath), "utf8");
          } catch {
            // Key file not found — will fail at runtime if auth is used
          }
        }

        if (publicKeyPath) {
          try {
            publicKey = fs.readFileSync(path.resolve(publicKeyPath), "utf8");
          } catch {
            // Key file not found
          }
        }

        return {
          privateKey,
          publicKey,
          signOptions: { algorithm: "RS256" },
        };
      },
      inject: [ConfigService],
    }),
    UsersModule,
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

---

## 4. JWT Strategy (Passport)

```typescript
// src/modules/auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { ConfigService } from "@nestjs/config";
import { UsersService } from "../../users/users.service";
import type { TokenPayload } from "@scope/shared";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private readonly configService: ConfigService,
    private readonly usersService: UsersService,
  ) {
    const publicKey = configService.get<string>("auth.jwtPublicKey");
    if (!publicKey) {
      throw new Error("JWT_PUBLIC_KEY is required");
    }

    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: publicKey, // Content of the PEM file (not the file path)
      algorithms: ["RS256"],
    });
  }

  async validate(payload: TokenPayload) {
    const user = await this.usersService.findById(payload.sub);
    if (!user || !user.isActive) {
      throw new UnauthorizedException("User not found or inactive");
    }
    return user; // assigned to request.user
  }
}
```

**Note**: The **content** of the PEM file should be passed to `secretOrKey`. It receives the values read by `fs.readFileSync()` in the JwtModule.

---

## 5. Guard Chain

Guard Application Order: `JwtAuthGuard` → `RolesGuard` → `DomainGuard`

### JwtAuthGuard

```typescript
// src/modules/auth/guards/jwt-auth.guard.ts
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

### RolesGuard

```typescript
// src/modules/auth/guards/roles.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { ROLE_HIERARCHY } from "@scope/shared";
import type { UserRole } from "@scope/shared";
import { ROLES_KEY } from "../../../common/decorators/roles.decorator";
import type { User } from "../../../generated/prisma/client";

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles || requiredRoles.length === 0) {
      return true; // No role restrictions
    }

    const request = context.switchToHttp().getRequest<{ user?: User }>();
    const user = request.user;

    if (!user) {
      throw new ForbiddenException("Authentication required");
    }

    const userLevel = ROLE_HIERARCHY[user.role as UserRole];
    const hasRole = requiredRoles.some(
      (role) => userLevel >= ROLE_HIERARCHY[role],
    );

    if (!hasRole) {
      throw new ForbiddenException("Insufficient permissions");
    }

    return true;
  }
}
```

**Pattern**: Hierarchical permissions utilizing `ROLE_HIERARCHY` — `admin(3) > editor(2) > viewer(1)`.

### DomainGuard

```typescript
// src/modules/auth/guards/domain.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import type { User } from "../../../generated/prisma/client";

@Injectable()
export class DomainGuard implements CanActivate {
  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<{ user?: User }>();
    const user = request.user;

    if (!user) {
      throw new ForbiddenException("Authentication required");
    }

    const allowedDomains = this.configService.get<string[]>(
      "auth.allowedDomains",
    ) ?? ["example.com"];
    const emailDomain = user.email.split("@")[1]?.toLowerCase();

    if (!allowedDomains.includes(emailDomain)) {
      throw new ForbiddenException(
        `Access restricted to @${allowedDomains.join(", @")} accounts`,
      );
    }

    return true;
  }
}
```

### Utilizing Guards in Controllers

```typescript
@Controller("admin")
@UseGuards(JwtAuthGuard, RolesGuard, DomainGuard)
export class AdminController {
  @Get("users")
  @Roles("admin") // Accessible only by admin
  findAll() {
    /* ... */
  }

  @Get("stats")
  @Roles("editor") // Accessible by editor or higher (includes admin)
  getStats() {
    /* ... */
  }
}
```

### Consumer-Facing Applications

For consumer-facing web frontends (e.g., PWA apps per [17-consumer-web-pwa](./17-consumer-web-pwa.md)), use **`JwtAuthGuard` only** — without `RolesGuard` or `DomainGuard`:

```typescript
@Controller("profile")
@UseGuards(JwtAuthGuard) // No RolesGuard, no DomainGuard
export class ProfileController {
  @Get()
  getProfile(@CurrentUser("id") userId: string) {
    /* ... */
  }

  @Post()
  createProfile(@CurrentUser("id") userId: string, @Body() dto: CreateProfileDto) {
    /* ... */
  }
}
```

| Aspect             | Admin Service                        | Consumer Service           |
| ------------------- | ------------------------------------ | -------------------------- |
| Guard chain         | `JwtAuth` + `Roles` + `Domain`      | `JwtAuth` only             |
| Domain restriction  | `ALLOWED_DOMAIN` env                 | None (open to all)         |
| Role checking       | `@Roles('admin')` decorators         | Not used                   |
| `ALLOWED_DOMAIN`    | Required                             | Omitted from `.env`        |

---

## 6. AuthService (Core Logic)

```typescript
// Token Generation Pattern (Core portion only)
private async generateTokens(
  user: User,
  ipAddress?: string,
  userAgent?: string,
): Promise<AuthTokens> {
  const payload: Omit<TokenPayload, 'iat' | 'exp'> = {
    sub: user.id,
    email: user.email,
    name: user.name,
    role: user.role as UserRole,
    iss: 'project-name',
    aud: 'project-client',
  };

  const accessExpiry = this.configService.get<string>('auth.jwtAccessExpiry') ?? '15m';
  const accessToken = this.jwtService.sign(payload, { expiresIn: accessExpiry });

  // Refresh Token: Random bytes (NOT JWT!)
  const refreshToken = crypto.randomBytes(64).toString('hex');
  const refreshTokenHash = this.hashToken(refreshToken);

  // Save to Session DB
  await this.usersService.createSession({
    userId: user.id,
    refreshTokenHash,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    ipAddress,
    userAgent,
  });

  return { accessToken, refreshToken, expiresIn: 900 };
}

private hashToken(token: string): string {
  return crypto.createHash('sha256').update(token).digest('hex');
}
```

### Refresh Token Rotation

```
1. Client → POST /auth/refresh (cookie: refresh_token)
2. Server: Retrieves the session utilizing the token hash
3. Server: Revokes the existing session (Sets revokedAt)
4. Server: Issues a new session + new token pair
5. Server → Client (new access + new refresh cookie)
```

### Session Limitation

```typescript
// Maximum 5 sessions per user — if exceeded, the oldest session is removed
await this.usersService.enforceMaxSessions(user.id, MAX_SESSIONS_PER_USER);
```

---

## 7. Auth DTO (Swagger Schema)

### GoogleLoginDto

```typescript
// src/modules/auth/dto/google-login.dto.ts
import { ApiProperty } from "@nestjs/swagger";
import { IsString } from "class-validator";

export class GoogleLoginDto {
  @ApiProperty({
    description: "ID Token issued by Google OAuth",
    example: "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  })
  @IsString()
  idToken!: string;
}
```

### Auth Response DTO

Mirrors the `AuthUser` and `AuthTokens` types from `@scope/shared` such that Swagger recognizes them:

```typescript
// src/modules/auth/swagger/auth-response.schema.ts
import { ApiProperty } from "@nestjs/swagger";

/** Public user info — Mirrors AuthUser */
export class PublicUserSchema {
  @ApiProperty({ example: "550e8400-e29b-41d4-a716-446655440000" })
  id!: string;

  @ApiProperty({ example: "user@example.com" })
  email!: string;

  @ApiProperty({ example: "John Doe" })
  name!: string;

  @ApiProperty({
    example: "https://lh3.googleusercontent.com/...",
    nullable: true,
  })
  profileImage!: string | null;

  @ApiProperty({ example: "example.com" })
  domain!: string;

  @ApiProperty({ example: "viewer", enum: ["admin", "editor", "viewer"] })
  role!: string;
}

/** POST /auth/google response data */
export class LoginResponseData {
  @ApiProperty({ type: PublicUserSchema })
  user!: PublicUserSchema;

  @ApiProperty({ example: "eyJhbGciOiJSUzI1NiIs..." })
  accessToken!: string;
}

/** POST /auth/google full response (Wrapped by ResponseTransformInterceptor) */
export class LoginResponse {
  @ApiProperty({ example: true })
  success!: boolean;

  @ApiProperty({ type: LoginResponseData })
  data!: LoginResponseData;

  @ApiProperty({ example: "2026-02-28T12:00:00.000Z" })
  timestamp!: string;
}

/** GET /auth/me full response */
export class MeResponse {
  @ApiProperty({ example: true })
  success!: boolean;

  @ApiProperty({ type: PublicUserSchema })
  data!: PublicUserSchema;

  @ApiProperty({ example: "2026-02-28T12:00:00.000Z" })
  timestamp!: string;
}
```

---

## 8. AuthController

```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Req,
  Res,
  UseGuards,
  HttpCode,
  HttpStatus,
} from "@nestjs/common";
import {
  ApiTags,
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
} from "@nestjs/swagger";
import { Response } from "express";
import { JwtAuthGuard } from "./guards/jwt-auth.guard";
import { CurrentUser } from "../../common/decorators/current-user.decorator";
import { ApiErrorSchema } from "../../common/swagger/schemas";
import { GoogleLoginDto } from "./dto/google-login.dto";
import { LoginResponse, MeResponse } from "./swagger/auth-response.schema";
import type { User } from "../../generated/prisma/client";

// src/modules/auth/auth.controller.ts
@ApiTags("Auth")
@Controller("auth")
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post("google")
  @HttpCode(HttpStatus.OK)
  @ApiOperation({
    summary: "Google Login",
    description:
      "Log in with Google ID Token and issue Access Token + Refresh Token cookies.",
  })
  @ApiResponse({
    status: HttpStatus.OK,
    description: "Login successful",
    type: LoginResponse,
  })
  @ApiResponse({
    status: HttpStatus.BAD_REQUEST,
    description: "Invalid ID Token",
    type: ApiErrorSchema,
  })
  @ApiResponse({
    status: HttpStatus.FORBIDDEN,
    description: "Unauthorized domain",
    type: ApiErrorSchema,
  })
  async googleLogin(
    @Body() dto: GoogleLoginDto,
    @Req() req,
    @Res({ passthrough: true }) res: Response,
  ) {
    const { user, tokens } = await this.authService.loginWithGoogle(
      dto.idToken,
      req.ip,
      req.headers["user-agent"],
    );
    this.setRefreshTokenCookie(res, tokens.refreshToken);
    return { user: this.toPublicUser(user), accessToken: tokens.accessToken };
  }

  @Post("refresh")
  @HttpCode(HttpStatus.OK)
  @ApiOperation({
    summary: "Token Refresh",
    description:
      "Issue a new Access Token + Refresh Token using the Refresh Token cookie.",
  })
  @ApiResponse({
    status: HttpStatus.OK,
    description: "Refresh successful",
    type: LoginResponse,
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Invalid Refresh Token",
    type: ApiErrorSchema,
  })
  async refresh(@Req() req, @Res({ passthrough: true }) res: Response) {
    /* ... */
  }

  @Post("logout")
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiBearerAuth("access-token")
  @UseGuards(JwtAuthGuard)
  @ApiOperation({
    summary: "Logout",
    description:
      "Revoke the current session's Refresh Token and clear the cookie.",
  })
  @ApiResponse({
    status: HttpStatus.NO_CONTENT,
    description: "Logout successful",
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Authentication required",
    type: ApiErrorSchema,
  })
  async logout(@Req() req, @Res({ passthrough: true }) res: Response) {
    /* ... */
  }

  @Get("me")
  @ApiBearerAuth("access-token")
  @UseGuards(JwtAuthGuard)
  @ApiOperation({ summary: "Get current user info" })
  @ApiResponse({
    status: HttpStatus.OK,
    description: "Retrieve successful",
    type: MeResponse,
  })
  @ApiResponse({
    status: HttpStatus.UNAUTHORIZED,
    description: "Authentication required",
    type: ApiErrorSchema,
  })
  getMe(@CurrentUser() user: User) {
    return this.toPublicUser(user);
  }

  private setRefreshTokenCookie(res: Response, token: string) {
    res.cookie("refresh_token", token, {
      httpOnly: true, // Prevent XSS
      secure: process.env.NODE_ENV === "production", // HTTPS only
      sameSite: "lax", // Prevent CSRF
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
      path: "/api/auth", // Auth paths only
    });
  }
}
```

---

## 9. AES-256-GCM Encryption Util

Utilized for storing sensitive data (such as external API tokens):

```typescript
// src/common/utils/encryption.util.ts
import * as crypto from "crypto";

const ALGORITHM = "aes-256-gcm";
const IV_LENGTH = 16;
const AUTH_TAG_LENGTH = 16;

function getKey(encryptionKey: string): Buffer {
  return crypto.createHash("sha256").update(encryptionKey).digest();
}

export function encrypt(plaintext: string, encryptionKey: string): string {
  const key = getKey(encryptionKey);
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv(ALGORITHM, key, iv);

  const encrypted = Buffer.concat([
    cipher.update(plaintext, "utf8"),
    cipher.final(),
  ]);
  const authTag = cipher.getAuthTag();

  // IV (16) + AuthTag (16) + Ciphertext → Base64
  const combined = Buffer.concat([iv, authTag, encrypted]);
  return combined.toString("base64");
}

export function decrypt(encryptedData: string, encryptionKey: string): string {
  const key = getKey(encryptionKey);
  const combined = Buffer.from(encryptedData, "base64");

  if (combined.length < IV_LENGTH + AUTH_TAG_LENGTH) {
    throw new Error("Invalid encrypted data: too short");
  }

  const iv = combined.subarray(0, IV_LENGTH);
  const authTag = combined.subarray(IV_LENGTH, IV_LENGTH + AUTH_TAG_LENGTH);
  const ciphertext = combined.subarray(IV_LENGTH + AUTH_TAG_LENGTH);

  const decipher = crypto.createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(authTag);

  const decrypted = Buffer.concat([
    decipher.update(ciphertext),
    decipher.final(),
  ]);
  return decrypted.toString("utf8");
}
```

**Usage Example**:

```typescript
import { encrypt, decrypt } from "../common/utils/encryption.util";

const encryptionKey = this.configService.get<string>("encryptionKey");
const encrypted = encrypt(apiToken, encryptionKey);
const decrypted = decrypt(encrypted, encryptionKey);
```

---

## 10. Authentication Flow Summary

```
Google ID Token → POST /auth/google
  → Verify Google token (google-auth-library)
  → Check domain restrictions (ALLOWED_DOMAIN)
  → Fetch/Create user (Prisma)
  → Limit active sessions (MAX_SESSIONS_PER_USER)
  → Issue Access Token(RS256, 15m) + Refresh Token(random, 7d)
  → Set Refresh Token as an httpOnly cookie
```

---

## Verification

```bash
# Verify RSA key generation
ls -la keys/private.pem keys/public.pem

# Test Authentication Endpoints (Swagger)
# Test POST /api/auth/google at http://localhost:15320/docs

# Verify Guard Behavior
curl -H "Authorization: Bearer <token>" http://localhost:15320/api/auth/me
```

### Checking the Swagger Document (http://localhost:15320/docs)

- [ ] Ensure the 4 endpoints (google, refresh, logout, me) show up under the `Auth` tag
- [ ] `POST /auth/google` — Confirm `GoogleLoginDto` schema is visible within the request body
- [ ] `POST /auth/google` — Confirm schemas for 200, 400, and 403 responses
- [ ] `POST /auth/refresh` — Confirm definitions for 200 and 401 responses
- [ ] `POST /auth/logout` — Verify lock icon (requires auth) is displayed, 204 response
- [ ] `GET /auth/me` — Verify lock icon is displayed, 200 response contains the `PublicUserSchema`
- [ ] Ensure public endpoints (`google`, `refresh`) do not contain a lock icon
