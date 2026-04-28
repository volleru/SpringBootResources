# OAuth2 Security Flow & Java Implementation

## What is OAuth2?

OAuth2 is an **authorization framework** that lets a third-party application obtain limited access to a service on behalf of a user — without exposing the user's credentials.

---

## Core Roles

| Role | Description |
|---|---|
| **Resource Owner** | The user who owns the data |
| **Client** | The app requesting access |
| **Authorization Server** | Issues tokens (e.g., Keycloak, Google Auth) |
| **Resource Server** | Hosts the protected API |

---

## OAuth2 Authorization Code Flow (Most Common)

```
User          Client App         Auth Server        Resource Server
 |                |                   |                    |
 |-- Click Login ->|                   |                    |
 |                |-- Redirect to Auth->|                   |
 |                |   (client_id,       |                   |
 |                |    redirect_uri,    |                   |
 |                |    scope, state)    |                   |
 |<----- Login Page ------------------|                    |
 |-- Enter Credentials --------------->|                   |
 |                |<-- Auth Code -------|                   |
 |                |                    |                    |
 |                |-- Exchange Code -->|                    |
 |                |   (code,           |                    |
 |                |    client_secret)  |                    |
 |                |<-- Access Token ---|                    |
 |                |                    |                    |
 |                |-- API Request with Bearer Token ------->|
 |                |<-- Protected Resource ------------------|
```

### Step-by-Step

1. **Authorization Request** — Client redirects user to Auth Server with `client_id`, `scope`, `redirect_uri`, `state`.
2. **User Login & Consent** — User authenticates and grants permission.
3. **Authorization Code** — Auth Server redirects back with a short-lived `code`.
4. **Token Exchange** — Client sends `code` + `client_secret` to Auth Server, receives `access_token` (and optionally `refresh_token`).
5. **API Call** — Client calls Resource Server using `Authorization: Bearer <access_token>`.
6. **Token Validation** — Resource Server validates the token and returns data.

---

## Grant Types Summary

| Grant Type | Use Case |
|---|---|
| **Authorization Code** | Web/mobile apps with a user |
| **Client Credentials** | Machine-to-machine (no user) |
| **Implicit** | Deprecated — avoid |
| **Resource Owner Password** | Legacy, avoid unless necessary |
| **Authorization Code + PKCE** | Mobile/SPA apps (no client secret) |

---

## Java Code Examples

### 1. Authorization Code Flow — Build the Auth URL

```java
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

public class OAuth2Client {

    private static final String AUTH_URL     = "https://auth.example.com/oauth2/authorize";
    private static final String CLIENT_ID    = "my-client-id";
    private static final String REDIRECT_URI = "https://myapp.com/callback";
    private static final String SCOPE        = "openid profile email";
    private static final String STATE        = "random-csrf-token-abc123"; // must be unique per request

    public static String buildAuthorizationUrl() throws Exception {
        return AUTH_URL
            + "?response_type=code"
            + "&client_id=" + URLEncoder.encode(CLIENT_ID, StandardCharsets.UTF_8)
            + "&redirect_uri=" + URLEncoder.encode(REDIRECT_URI, StandardCharsets.UTF_8)
            + "&scope=" + URLEncoder.encode(SCOPE, StandardCharsets.UTF_8)
            + "&state=" + URLEncoder.encode(STATE, StandardCharsets.UTF_8);
    }

    public static void main(String[] args) throws Exception {
        System.out.println("Redirect user to:\n" + buildAuthorizationUrl());
    }
}
```

---

### 2. Exchange Authorization Code for Access Token

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class TokenExchange {

    private static final String TOKEN_URL     = "https://auth.example.com/oauth2/token";
    private static final String CLIENT_ID     = "my-client-id";
    private static final String CLIENT_SECRET = "my-client-secret";
    private static final String REDIRECT_URI  = "https://myapp.com/callback";

    public static String exchangeCodeForToken(String code) throws Exception {
        String body = "grant_type=authorization_code"
            + "&code=" + code
            + "&redirect_uri=" + REDIRECT_URI
            + "&client_id=" + CLIENT_ID
            + "&client_secret=" + CLIENT_SECRET;

        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(TOKEN_URL))
            .header("Content-Type", "application/x-www-form-urlencoded")
            .POST(HttpRequest.BodyPublishers.ofString(body))
            .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println("Token Response: " + response.body());
        return response.body(); // parse JSON to extract access_token
    }
}
```

---

### 3. Client Credentials Flow (Machine-to-Machine)

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Base64;

public class ClientCredentialsFlow {

    private static final String TOKEN_URL     = "https://auth.example.com/oauth2/token";
    private static final String CLIENT_ID     = "service-client-id";
    private static final String CLIENT_SECRET = "service-client-secret";

    public static String getAccessToken() throws Exception {
        String credentials = Base64.getEncoder()
            .encodeToString((CLIENT_ID + ":" + CLIENT_SECRET).getBytes());

        String body = "grant_type=client_credentials&scope=api.read";

        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(TOKEN_URL))
            .header("Authorization", "Basic " + credentials)
            .header("Content-Type", "application/x-www-form-urlencoded")
            .POST(HttpRequest.BodyPublishers.ofString(body))
            .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        return response.body(); // parse JSON → access_token
    }
}
```

---

### 4. Call a Protected API with Bearer Token

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class ProtectedApiCall {

    public static void callApi(String accessToken) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/v1/user/profile"))
            .header("Authorization", "Bearer " + accessToken)
            .GET()
            .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println("Status: " + response.statusCode());
        System.out.println("Body: " + response.body());
    }
}
```

---

### 5. Spring Boot Resource Server (Validate JWT Tokens)

**`pom.xml` dependency:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**`application.yml`:**
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

**Security Config:**
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}));

        return http.build();
    }
}
```

**Protected Controller:**
```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/api/profile")
    public String profile(@AuthenticationPrincipal Jwt jwt) {
        return "Hello, " + jwt.getClaimAsString("sub");
    }
}
```

---

## Token Types

| Token | Lifetime | Purpose |
|---|---|---|
| **Access Token** | Short (minutes/hours) | Sent with every API request |
| **Refresh Token** | Long (days/weeks) | Exchange for a new access token |
| **ID Token** | Short | OpenID Connect — contains user info |

---

## Security Best Practices

- Always use **HTTPS** — never send tokens over HTTP.
- Validate the `state` parameter to prevent CSRF attacks.
- Use **PKCE** for mobile and SPA clients (no client secret).
- Store tokens in **memory or HttpOnly cookies**, never `localStorage`.
- Set short expiry on access tokens; use refresh tokens to renew.
- Validate JWT signature, issuer (`iss`), audience (`aud`), and expiry (`exp`) on the resource server.
- Revoke refresh tokens on logout.
