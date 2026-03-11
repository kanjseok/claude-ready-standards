# 15. Backend Skeleton (Kotlin + Spring Boot)

> Kotlin + Spring Boot backend skeleton guide.
> Covers dependencies, directory structure, entry point, common utilities, and Kotlin/Spring Boot-specific patterns.

---

## 1. Dependencies

### `build.gradle.kts`

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "2.0.20"
    kotlin("plugin.spring") version "2.0.20"
    kotlin("plugin.jpa") version "2.0.20"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_21

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot Core
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    // Database
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    runtimeOnly("org.postgresql:postgresql")

    // Auth & JWT
    implementation("io.jsonwebtoken:jjwt-api:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")

    // OpenAPI (Swagger)
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## 2. Directory Structure

```
apps/backend-kt/src/main/kotlin/com/example/backend/
├── BackendApplication.kt      # Application entry point
├── common/                    # Common utilities, decorators, exceptions
│   ├── config/                # Security, Swagger, CORS, Redis Config
│   ├── dto/                   # Common DTOs (ApiResponse, Pagination)
│   ├── exception/             # Custom exceptions, @RestControllerAdvice
│   ├── resolver/              # ArgumentResolvers (CurrentUser)
│   └── utils/                 # Utility logic (Encryption)
├── core/                      # Core domain abstractions
├── modules/                   # Feature modules
│   ├── auth/
│   │   ├── AuthController.kt
│   │   ├── AuthService.kt
│   │   ├── JwtProvider.kt
│   │   └── security/          # Filters, UserDetails
│   └── users/
│       ├── UsersController.kt
│       ├── UsersService.kt
│       ├── UsersRepository.kt # JPA/MongoDB Repositories
│       ├── entity/            # JPA/Document Entities
│       └── dto/               # Request/Response DTOs
```

**Key Patterns**:

- **Package-by-Feature**: Features are grouped within `modules/{feature-name}` matching the NestJS module structure.
- **DTO Isolation**: Request and Response bodies are encapsulated within the inner `dto` package respective to the component's module.

---

## 3. Entry Point & Configuration

### Application Entry

```kotlin
// src/main/kotlin/com/example/backend/BackendApplication.kt
package com.example.backend

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class BackendApplication

fun main(args: Array<String>) {
    runApplication<BackendApplication>(*args)
}
```

### application.yml

Configuration files manage active profiles overriding application states. Replaces `ConfigModule` and `.env` parsing algorithms.

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: weav-standards-api
  profiles:
    active: ${ENVIRONMENT:development}

  datasource:
    url: ${DATABASE_URL}

  data:
    mongodb:
      uri: ${MONGODB_URI}
    redis:
      url: ${REDIS_URL}

app:
  jwt:
    private-key: ${JWT_PRIVATE_KEY}
    public-key: ${JWT_PUBLIC_KEY}
    access-expiry: 15m
  cors:
    origins: "http://localhost:3000,http://localhost:15310"
```

A `@ConfigurationProperties` class performs validation.

```kotlin
// src/main/kotlin/com/example/backend/common/config/AppProperties.kt
package com.example.backend.common.config

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Configuration

@Configuration
@ConfigurationProperties(prefix = "app")
class AppProperties {
    val jwt = Jwt()
    val cors = Cors()

    class Jwt {
        lateinit var privateKey: String
        lateinit var publicKey: String
        lateinit var accessExpiry: String
    }

    class Cors {
        lateinit var origins: String
        fun getOriginList(): List<String> = origins.split(",").map { it.trim() }
    }
}
```

---

## 4. Common Utilities

### 4.1 ApiError Exception & Advice

Maps precisely towards standard JSON `error` parameters, substituting NestJS's `HttpExceptionFilter`.

```kotlin
// src/main/kotlin/com/example/backend/common/exception/ApiException.kt
package com.example.backend.common.exception

import java.time.ZoneOffset
import java.time.ZonedDateTime
import java.time.format.DateTimeFormatter

