# Autenticação e Autorizacão com Spring Security e JWT

## 1. Adicionando Dependências

Para iniciar o projeto, adicione as dependências necessárias para o banco de dados e a biblioteca para gerar os tokens JWT:

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>4.4.0</version>
</dependency>
```

## 2. Criando a Classe `TokenService`
A classe `TokenService` será responsável por gerar e validar os tokens JWT.

### **Gerando um Token**
Para gerar um token, precisamos criar um algoritmo de criptografia usando `Algorithm.HMAC256()`:

```java
Algorithm algorithm = Algorithm.HMAC256("chave-secreta");
String token = JWT.create()
    .withIssuer("API Teste")
    .withSubject(user.getName())
    .withExpiresAt(Instant.now().plus(1, ChronoUnit.DAYS))
    .sign(algorithm);
```

### **Validando um Token**

```java
Algorithm algorithm = Algorithm.HMAC256("chave-secreta");
String subject = JWT.require(algorithm)
    .withIssuer("API Teste")
    .build()
    .verify(token)
    .getSubject();
```

## 3. Criando a Classe de Filtro de Autenticação
A classe `SecurityFilter` interceptará todas as requisições para validar o token JWT.

### **Recuperando o Token do Header**
```java
private String recoverToken(HttpServletRequest request) {
    String authHeader = request.getHeader("Authorization");
    return authHeader != null ? authHeader.replace("Bearer ", "") : null;
}
```

### **Fluxo do Filtro**
1. Recuperar o token do cabeçalho.
2. Validar o token usando `TokenService`.
3. Buscar as informações do usuário.
4. Criar um objeto `UsernamePasswordAuthenticationToken`.
5. Passar esse objeto para o contexto da aplicação.

```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```

## 4. Criando a Classe `UserDetailsService`
A classe `CustomUserDetailsService` implementa `UserDetailsService` e carrega os usuários pelo username.

```java
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username);
        return new org.springframework.security.core.userdetails.User(
            user.getEmail(),
            user.getPassword(),
            new ArrayList<>());
    }
}
```

## 5. Configuração do Spring Security
Criamos uma classe de configuração anotada com `@Configuration` e `@EnableWebSecurity`.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.POST, "/auth/login").permitAll()
            .anyRequest().authenticated())
        .addFilterBefore(new SecurityFilter(), UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

## 6. Criando o `PasswordEncoder`

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

## 7. Criando o `AuthenticationManager`

```java
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) {
    return authenticationConfiguration.getAuthenticationManager();
}
```

## 8. Criando a `AuthController`

```java
@RestController
@RequestMapping("/auth")
public class AuthController {
    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(loginRequest.getUsername(), loginRequest.getPassword()));
        String token = tokenService.generateToken(authentication.getName());
        return ResponseEntity.ok(token);
    }
}
```

## 9. Criando o `UserController`

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody User user) {
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        userRepository.save(user);
        return ResponseEntity.ok("Usuário cadastrado com sucesso!");
    }
}
```

## 📌 **Conclusão**
Agora sua aplicação está protegida com autenticação JWT e Spring Security! 🚀

