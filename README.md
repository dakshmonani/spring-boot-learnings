# 🔐 JWT Authentication — Spring Security

A complete JWT-based authentication implementation for a Spring Boot e-commerce project. This module handles token generation, request filtering, and unauthorized access responses.

---

## 📁 File Structure

```
src/main/java/com/ecommerce/project/security/jwt/
├── AuthEntryPointJwt.java       # Handles unauthorized access errors
├── AuthTokenFilter.java         # Intercepts & validates every request
└── JwtUtils.java                # Core JWT logic (create, decode, validate)
```

---

## ⚙️ How It Works

Every incoming HTTP request passes through the following flow:

```
Incoming Request
      ↓
AuthTokenFilter          ← Runs FIRST on every request
      ↓
JwtUtils.validateJwtToken()
      ↓
 Valid?  ──── NO ──────→ AuthEntryPointJwt → 401 Unauthorized JSON
      ↓ YES
JwtUtils.getUserNameFromJwtToken()
      ↓
UserDetailsService.loadUserByUsername()
      ↓
SecurityContextHolder.setAuthentication()
      ↓
Request reaches Controller ✅
```

---

## 📄 File Breakdown

### 1. `AuthTokenFilter.java`
> Intercepts every HTTP request before it reaches any controller.

Extends `OncePerRequestFilter` — guarantees it runs **exactly once per request**.

**What it does:**
- Extracts the JWT from the `Authorization` header
- Calls `JwtUtils` to validate the token
- If valid, loads the user from the database and sets them as authenticated in Spring's `SecurityContextHolder`
- Always calls `filterChain.doFilter()` to pass the request forward regardless

```java
// Authorization header format expected:
Authorization: Bearer <your_jwt_token>
```

| Method | Description |
|---|---|
| `doFilterInternal()` | Main filter logic — runs on every request |
| `parseJwt()` | Helper that extracts the raw JWT string from the request header |

---

### 2. `JwtUtils.java`
> The core engine — creates, decodes, and validates JWT tokens.

Reads configuration from `application.properties` via `@Value`.

**Required properties:**
```properties
spring.app.jwtSecret=<your_base64url_encoded_secret>
spring.app.jwtExpirationMs=86400000
```

| Method | Description |
|---|---|
| `key()` | Converts the Base64URL secret string into a cryptographic `SecretKey` |
| `generateTokenFromUserName()` | Creates a signed JWT token for a given user |
| `getJwtFromHeader()` | Reads and strips `Bearer ` from the Authorization header |
| `getUserNameFromJwtToken()` | Decodes the token and returns the embedded username |
| `validateJwtToken()` | Validates the token and catches all failure cases |

**Token validation catches:**

| Exception | Meaning |
|---|---|
| `MalformedJwtException` | Token structure is invalid |
| `ExpiredJwtException` | Token has passed its expiry time |
| `UnsupportedJwtException` | Token uses an unsupported algorithm |
| `IllegalArgumentException` | Token string is null or empty |

---

### 3. `AuthEntryPointJwt.java`
> Called automatically by Spring Security when an unauthenticated request hits a protected endpoint.

Implements `AuthenticationEntryPoint` — Spring calls `commence()` whenever authentication fails.

**Returns a structured JSON error response:**
```json
{
  "status": 401,
  "Error": "Unauthorized",
  "Message": "Full authentication is required to access this resource",
  "path": "/api/products"
}
```

| Method | Description |
|---|---|
| `commence()` | Triggered by Spring Security; builds and writes the 401 JSON response |

---

## 🔑 JWT Token Structure

A JWT is made of three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9  .  eyJzdWIiOiJqb2huX2RvZSIsImlhdCI6Li4ufQ  .  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
       HEADER                          PAYLOAD                                        SIGNATURE
```

| Part | Contains |
|---|---|
| Header | Algorithm used (`HS256`) |
| Payload | `sub` (username), `iat` (issued at), `exp` (expiry) |
| Signature | HMAC-SHA256 of header + payload, signed with your secret key |

---

## 🚀 Usage

Once configured, protect your endpoints in your `SecurityConfig`:

```java
// Public endpoints (no token needed)
.requestMatchers("/api/auth/**").permitAll()

