# Java (Spring Boot) チートシート

## Spring Boot プロジェクト構成

```
src/
├── main/
│   ├── java/
│   │   └── com/example/demo/
│   │       ├── DemoApplication.java
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       ├── model/
│   │       ├── dto/
│   │       └── config/
│   └── resources/
│       ├── application.yml
│       ├── static/
│       └── templates/
└── test/
    └── java/
```

## アプリケーション起動

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

## アノテーション

### コアアノテーション

```java
// Bean定義
@Component          // 汎用的なBean
@Service            // ビジネスロジック層
@Repository         // データアクセス層
@Controller         // MVCコントローラ
@RestController     // RESTコントローラ（@Controller + @ResponseBody）
@Configuration      // 設定クラス

// 依存性注入
@Autowired          // 自動注入
@Qualifier("name")  // Bean名指定
@Value("${prop}")   // プロパティ値注入

// Bean設定
@Bean               // Beanメソッド定義
@Primary            // 優先Bean
@Scope("prototype") // スコープ指定

// コンポーネントスキャン
@ComponentScan("com.example")
```

### RESTコントローラ

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET /api/users
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    // GET /api/users/{id}
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // GET /api/users?name=Alice
    @GetMapping
    public List<User> searchUsers(@RequestParam String name) {
        return userService.findByName(name);
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@RequestBody @Valid UserDto dto) {
        return userService.create(dto);
    }

    // PUT /api/users/{id}
    @PutMapping("/{id}")
    public User updateUser(
        @PathVariable Long id,
        @RequestBody @Valid UserDto dto
    ) {
        return userService.update(id, dto);
    }

    // DELETE /api/users/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## エンティティ（JPA）

```java
@Entity
@Table(name = "users")
@Data  // Lombok: getter/setter/toString等
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    private Integer age;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    // リレーション
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;

    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles;
}
```

## Repository

```java
// Spring Data JPA
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // メソッド名から自動生成
    List<User> findByName(String name);
    List<User> findByNameAndAge(String name, Integer age);
    List<User> findByAgeGreaterThan(Integer age);
    List<User> findByNameContaining(String keyword);
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
    long countByAge(Integer age);
    void deleteByName(String name);

    // ページング
    Page<User> findByAge(Integer age, Pageable pageable);

    // ソート
    List<User> findByAge(Integer age, Sort sort);

    // カスタムクエリ（JPQL）
    @Query("SELECT u FROM User u WHERE u.age > :age")
    List<User> findUsersOlderThan(@Param("age") Integer age);

    // ネイティブSQL
    @Query(value = "SELECT * FROM users WHERE age > ?1", nativeQuery = true)
    List<User> findUsersOlderThanNative(Integer age);

    // 更新クエリ
    @Modifying
    @Query("UPDATE User u SET u.name = :name WHERE u.id = :id")
    int updateName(@Param("id") Long id, @Param("name") String name);
}

// 使用例
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> findAll() {
        return userRepository.findAll();
    }

    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }

    public User save(User user) {
        return userRepository.save(user);
    }

    public void deleteById(Long id) {
        userRepository.deleteById(id);
    }

    // ページング
    public Page<User> findAllPaged(int page, int size) {
        return userRepository.findAll(
            PageRequest.of(page, size, Sort.by("name").ascending())
        );
    }
}
```

## DTO（Data Transfer Object）

```java
// Lombok使用
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDto {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    private String name;

    @Email(message = "Invalid email format")
    @NotBlank
    private String email;

    @Min(0)
    @Max(150)
    private Integer age;

    // エンティティ変換
    public User toEntity() {
        return User.builder()
            .name(this.name)
            .email(this.email)
            .age(this.age)
            .build();
    }

    public static UserDto fromEntity(User user) {
        return UserDto.builder()
            .name(user.getName())
            .email(user.getEmail())
            .age(user.getAge())
            .build();
    }
}
```

## バリデーション

```java
// DTOにアノテーション
public class UserDto {
    @NotNull
    @NotBlank
    @NotEmpty
    @Size(min = 2, max = 100)
    @Min(0)
    @Max(150)
    @Email
    @Pattern(regexp = "^[0-9]+$")
    @Past  // 過去の日付
    @Future  // 未来の日付
    private String field;
}

// コントローラで検証
@PostMapping
public User create(@RequestBody @Valid UserDto dto) {
    return userService.create(dto);
}

// カスタムバリデーション
@Validated
public class UserDto {
    @CustomConstraint
    private String field;
}

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = CustomValidator.class)
public @interface CustomConstraint {
    String message() default "Invalid value";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class CustomValidator
    implements ConstraintValidator<CustomConstraint, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && value.startsWith("A");
    }
}
```

## 例外処理

```java
// カスタム例外
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// グローバル例外ハンドラ
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationErrors(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> {
            errors.put(error.getField(), error.getDefaultMessage());
        });
        return new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors
        );
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericError(Exception ex) {
        return new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal server error",
            LocalDateTime.now()
        );
    }
}

@Data
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private Object details;
}
```

