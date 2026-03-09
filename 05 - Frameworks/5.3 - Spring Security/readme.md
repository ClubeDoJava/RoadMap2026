# 5.3 — Spring Security

Spring Security é o framework de segurança padrão para aplicações Java. Ele cuida de autenticação (quem é você?) e autorização (o que você pode fazer?).

---

## Conceitos de Segurança

### Autenticação vs Autorização

| Conceito | Pergunta | Exemplo |
|---|---|---|
| **Autenticação** | Quem é você? | Login com usuário e senha |
| **Autorização** | O que você pode fazer? | Somente ADMINs acessam `/admin/*` |

### OWASP Top 10 — principais vulnerabilidades web

| # | Vulnerabilidade | Mitigação |
|---|---|---|
| A01 | **Broken Access Control** | Verifique permissões em cada endpoint |
| A02 | **Cryptographic Failures** | Use BCrypt para senhas, HTTPS para tudo |
| A03 | **Injection (SQL, NoSQL, OS)** | PreparedStatement, validação de entrada |
| A04 | **Insecure Design** | Modelagem de ameaças, defense in depth |
| A05 | **Security Misconfiguration** | Princípio do menor privilégio |
| A06 | **Vulnerable Components** | Atualize dependências regularmente |
| A07 | **Auth Failures** | Tokens seguros, logout correto, brute-force protection |
| A08 | **Software Integrity Failures** | Verificar integridade de dependências |
| A09 | **Logging Failures** | Log de eventos de segurança |
| A10 | **SSRF** | Validar e sanitizar URLs de entrada |

---

## Dependência Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT com jjwt -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

---

## Spring Security — Configuração Moderna

A partir do Spring Security 5.7+, não se usa mais `extends WebSecurityConfigurerAdapter`. Toda a configuração é feita via `SecurityFilterChain` bean.

### `SecurityFilterChain`

```java
import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtFilter jwtFilter;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtFilter jwtFilter,
                          UserDetailsService userDetailsService) {
        this.jwtFilter = jwtFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // Desabilita CSRF para APIs REST stateless (JWT não precisa)
            .csrf(csrf -> csrf.disable())

            // Configuração de CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // Sessão stateless: nenhuma sessão HTTP criada
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Regras de autorização
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )

            // Autenticação via UserDetailsService
            .userDetailsService(userDetailsService)

            // Adiciona o filtro JWT antes do filtro de autenticação padrão
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)

            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // custo 12 (mais lento = mais seguro)
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### `UserDetails` e `UserDetailsService`

```java
// Entidade de usuário
@Entity
@Table(name = "usuarios")
public class Usuario implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String senha;

    @Enumerated(EnumType.STRING)
    private Role role;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }

    @Override
    public String getPassword() { return senha; }

    @Override
    public String getUsername() { return email; }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }
}

public enum Role { USER, ADMIN }

// Serviço que carrega usuário do banco
@Service
public class UsuarioDetailsService implements UserDetailsService {

    private final UsuarioRepository repository;

    public UsuarioDetailsService(UsuarioRepository repository) {
        this.repository = repository;
    }

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return repository.findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException(
                        "Usuário não encontrado: " + email));
    }
}
```

### `PasswordEncoder` — por que BCrypt?

```java
PasswordEncoder encoder = new BCryptPasswordEncoder(12);

// Armazenar: nunca salve a senha em texto puro
String senhaHash = encoder.encode("minha_senha_secreta");
// $2a$12$8K1p/a0dR1LXx0W7yFfKjuypC...

// Verificar no login
boolean valida = encoder.matches("minha_senha_secreta", senhaHash); // true

// Por que NÃO usar MD5 ou SHA-1:
// - São algoritmos de hashing rápidos: um GPU moderno testa bilhões de hashes/segundo
// - BCrypt é intencionalmente lento e usa salt automático
// - SHA-256 também é inadequado para senhas pelo mesmo motivo de velocidade
```

---

## JWT (JSON Web Token)

### Estrutura

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbmFAZW1haWwuY29tIiwicm9sZSI6IlVTRVIifQ.abc123
|------ HEADER -------|.|--------------------- PAYLOAD --------------------|.SIGNATURE
```

- **Header:** algoritmo e tipo do token
- **Payload:** claims (dados do usuário)
- **Signature:** garante que o token não foi adulterado

> JWT não é criptografado por padrão — o payload é apenas Base64. Nunca coloque dados sensíveis (senha, CPF) no payload.

### Serviço JWT

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import java.security.Key;
import java.util.*;

@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration-ms:86400000}") // 24h padrão
    private long expirationMs;

    private Key getKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }

    public String gerarToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", userDetails.getAuthorities().iterator().next().getAuthority());

        return Jwts.builder()
                .claims(claims)
                .subject(userDetails.getUsername())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + expirationMs))
                .signWith(getKey())
                .compact();
    }

    public String extrairEmail(String token) {
        return parsearClaims(token).getSubject();
    }

    public boolean tokenValido(String token, UserDetails userDetails) {
        String email = extrairEmail(token);
        return email.equals(userDetails.getUsername()) && !tokenExpirado(token);
    }

    private boolean tokenExpirado(String token) {
        return parsearClaims(token).getExpiration().before(new Date());
    }

    private Claims parsearClaims(String token) {
        return Jwts.parser()
                .verifyWith((javax.crypto.SecretKey) getKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

### Filtro JWT

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;

@Component
public class JwtFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = header.substring(7); // remove "Bearer "

        try {
            String email = jwtService.extrairEmail(token);

            if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(email);

                if (jwtService.tokenValido(token, userDetails)) {
                    var authToken = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            // token inválido ou expirado — continua sem autenticar
        }

        chain.doFilter(request, response);
    }
}
```

### Controller de Autenticação

```java
import org.springframework.http.*;
import org.springframework.security.authentication.*;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthenticationManager authManager;
    private final JwtService jwtService;
    private final UsuarioService usuarioService;

    public AuthController(AuthenticationManager authManager,
                          JwtService jwtService,
                          UsuarioService usuarioService) {
        this.authManager = authManager;
        this.jwtService = jwtService;
        this.usuarioService = usuarioService;
    }

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest request) {
        try {
            authManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            request.email(), request.senha()));
        } catch (BadCredentialsException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body(new LoginResponse(null, "Credenciais inválidas"));
        }

        UserDetails userDetails = usuarioService.loadUserByUsername(request.email());
        String token = jwtService.gerarToken(userDetails);

        return ResponseEntity.ok(new LoginResponse(token, null));
    }

    @PostMapping("/registro")
    public ResponseEntity<UsuarioResponse> registrar(
            @Valid @RequestBody RegistroRequest request) {
        UsuarioResponse usuario = usuarioService.registrar(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(usuario);
    }
}