open class ApiException(
    val statusCode: Int,
    override val message: String,
    val code: String = "ERR_$statusCode",
    val details: Any? = null,
) : RuntimeException(message) {
    val timestamp: String = ZonedDateTime.now(ZoneOffset.UTC).format(DateTimeFormatter.ISO_INSTANT)
}
```

```kotlin
// src/main/kotlin/com/example/backend/common/exception/GlobalExceptionHandler.kt
package com.example.backend.common.exception

import com.example.backend.common.dto.ApiError
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ApiException::class)
    fun handleApiException(ex: ApiException): ResponseEntity<Map<String, Any>> {
        val payload = mapOf(
            "success" to false,
            "error" to ApiError(
                code = ex.code,
                message = ex.message,
                details = ex.details
            ),
            "timestamp" to ex.timestamp
        )
        return ResponseEntity.status(ex.statusCode).body(payload)
    }

    // Include bindings targeting Validation exceptions mapping...
}
```

### 4.2 Pagination Request/Response

Substitutes the NestJS equivalent abstractions. Kotlin `data class` enables immutable structures.

```kotlin
// src/main/kotlin/com/example/backend/common/dto/Pagination.kt
package com.example.backend.common.dto

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import java.time.ZoneOffset
import java.time.ZonedDateTime
import java.time.format.DateTimeFormatter

data class PaginationRequest(
    @field:Min(1)
    val page: Int = 1,

    @field:Min(1)
    @field:Max(100)
    val limit: Int = 20
)

data class PaginationMeta(
    val totalItems: Long,
    val itemCount: Int,
    val itemsPerPage: Int,
    val totalPages: Int,
    val currentPage: Int
)

data class PaginatedResult<T>(
    val success: Boolean = true,
    val data: List<T>,
    val meta: PaginationMeta,
    val timestamp: String = ZonedDateTime.now(ZoneOffset.UTC).format(DateTimeFormatter.ISO_INSTANT)
)

data class ApiResponse<T>(
    val success: Boolean = true,
    val data: T,
    val timestamp: String = ZonedDateTime.now(ZoneOffset.UTC).format(DateTimeFormatter.ISO_INSTANT)
)
```

---

## 5. Security & Authentication

### 5.1 JwtAuthenticationFilter

Substitute the `PassportStrategy` operations natively mapping parameters integrating into Spring Security chains.

```kotlin
// src/main/kotlin/com/example/backend/modules/auth/security/JwtAuthenticationFilter.kt
package com.example.backend.modules.auth.security

import com.example.backend.modules.auth.JwtProvider
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class JwtAuthenticationFilter(
    private val jwtProvider: JwtProvider
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val token = extractToken(request)
        if (token != null && jwtProvider.validateToken(token)) {
            val authentication = jwtProvider.getAuthentication(token)
            SecurityContextHolder.getContext().authentication = authentication
        }
        filterChain.doFilter(request, response)
    }

    private fun extractToken(request: HttpServletRequest): String? {
        val bearerToken = request.getHeader("Authorization")
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7)
        }
        return null
    }
}
```

### 5.2 CurrentUser Resolver

Substitutes the ubiquitous `@CurrentUser()` mapping decorator capabilities natively routing inside handler arguments.

```kotlin
// src/main/kotlin/com/example/backend/common/resolver/CurrentUserArgumentResolver.kt
package com.example.backend.common.resolver

import org.springframework.core.MethodParameter
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.web.bind.support.WebDataBinderFactory
import org.springframework.web.context.request.NativeWebRequest
import org.springframework.web.method.support.HandlerMethodArgumentResolver
import org.springframework.web.method.support.ModelAndViewContainer

@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class CurrentUser

class CurrentUserArgumentResolver : HandlerMethodArgumentResolver {

    override fun supportsParameter(parameter: MethodParameter): Boolean {
        return parameter.hasParameterAnnotation(CurrentUser::class.java)
    }