## 設定ファイル（application.yml）

```yaml
# サーバー設定
server:
  port: 8080
  servlet:
    context-path: /api

# データベース設定
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: password
    driver-class-name: org.postgresql.Driver

  # JPA設定
  jpa:
    hibernate:
      ddl-auto: update  # validate, update, create, create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

  # ロギング
  logging:
    level:
      root: INFO
      com.example: DEBUG
      org.hibernate.SQL: DEBUG

# カスタムプロパティ
app:
  name: My Application
  version: 1.0.0
  api-key: ${API_KEY:default-key}

# プロファイル別設定
---
spring:
  config:
    activate:
      on-profile: dev

server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: prod

server:
  port: 80
```

## Configuration

```java
@Configuration
public class AppConfig {

    @Bean
    public ModelMapper modelMapper() {
        return new ModelMapper();
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    // プロパティ値注入
    @Value("${app.name}")
    private String appName;

    @ConfigurationProperties(prefix = "app")
    @Data
    public static class AppProperties {
        private String name;
        private String version;
        private String apiKey;
    }
}

// CORS設定
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("http://localhost:3000")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("*")
                    .allowCredentials(true);
            }
        };
    }
}
```

## トランザクション

```java
@Service
public class UserService {

    @Transactional  // デフォルト: 例外時ロールバック
    public User createUser(UserDto dto) {
        User user = dto.toEntity();
        return userRepository.save(user);
    }

    @Transactional(readOnly = true)  // 読み取り専用
    public List<User> findAll() {
        return userRepository.findAll();
    }

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        rollbackFor = Exception.class
    )
    public void complexOperation() {
        // ...
    }
}
```

## セキュリティ（Spring Security）

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic()
            .and()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

// UserDetailsService実装
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getEmail())
            .password(user.getPassword())
            .roles(user.getRoles().toArray(new String[0]))
            .build();
    }
}
```

## テスト

```java
// ユニットテスト
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @MockBean
    private UserRepository userRepository;

    @Test
    void testFindById() {
        User user = new User(1L, "Alice", "alice@example.com", 30);
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        Optional<User> result = userService.findById(1L);

        assertTrue(result.isPresent());
        assertEquals("Alice", result.get().getName());
        verify(userRepository, times(1)).findById(1L);
    }
}

// コントローラテスト
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void testGetUser() throws Exception {
        User user = new User(1L, "Alice", "alice@example.com", 30);
        when(userService.findById(1L)).thenReturn(Optional.of(user));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    void testCreateUser() throws Exception {
        UserDto dto = new UserDto("Bob", "bob@example.com", 25);
        User user = new User(2L, "Bob", "bob@example.com", 25);
        when(userService.create(any())).thenReturn(user);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"Bob\",\"email\":\"bob@example.com\",\"age\":25}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(2));
    }
}

// リポジトリテスト
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void testFindByEmail() {
        User user = new User(null, "Alice", "alice@example.com", 30);
        userRepository.save(user);

        Optional<User> result = userRepository.findByEmail("alice@example.com");

        assertTrue(result.isPresent());
        assertEquals("Alice", result.get().getName());
    }
}
```

## スケジューリング

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}

@Component
public class ScheduledTasks {

    // 固定間隔（前回の実行終了から5秒後）
    @Scheduled(fixedDelay = 5000)
    public void taskWithFixedDelay() {
        System.out.println("Fixed delay task");
    }

    // 固定レート（前回の実行開始から5秒後）
    @Scheduled(fixedRate = 5000)
    public void taskWithFixedRate() {
        System.out.println("Fixed rate task");
    }

    // Cron式
    @Scheduled(cron = "0 0 9 * * MON-FRI")  // 平日9時
    public void cronTask() {
        System.out.println("Cron task");
    }

    // 初期遅延
    @Scheduled(initialDelay = 10000, fixedRate = 5000)
    public void taskWithInitialDelay() {
        System.out.println("Task with initial delay");
    }
}
```

## 非同期処理

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class AsyncService {

    @Async
    public CompletableFuture<String> asyncMethod() {
        // 非同期処理
        return CompletableFuture.completedFuture("Result");
    }
}
```

## Lombok アノテーション

```java
@Data                 // @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@Getter               // すべてのフィールドにgetter
@Setter               // すべてのフィールドにsetter
@ToString             // toStringメソッド生成
@EqualsAndHashCode    // equals/hashCode生成
@NoArgsConstructor    // 引数なしコンストラクタ
@AllArgsConstructor   // 全フィールドコンストラクタ
@RequiredArgsConstructor  // final/NotNullフィールドのコンストラクタ
@Builder              // Builderパターン
@Slf4j                // ロガー（log変数）
@Value                // イミュータブルクラス
```

## pom.xml（Maven）

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## build.gradle（Gradle）

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    runtimeOnly 'org.postgresql:postgresql'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```