record LoginRequest(
    @NotBlank String email,
    @NotBlank String senha
) {}

record LoginResponse(String token, String erro) {}
```

### Configuração no `application.yml`

```yaml
jwt:
  secret: ${JWT_SECRET:minha-chave-super-secreta-de-pelo-menos-32-caracteres}
  expiration-ms: 86400000  # 24 horas
```

> Em produção, use variável de ambiente para o `jwt.secret`. A chave deve ter pelo menos 256 bits (32 caracteres).

### Refresh Token — conceito e implementação básica

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @OneToOne
    @JoinColumn(name = "usuario_id", unique = true)
    private Usuario usuario;

    @Column(unique = true, nullable = false)
    private String token;

    @Column(nullable = false)
    private Instant expiracao;

    public boolean expirado() {
        return expiracao.isBefore(Instant.now());
    }
}

@Service
public class RefreshTokenService {

    private static final Duration DURACAO = Duration.ofDays(7);

    private final RefreshTokenRepository repository;

    public RefreshToken criar(Usuario usuario) {
        repository.deleteByUsuario(usuario); // remove token anterior

        RefreshToken token = new RefreshToken();
        token.setUsuario(usuario);
        token.setToken(UUID.randomUUID().toString());
        token.setExpiracao(Instant.now().plus(DURACAO));

        return repository.save(token);
    }

    public RefreshToken verificar(String token) {
        RefreshToken rt = repository.findByToken(token)
                .orElseThrow(() -> new RuntimeException("Refresh token inválido"));

        if (rt.expirado()) {
            repository.delete(rt);
            throw new RuntimeException("Refresh token expirado. Faça login novamente.");
        }

        return rt;
    }
}
```

---

## Method Security

```java
// Habilitar na configuração
@Configuration
@EnableMethodSecurity
public class SecurityConfig {}

// Nos serviços/controllers
@Service
public class ProdutoService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deletar(Long id) { /* ... */ }

    @PreAuthorize("hasRole('ADMIN') or #usuarioId == authentication.principal.id")
    public UsuarioResponse buscar(Long usuarioId) { /* ... */ }

    @PostAuthorize("returnObject.proprietarioId == authentication.principal.id")
    public DocumentoResponse buscarDocumento(Long id) { /* ... */ }

    @Secured("ROLE_ADMIN")  // mais simples, menos flexível
    public void operacaoAdministrativa() { /* ... */ }
}
```

---

## OAuth2 — Login com Google/GitHub

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,email,profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
```

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .defaultSuccessUrl("/dashboard")
            .failureUrl("/login?error")
        )
        .build();
}
```

---

## Keycloak — overview

**Keycloak** é um servidor IAM (Identity and Access Management) open source. É um IdP (Identity Provider) completo.

| Característica | JWT Próprio | Keycloak |
|---|---|---|
| Complexidade | Baixa | Alta |
| Funcionalidades | Básica | SSO, MFA, Social Login, Admin UI |
| Manutenção | Você mantém | Keycloak mantém |
| Recomendado para | APIs simples, startups | Múltiplas aplicações, enterprise |

Quando usar Keycloak: múltiplas aplicações que precisam de SSO (Single Sign-On), autenticação de dois fatores (MFA), gerenciamento de usuários via UI, ou quando você quer delegar toda a segurança para um servidor dedicado.

---

## Exemplo Completo — API com Registro, Login JWT e Rotas Protegidas

### Fluxo completo

```
1. POST /api/v1/auth/registro   → cria conta
2. POST /api/v1/auth/login      → retorna token JWT
3. GET  /api/v1/produtos        → requer token (qualquer autenticado)
4. DELETE /api/v1/produtos/{id} → requer token + role ADMIN
5. GET  /api/v1/perfil          → requer token (dados do próprio usuário)
```

### Testando com curl

```bash
# 1. Registro
curl -X POST http://localhost:8080/api/v1/auth/registro \
  -H "Content-Type: application/json" \
  -d '{"nome": "Ana", "email": "ana@email.com", "senha": "Senha@123"}'

# 2. Login
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "ana@email.com", "senha": "Senha@123"}' \
  | jq -r '.token')

echo "Token: $TOKEN"

# 3. Acessar rota protegida
curl http://localhost:8080/api/v1/produtos \
  -H "Authorization: Bearer $TOKEN"

# 4. Tentar rota de admin (deve retornar 403 se não for ADMIN)
curl -X DELETE http://localhost:8080/api/v1/produtos/1 \
  -H "Authorization: Bearer $TOKEN"

# 5. Sem token (deve retornar 401)
curl http://localhost:8080/api/v1/produtos
```

### Configuração de CORS no `SecurityConfig`

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://meusite.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```