    override fun resolveArgument(
        parameter: MethodParameter,
        mavContainer: ModelAndViewContainer?,
        webRequest: NativeWebRequest,
        binderFactory: WebDataBinderFactory?
    ): Any? {
        val authentication = SecurityContextHolder.getContext().authentication
        if (authentication == null || !authentication.isAuthenticated) {
            return null
        }
        return authentication.principal // Assuming mapping returns internal domain structure User model equivalents
    }
}
```

---

## 6. Database & Repository

Spring Data interfaces serve as structural substitutes matching Prisma operations utilizing underlying active definitions mapping Entities effectively.

```kotlin
// Entity
@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue
    val id: UUID? = null,

    @Column(unique = true, nullable = false)
    val email: String,

    @Column(nullable = false)
    val role: String = "viewer",

    var isActive: Boolean = true
)

// Repository
interface UsersRepository : JpaRepository<User, UUID> {
    fun findByEmail(email: String): User?
}
```

---

## 7. Comparative Stack Conversion (NestJS vs Spring Boot Kotlin)

| Target Concept                     | NestJS Equivalent                    | Kotlin + Spring Boot Implementation                         |
| ---------------------------------- | ------------------------------------ | ----------------------------------------------------------- |
| Environment Variable Binding       | `ConfigModule` + `configuration.ts`  | `application.yml` + `@ConfigurationProperties`              |
| Dependency Injection               | `@Injectable()` + Constructor Target | `@Component` + Primary constructor operations               |
| Global Errors Management           | `HttpExceptionFilter`                | `@RestControllerAdvice`                                     |
| Formatted Return Payloads          | `ResponseTransformInterceptor`       | Manually wrapping with `ApiResponse<T>`                     |
| Payload Parameter Constraints      | `class-validator` + `ValidationPipe` | Jakarta Validation mapping appending `@field:` designations |
| Primary ORM                        | Prisma                               | Spring Data JPA                                             |
| Unstructured Persistence (ODM)     | Mongoose                             | Spring Data MongoDB                                         |
| Framework API Documentation Output | `@nestjs/swagger` decorators         | Springdoc + `@Schema`                                       |
| Core Authentication Middleware     | Passport Module + Guards             | Spring Security configurations alongside Custom Filters     |

---

## 8. Springdoc OpenAPI Patterns

### 8.1 DTO @Schema Blueprinting

- Mandatory variables necessitate validations like `@field:NotBlank`
- Include static context utilizing `@Schema(description = ..., example = ...)` globally
- Enum structures obligate declaration indicating allowed scope parameters through `allowableValues`
- Embed meta-descriptions atop fundamental class architectures leveraging `@Schema(description = "...")`

```kotlin
@Schema(description = "User Creation Request Matrix")
data class CreateUserRequest(
    @field:NotBlank
    @field:Email
    @Schema(description = "Contact e-mail address", example = "user@example.com")
    val email: String,

    @Schema(description = "Role designation parameter", example = "viewer", allowableValues = ["admin", "editor", "viewer"])
    val role: String = "viewer",
)
```

### 8.2 Standard Responses

Wrapper integrations leverage `ApiResponse<T>` wrapping successful results consistently. Follows respective equivalency correlating alongside NestJS `ResponseTransformInterceptor`.

```kotlin
// Controller explicitly managing wrapper returns
fun findOne(id: UUID): ApiResponse<UserResponse> {
    val user = usersService.findById(id)
    return ApiResponse(data = user)
}

// Utilizing Pagination mechanisms
fun findAll(pagination: PaginationRequest): PaginatedResult<UserResponse> {
    return usersService.findAll(pagination)
}
```

### 8.3 Controller Swagger Annotations

```kotlin
// src/main/kotlin/com/example/backend/modules/users/UsersController.kt
package com.example.backend.modules.users