// Protected endpoints (token required)
.anyRequest().authenticated()
```

To get a token, hit your login endpoint and include it in subsequent requests:

```bash
# Login
POST /api/auth/signin
{ "username": "john", "password": "secret" }

# Use the token
GET /api/products
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

---

## 🧩 Dependencies

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.x</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.x</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.x</version>
</dependency>
```

---

## 📝 Source Code

### `AuthEntryPointJwt.java`

```java
package com.ecommerce.project.security.jwt;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;
import tools.jackson.databind.ObjectMapper;

import java.io.IOException;
import java.util.HashMap;

@Component
public class AuthEntryPointJwt implements AuthenticationEntryPoint {
    private static final Logger logger = LoggerFactory.getLogger(AuthEntryPointJwt.class);

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        logger.debug("Unauthorized Error : {} " ,authException.getMessage());

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        HashMap<String, Object> body = new HashMap<>();
        body.put("status", HttpServletResponse.SC_UNAUTHORIZED);
        body.put("Error", "Unauthorized");
        body.put("Message", authException.getMessage());
        body.put("path", request.getServletPath());

        final ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

---

### `JwtUtils.java`

```java
package com.ecommerce.project.security.jwt;

import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.MalformedJwtException;
import io.jsonwebtoken.UnsupportedJwtException;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.security.Key;
import java.util.Date;

@Component
public class JwtUtils {
    private static final Logger logger = LoggerFactory.getLogger(JwtUtils.class);

    @Value("${spring.app.jwtExpirationMs}")
    private int jwtExpirationMs;

    @Value("${spring.app.jwtSecret}")
    private String jwtSecret;

    // Getting jwt from header
    public String getJwtFromHeader(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        logger.debug("Authorization Header {} ", bearerToken);

        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    // Generating Token from username
    public String generateTokenFromUserName(UserDetails userDetails) {
        String username = userDetails.getUsername();
        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(new Date().getTime() + jwtExpirationMs))
                .signWith(key())
                .compact();
    }

    // Getting username from jwt token
    public String getUserNameFromJwtToken(String token) {
        return Jwts.parser()
                .verifyWith((SecretKey) key())
                .build().parseSignedClaims(token)
                .getPayload().getSubject();
    }

    // Generating Signing Key
    public Key key() {
        return Keys.hmacShaKeyFor(Decoders.BASE64URL.decode(jwtSecret));
    }

    // Validate JWT token
    public boolean validateJwtToken(String authToken) {
        try {
            System.out.println("Validate");
            Jwts.parser().verifyWith((SecretKey) key()).build().parseSignedClaims(authToken);
            return true;
        } catch (MalformedJwtException malformedJwtException) {
            logger.error("Invalid JWT Token : {}", malformedJwtException.getMessage());
        } catch (ExpiredJwtException expiredJwtException) {
            logger.error("JWT Token is expired : {}", expiredJwtException.getMessage());
        } catch (UnsupportedJwtException unsupportedJwtException) {
            logger.error("JWT Token is unsupported {}", unsupportedJwtException.getMessage());
        } catch (IllegalArgumentException exception) {
            logger.error("JWT claims string is empty {} ", exception.getMessage());
        }
        return false;
    }
}
```

---

### `AuthTokenFilter.java`

```java
package com.ecommerce.project.security.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class AuthTokenFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private UserDetailsService userDetailsService;

    private static final Logger logger = LoggerFactory.getLogger(AuthTokenFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        logger.debug("AuthTokenFilter called for URI {}", request.getRequestURI());
        try {
            String jwt = parseJwt(request);
            if (jwt != null && jwtUtils.validateJwtToken(jwt)) {
                String username = jwtUtils.getUserNameFromJwtToken(jwt);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
                logger.debug("JWT roles {}", userDetails.getAuthorities());
            }
        } catch (Exception e) {
            logger.debug("Cannot set user authentication {}", e);
        }
        filterChain.doFilter(request, response);
    }

    private String parseJwt(HttpServletRequest request) {
        String jwt = jwtUtils.getJwtFromHeader(request);
        logger.debug("AuthenticationTokenFilter.java {}", jwt);
        return jwt;
    }
}
```

---

## 👤 Author

> Part of the **E-Commerce Spring Boot Project**  
> JWT Security Module