import com.example.backend.common.dto.ApiError
import com.example.backend.common.dto.ApiResponse
import com.example.backend.common.dto.PaginatedResult
import com.example.backend.common.dto.PaginationRequest
import com.example.backend.common.resolver.CurrentUser
import com.example.backend.modules.users.dto.CreateUserRequest
import com.example.backend.modules.users.dto.UserResponse
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.web.bind.annotation.*
import java.util.UUID

@Tag(name = "Users")
@RestController
@RequestMapping("/api/users")
class UsersController(
    private val usersService: UsersService,
) {

    @GetMapping
    @Operation(summary = "Fetch user lists", description = "Produces formatted paginated users structure.")
    @ApiResponses(
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "200", description = "Successful Query",
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "401", description = "Authentication Requirement Default",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
    )
    fun findAll(
        @Valid pagination: PaginationRequest,
        @CurrentUser user: Any,
    ): PaginatedResult<UserResponse> =
        usersService.findAll(pagination)

    @GetMapping("/{id}")
    @Operation(summary = "Fetch distinct user identifier")
    @ApiResponses(
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "200", description = "Successful Query",
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "401", description = "Authentication Requirement Default",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "404", description = "User Absent from Query",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
    )
    fun findOne(
        @Parameter(description = "User unique UUID parameter", example = "550e8400-e29b-41d4-a716-446655440000")
        @PathVariable id: UUID,
        @CurrentUser user: Any,
    ): ApiResponse<UserResponse> =
        ApiResponse(data = usersService.findById(id))

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create internal user operations")
    @ApiResponses(
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "201", description = "Execution Successful",
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "400", description = "Payload Integrity Exception Failed",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "401", description = "Authentication Requirement Default",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
        io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "409", description = "Duplicate Address Collision",
            content = [Content(schema = Schema(implementation = ApiError::class))],
        ),
    )
    fun create(
        @Valid @RequestBody dto: CreateUserRequest,
        @CurrentUser user: Any,
    ): ApiResponse<UserResponse> =
        usersService.create(dto)
}
```

### 8.4 Public Endpoint Schema

Exclude public paths lacking token authorization checks using `.permitAll()` within the subsequent `SecurityConfig` components. Appending descriptors explicitly within `@Operation(description = "...")` conveys the omitted verification sequences.

```kotlin
@Tag(name = "Health")
@RestController
@RequestMapping("/api/health")
class HealthController {

    @GetMapping
    @Operation(summary = "Check infrastructure state", description = "Authentication checks nullified")
    fun check(): Map<String, String> = mapOf("status" to "ok")
}
```

### 8.5 Springdoc Checklist Validation Parameters

Execute comprehensive integrity matching validation prior to concluding endpoint logic adjustments:

- [ ] Ensure controller components reflect explicit `@Tag(name = "...")` definitions.
- [ ] Incorporate concise string descriptors mapping into overarching `@Operation(summary = "...")` decorators.
- [ ] Confirm encompassing lists reflecting positive success outcomes intertwined with granular error possibilities inside `@ApiResponses`.
- [ ] Supplement query and URL boundaries defining descriptions employing complementary `@Parameter` variables.
- [ ] Populate overarching data encapsulation constraints employing intrinsic variable tags mapped via `@Schema(description = ..., example = ...)`.
- [ ] Formulate immutable model properties alongside Jakarta definitions (e.g., Use `@field:Email` to implement constraint mappings explicitly to target variables directly).
- [ ] Calibrate mock strings guaranteeing the absolute presence of explicitly corresponding representation formats mimicking valid target environments.

---

## Verification

```bash
# Execute overarching Gradle compilation sequences
cd apps/backend-kt && ./gradlew build

# Proceed targeting explicit Kotlin compilations exclusively
cd apps/backend-kt && ./gradlew compileKotlin

# Verify localized code styling compliances
cd apps/backend-kt && ./gradlew ktlintCheck

# Spin up operations deploying Development infrastructure parameters natively
cd apps/backend-kt && ./gradlew bootRun

# Instantiate testing operations validations
cd apps/backend-kt && ./gradlew test
```
