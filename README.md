# Polling_WebApp

> Spring Boot와 React 기반 polling 웹 애플리케이션 만들기

[참고 링크](<https://www.callicoder.com/spring-boot-spring-security-jwt-mysql-react-app-part-1/>)

<br>

최종적으로 만들어지는 페이지는 아래와 같다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2Fc9DiFu%2Fbtqw7QFP8ud%2F0FqVhmwDaARxsVkxKxAHaK%2Fimg.jpg">

<br>

#### [구성]

- ##### Back-End

  > Spring Boot (Spring Security와 JWT 인증을 활용한 백엔드 서버 구축)
  >
  > Database는 MySQL 활용 (JPA)

- ##### Front-End

  > React

<br>

<br>

### 전체 튜토리얼 구성

---

**Part 1** : 프로젝트 생성 및 기본 도메인 모델과 레포지토리 생성

**Part 2** : JWT 인증과 함께 Spring Security 구성 및 로그인 및 SignUp을위한 Rest API 빌드

**Part 3** :설문 조사 작성, 설문 조사에 대한 투표, 사용자 프로필 검색 등을위한 나머지 API 작성

**Part 4** : React와 Ant 디자인을 사용하여 프론트 엔드 구현하기

<br>

<br>

<br>

## [1부]

> 프로젝트 생성 및 기본 도메인 모델과 레포지토리 생성

<br>

### 스프링 부트 프로젝트 생성하기

---

http://start.spring.io/에 접속해서 프로젝트를 생성해보자

자신의 프로젝트에 맞게 설정해주면 된다.

기본적으로 Dependencies는 Web, JPA, MySQL, Security를 추가했다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FbHV6fP%2Fbtqw7cJl4P9%2FvaCcIz1N2rv0EB8FWnak8K%2Fimg.png">

<br>

프로젝트를 다운받으면, 압축을 풀고 IDE로 프로젝트를 OPEN하자 (튜토리얼은 IntelliJ로 진행함)

<br>

첫 프로젝트 구조는 아래와 같다

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FoCOMD%2Fbtqw7duHB71%2FCGogcNoyaJ04ExP8K0R5q0%2Fimg.png">

<br>

#### dependency 추가하기

프로젝트에 필요한 몇가지 dependency를 추가해야 한다.

`pom.xml`을 열어 dependencies 안에 추가시키자

```
<!-- For Working with Json Web Tokens (JWT) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.0</version>
</dependency>

<!-- For Java 8 Date/Time Support -->
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

<br>

다음으로 `src/main/resources/application.properties`에 우리가 사용할 서버, 데이터베이스, 하이버네이트, 젝슨을 추가하자

```
## Server Properties
server.port= 5000

## Spring DATASOURCE (DataSourceAutoConfiguration & DataSourceProperties)
spring.datasource.url= jdbc:mysql://localhost:3306/polling_app?useSSL=false&serverTimezone=UTC&useLegacyDatetimeCode=false
spring.datasource.username= root
spring.datasource.password= '비밀번호'

## Hibernate Properties

# The SQL dialect makes Hibernate generate better SQL for the chosen database
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.MySQL5InnoDBDialect
spring.jpa.hibernate.ddl-auto = update

## Hibernate Logging
logging.level.org.hibernate.SQL= DEBUG

# Initialize the datasource with available DDL and DML scripts
spring.datasource.initialization-mode=always

## Jackson Properties
spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS= false
spring.jackson.time-zone= UTC
```

<br>

<br>

JPA의 hibernate를 ddl-auto 설정을 update로 설정해놓았다. 이렇게 하면 엔티티에 따라 DB 테이블이 자동으로 업데이트 되어 적용이 가능하다.

<br>

Jackson의 `WRITE_DATES_AS_TIMESTAMPS` 속성은 Java 8 Date/Time 값을 timestamp로 직렬화하지 못하도록 하는데 사용된다.

다음으로는 MYSQL 워크벤치에서 polling_app 이름으로 데이터베이스 스키마를 만들자

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2Fbh4t2p%2Fbtqw79ZtrAz%2FgeDbYHyvFgwSwzd6RhXu1k%2Fimg.png">

<br>

<br>

#### Java 8 날짜 / UTC 시간대 사용하기 위한 부트 구성

도메인 모델에서 Java 8 Date/Time을 사용하기 위해 JPA 2.1을 등록해야 한다. 그러면 해당 필드가 DB에 유지될 때 자동으로 SQL 형식으로 변환이 가능하다.

메인 클래스 `PollsApplication.java`를 아래와 같이 수정하자

```java
package com.example.polls;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.convert.threeten.Jsr310JpaConverters;

import javax.annotation.PostConstruct;
import java.util.TimeZone;

@SpringBootApplication
@EntityScan(basePackageClasses = { 
		PollsApplication.class,
		Jsr310JpaConverters.class 
})
public class PollsApplication {

	@PostConstruct
	void init() {
		TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
	}

	public static void main(String[] args) {
		SpringApplication.run(PollsApplication.class, args);
	}
}
```

<br>

<br>

<br>

### 도메인 모델 만들기

---

사용자가 회원가입을 하고, 로그인하도록 구현할 것이다. 또한 사용자의 권한에 대한 역할도 필요하다.

model 패키지를 만들어서 도메인 모델(User, Role)들을 관리하도록 하자

<br>

#### User 모델

- id : 기본키
- username : 유저 이름
- email : 이메일
- password : 암호화 형식으로 비밀번호 관리
- roles : Role 엔티티와 다대다 관계

`com/example/polls/model/User.java`

```java
package com.example.polls.model;

import com.example.polls.model.audit.DateAudit;
import org.hibernate.annotations.NaturalId;
import javax.persistence.*;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Size;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users", uniqueConstraints = {
        @UniqueConstraint(columnNames = {
            "username"
        }),
        @UniqueConstraint(columnNames = {
            "email"
        })
})
public class User extends DateAudit {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Size(max = 40)
    private String name;

    @NotBlank
    @Size(max = 15)
    private String username;

    @NaturalId
    @NotBlank
    @Size(max = 40)
    @Email
    private String email;

    @NotBlank
    @Size(max = 100)
    private String password;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "user_roles",
            joinColumns = @JoinColumn(name = "user_id"),
            inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();

    public User() {

    }

    public User(String name, String username, String email, String password) {
        this.name = name;
        this.username = username;
        this.email = email;
        this.password = password;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public Set<Role> getRoles() {
        return roles;
    }

    public void setRoles(Set<Role> roles) {
        this.roles = roles;
    }
}
```

> DateAudit은 아래에 추가로 모델을 구현할 것이다.

<br>

<br>

#### Role 모델

- id
- name

`com/example/polls/model/Role.java`

```java
package com.example.polls.model;

import org.hibernate.annotations.NaturalId;
import javax.persistence.*;

@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @NaturalId
    @Column(length = 60)
    private RoleName name;

    public Role() {

    }

    public Role(RoleName name) {
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public RoleName getName() {
        return name;
    }

    public void setName(RoleName name) {
        this.name = name;
    }
}
```

enum 형식으로 작성된 RoleName도 클래스로 구현하자

`com/example/polls/model/RoleName.java`

```java
package com.example.polls.model;

public enum RoleName {
    ROLE_USER,
    ROLE_ADMIN
}
```

ROLE_USER와 ROLE_ADMIN으로 두가지 역할을 나누어 관리할 것이다.

<br>

<br>

#### DateAudit 모델

createdAt과 updatedAt 필드를 가지는 모델이다.

model안에 audit 패키지를 만들어서 관리하자 (JPA Auditing를 관련 모델에 모두 적용시키기 위해)

`com/example/polls/model/audit/DateAudit.java`

```java
package com.example.polls.model.audit;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import javax.persistence.Column;
import javax.persistence.EntityListeners;
import javax.persistence.MappedSuperclass;
import java.io.Serializable;
import java.time.Instant;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@JsonIgnoreProperties(
        value = {"createdAt", "updatedAt"},
        allowGetters = true
)
public abstract class DateAudit implements Serializable {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Instant createdAt) {
        this.createdAt = createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Instant updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```

JPA Auditing 기능을 사용하기 위해선 `@EnableJpaAuditing`를 클래스에 추가하면 된다.

이를 적용할 AuditingConfig 구성 클래스를 작성하여 추가하자

config 패키지를 따로 만들어 관리한다.

`com/example/polls/config/AuditingConfig.java`

```java
package com.example.polls.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@Configuration
@EnableJpaAuditing
public class AuditingConfig {
    // That's all here for now. We'll add more auditing configurations later.
}
```

<br>

<br>

<br>

#### User 및 Role 모델 데이터를 위한 레포지토리 생성하기

이제 도메인 모델을 모두 정의했기 때문에, 이 모델들을 데이터베이스에 유지하고 검색하기 위한 레포지토리를 만들어야 한다.

우리가 만들 모든 레포지토리는 repository 패키지 안에다가 만들 것이다.

`com/example/polls/repository`

<br>

우선, 해당 패키지에서 UserRepository와 RoleRepository 인터페이스를 구현하자

JpaRepository를 상속받아 구현할 것이다.

`com/example/polls/repository/UserRepository.java`

```java
package com.example.polls.repository;

import com.example.polls.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);

    Optional<User> findByUsernameOrEmail(String username, String email);

    List<User> findByIdIn(List<Long> userIds);

    Optional<User> findByUsername(String username);

    Boolean existsByUsername(String username);

    Boolean existsByEmail(String email);
}
```

<br>

`com/example/polls/repository/RoleRepository.java`

```java
package com.example.polls.repository;

import com.example.polls.model.Role;
import com.example.polls.model.RoleName;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(RoleName roleName);
}
```

<br>

<br>

여기까지 잘 진행한 프로젝트 구조는 아래와 같다.

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FeWdeKz%2Fbtqw7PfRXOK%2Fz1p1J2iqfS43rDoxif0YY1%2Fimg.png">

<br>

이제 기본적인 틀을 갖췄다.

서버가 잘 돌아가는지 확인해보자. 루트 디렉토리에서 아래와 같이 명령어를 입력해보자

```
mvnw spring-boot:run
```

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FpcBsu%2Fbtqw8ajOtbW%2FkC32ViH9qsfE5oZojiM5dK%2Fimg.png">

빨간색으로 에러가 나오지 않으면 잘 구동되는 것이다!

만약 에러가 나왔다면 메시지를 잘 확인해서 고쳐야한다. 나는 MySQL 데이터베이스의 password를 잘못입력해서 접근이 안됐었다.

<br>

서버가 실행되면, MySQL 워크벤치에서 `polling_app` 스키마에 우리가 만든 model들이 테이블로 생성된 모습을 확인할 수 있다. JPA를 쓰면 이렇게 테이블을 자동으로 만들어주는 장점을 느낄 수 있다!

<br>

마지막으로, 미리 데이터베이스에서 기본 역할 2가지를 INSERT로 추가해줘야 한다.

```sql
INSERT INTO roles(name) VALUES('ROLE_USER');
INSERT INTO roles(name) VALUES('ROLE_ADMIN');
```

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FY4dBs%2Fbtqw85I5the%2FTazTzi76PbVuUjcd29oHc0%2Fimg.png">

정상적으로 들어간 모습을 확인할 수 있다.

<br>

<br>

<br>

<br>

## [2부]

> JWT 인증과 함께 Spring Security 구성 및 로그인 및 SignUp을위한 Rest API 빌드

<br>

### Spring Security와 JWT를 통한 사용자 인증 구축하기

---

사용자가 웹 애플리케이션에 회원가입하고, 로그인할 수 있는 API를 만들 것이다.



우선 진행하기에 앞서, 어떤 식으로 인증과정을 설계할건지 요약해보자

```
- 새로운 사용자를 Full Name, UserName, Email, Password로 등록할 것이다.

- UserName/Email과 Password를 통해 로그인할 수 있는 API를 작성할 것이다. 이때, 사용자의 자격 증명이 확인되면, API는 JWT 인증 토큰을 생성해 Response 시 반환시켜준다.
클라이언트는 모든 JWT 토큰을 모든 Request 인증 Header에 보내면서 보호 자원에 접근하게 된다.

- 이때 Spring Security를 설정해 보호된 자원에 대한 접근을 제한시킬 것이다.
(로그인, 회원가입 등 정적 리소스에 대한 API는 모든 사용자가 접근 가능)
(설문조사 작성하는 API, 설문조사에 대한 투표는 인증된 사용자만 접근 가능)

- 클라이언트가 유효한 JWT 토큰이 없는데 보호 자원에 접근을 시도하면, 401 오류를 발생시키도록 Spring Security를 설정할 것이다.

- Admin과 User 권한을 나누어 서버의 자원을 보호할 것이다.
(Admin만 설문지를 생성할 수 있다)
(User만 설문지를 투표할 수 있다)
```



Spring Security 구현의 핵심은 config 패키지에서 이루어진다.

해당 패키지 안에 SecurityConfig를 생성하자

`com/example/polls/config/SecurityConfig.java`

```java
package com.example.polls.config;

import com.example.polls.security.CustomUserDetailsService;
import com.example.polls.security.JwtAuthenticationEntryPoint;
import com.example.polls.security.JwtAuthenticationFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.BeanIds;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(
        securedEnabled = true,
        jsr250Enabled = true,
        prePostEnabled = true
)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    CustomUserDetailsService customUserDetailsService;

    @Autowired
    private JwtAuthenticationEntryPoint unauthorizedHandler;

    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter();
    }

    @Override
    public void configure(AuthenticationManagerBuilder authenticationManagerBuilder) throws Exception {
        authenticationManagerBuilder
                .userDetailsService(customUserDetailsService)
                .passwordEncoder(passwordEncoder());
    }

    @Bean(BeanIds.AUTHENTICATION_MANAGER)
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .cors()
                    .and()
                .csrf()
                    .disable()
                .exceptionHandling()
                    .authenticationEntryPoint(unauthorizedHandler)
                    .and()
                .sessionManagement()
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                    .and()
                .authorizeRequests()
                    .antMatchers("/",
                        "/favicon.ico",
                        "/**/*.png",
                        "/**/*.gif",
                        "/**/*.svg",
                        "/**/*.jpg",
                        "/**/*.html",
                        "/**/*.css",
                        "/**/*.js")
                        .permitAll()
                    .antMatchers("/api/auth/**")
                        .permitAll()
                    .antMatchers("/api/user/checkUsernameAvailability", "/api/user/checkEmailAvailability")
                        .permitAll()
                    .antMatchers(HttpMethod.GET, "/api/polls/**", "/api/users/**")
                        .permitAll()
                    .anyRequest()
                        .authenticated();

        // Add our custom JWT security filter
        http.addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);

    }
}
```

아직 해당 코드 내부에서 사용되는 클래스가 구현이 안돼있기 때문에 오류 표시가 많을 것이다.

앞으로 차차 정의해나갈 것이다.

<br>

우선 현재 SecurityConfig 클래스를 보면 상당히 복잡하다. 사용되는 기능에 대해 하나하나 이해해보자.

1. @EnableWebSecurity

   > 프로젝트에서 웹 보안을 가능하도록 해주는 기본 스프링 보안 어노테이션이다.

   <br>

2. @EnableGlobalMethodSecurity

   > 어노테이션에 기반한 ''메소드 레벨 보안''을 가능하도록 해준다. 다음 3가지 유형의 어노테이션을 제공해준다.
   >
   > - securedEnabled : @Secured 어노테이션으로 컨트롤러/서비스 메소드를 보호해줌
   >
   > ```java
   > @Secured("ROLE_ADMIN")
   > public User getAllUsers() {}
   > 
   > @Secured({"ROLE_USER", "ROLE_ADMIN"})
   > public User getUser(Long id) {}
   > 
   > @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
   > public boolean isUsernameAvailable() {}
   > ```
   >
   > 
   >
   > - jsr250Enabled : @RolesAllowed 어노테이션을 사용할 수 있도록 지원해준다. (권한 부여 결정)
   >
   > ```java
   > @RolesAllowed("ROLE_ADMIN")
   > public Poll createPoll() {}  
   > ```
   >
   > 
   >
   > - prePostEnabled : @PreAuthorize와 @PostAuthorize 어노테이션으로 액세스 제어 구문을 기반한 더 복잡한 표현을 가능하도록 해준다.
   >
   > ```java
   > @PreAuthorize("isAnonymous()")
   > public boolean isUsernameAvailable() {}
   > 
   > @PreAuthorize("hasRole('USER')")
   > public Poll createPoll() {}
   > ```
   >

   <br>

3. WebSecurityConfigurerAdapter

   > 스프링 보안의 WebSecurityConfigurer 인터페이스를 구현해주는 클래스다.
   >
   > 기본 보안 구성을 제공해주고, 다른 클래스가 이 클래스를 상속받아 해당 메소드를 오버라이딩하여 보안 구성을 커스터마이징할 수 있다.
   >
   > 현재 프로젝트에서 WebSecurityConfigurerAdapter는 사용자 정의 보안 구성을 제공하기 위해 메소드를 확장하여 오버라이딩 하는 모습이다.

   <br>

4. CustomUserDetailsService

   >사용자 인증을 하거나, 다양한 역할 기반 검사를 수행하려면 Spring Security는 사용자 세부사항을 불러와야 한다.
   >
   >이를 위해 UserDetailsService username이라는 이름을 기반으로 사용자를 로드하는 단일 메소드를 가진 인터페이스로 구성된다.
   >
   >```java
   >UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
   >```

   우리는 UserDetailsService 인터페이스를 구현하는 CustomUserDetailsService를 정의할 것이고, loadUserByname() 메소드를 위한 구현을 제공해야 한다.

   loadUserByUsername() 메소드는 Spring Security가 다양한 인증과 역할 기반 검증을 수행하기 위해 사용하는 UserDetails 객체를 리턴해준다.

   구현에 있어서, 우리는 또한 UserPrincipal 클래스를 UserDetails 인터페이스로 구현해 정의할 것이며,  loadUserByUsername()로부터 UserPrincipal 객체를 리턴받는다.

   <br>

5. JwtAuthenticationEntryPoint

   > 해당 클래스는, 인증이 이루어지지 않은 상태에서 보호 자원에 접근 시도가 들어오면 클라이언트에게 401 에러를 반환하는데 사용한다. 이는 Spring Security의 AuthenticationEntryPoint 인터페이스에서 구현한다.

   <br>

6. JwtAuthenticationFilter

   > JwtAuthenticationFilter 필터를 구현하는데 사용한다.
   >
   > - 모든 Request의 Authorization Header에서 JWT 인증 토큰을 읽는다.
   > - 토큰의 유효성을 검사한다.
   > - 해당 토큰을 가진 사용자 세부 사항을 로드한다.
   > - Spring Security의 SecurityContext에서 사용자 세부 사항을 설정하고 이를 사용해 권한 검사를 수행한다. 그리고 SecurityContext 컨트롤러에 저장된 사용자 세부 정조에 접근하여 비즈니스 로직을 수행할 수 있다.

   <br>

7. AuthenticationManagerBuilder / AuthentaicationManager

   > AuthenticationManagerBuilder는 AuthentaicationManager 사용자를 인증하기 위해 주요 Spring Security 인터페이스의 인스턴스를 생성하는데 사용된다.
   >
   > 메모리 내 인증, LDAP 인증, JDBC 인증을 작성하거나 사용자 인증 제공자를 추가하는데 사용된다.
   >
   > 우리 프로젝트에서는 customUserDetailsService와 passwordEncoder를 제공하여 AuthentaicationManager를 구축했다.
   >
   > 따라서 AuthentaicationManager 기반으로 로그인 API에서 사용자 인증을 진행할 것이다.

   <br>

8. HttpSecurity 설정

   > HttpSecurty는 csrf, sessionManagement와 같은 보안 기능을 구성하는데 사용한다. 그리고 다양한 조건에 따른 자원 보호에 규칙을 추가한다.
   >
   > 우리 프로젝트에서는 모든 사용자에게 정적 리소스와 몇몇 공개 API에 대한 접근을 허용해주고, 오직 인증된 사용자에게만 다른 API 접근을 제한할 것이다.
   >
   > 우리는 또한, HttpSecurit 구성에서 JWTAuthenticationEntryPoint와 사용자 정의 JWTAuthenticationFilter를 추가할 것이다.

   <br>

<br>

#### 커스텀 Spring Security 클래스, 필터, 어노테이션 생성하기

여태까지 많은 사용자 정의 클래스와 필터로 Spring Security를 구성했다. 이제 이 클래스들을 하나씩 정의해보자.

앞으로 모든 사용자 정의 보안과 관련된 클래스는 `com.example.polls.security` 패키지에 저장할 것이다.

<br>

<br>

##### 1. 커스텀 Spring Security AuthenticationEntryPoint

우리가 정의할 첫번째 보안 관련 클래스는 JwtAuthenticationEntryPoint다.

AuthenticationEntryPoint 인터페이스를 통해 이에 필요한 commence()를 구현한다.

인증이 필요한 자원에 접근하려고 시도하는 인증되지 않은 사용자에게 예외를 발생시킬 때마다 해당 메소드가 호출된다.

이럴때는 간단히 예외 메시지가 포함된 401 오류를 보여주도록 구현하자

`com/example/polls/security/JwtAuthenticationEntryPoint.java`

```java
package com.example.polls.security;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private static final Logger logger = LoggerFactory.getLogger(JwtAuthenticationEntryPoint.class);
    @Override
    public void commence(HttpServletRequest httpServletRequest,
                         HttpServletResponse httpServletResponse,
                         AuthenticationException e) throws IOException, ServletException {
        logger.error("Responding with unauthorized error. Message - {}", e.getMessage());
        httpServletResponse.sendError(HttpServletResponse.SC_UNAUTHORIZED, e.getMessage());
    }
}
```

<br>

<br>

##### 2. 사용자 정의 Spring Security UserDetails

이번에는 UserDetails에 해당하는 UserPrincipal 클래스를 정의해보자.

이 인스턴스는 UserDetailsService를 통해 반환될 것이다. Spring Security는 UserPrincipal 객체에 저장된 정보를 사용해 인증과 권한 부여를 수행하게 된다.

`com/example/polls/security/UserPrincipal.java`

```java
package com.example.polls.security;

import com.example.polls.model.User;
import com.fasterxml.jackson.annotation.JsonIgnore;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

public class UserPrincipal implements UserDetails {
    private Long id;

    private String name;

    private String username;

    @JsonIgnore
    private String email;

    @JsonIgnore
    private String password;

    private Collection<? extends GrantedAuthority> authorities;

    public UserPrincipal(Long id, String name, String username, String email, String password, Collection<? extends GrantedAuthority> authorities) {
        this.id = id;
        this.name = name;
        this.username = username;
        this.email = email;
        this.password = password;
        this.authorities = authorities;
    }

    public static UserPrincipal create(User user) {
        List<GrantedAuthority> authorities = user.getRoles().stream().map(role ->
                new SimpleGrantedAuthority(role.getName().name())
        ).collect(Collectors.toList());

        return new UserPrincipal(
                user.getId(),
                user.getName(),
                user.getUsername(),
                user.getEmail(),
                user.getPassword(),
                authorities
        );
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        UserPrincipal that = (UserPrincipal) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {

        return Objects.hash(id);
    }
}
```

<br>

<br>

##### 3. 사용자 정의 Spring Security UserDetailService

UserDetailsService 인터페이스로 사용자 이름이 주어지면, 해당 사용자의 데이터를 로드하는 CustomUserDetailsService 클래스를 정의하자

`com/example/polls/security/CustomUserDetailsService.java`

```java
package com.example.polls.security;

import com.example.polls.model.User;
import com.example.polls.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    UserRepository userRepository;

    @Override
    @Transactional
    public UserDetails loadUserByUsername(String usernameOrEmail)
            throws UsernameNotFoundException {
        // Let people login with either username or email
        User user = userRepository.findByUsernameOrEmail(usernameOrEmail, usernameOrEmail)
                .orElseThrow(() -> 
                        new UsernameNotFoundException("User not found with username or email : " + usernameOrEmail)
        );

        return UserPrincipal.create(user);
    }

    // This method is used by JWTAuthenticationFilter
    @Transactional
    public UserDetails loadUserById(Long id) {
        User user = userRepository.findById(id).orElseThrow(
            () -> new UsernameNotFoundException("User not found with id : " + id)
        );

        return UserPrincipal.create(user);
    }
}
```

첫번째로 구현된 loadUserByUsername()은 Spring Security에 의해 사용된다. 해당 메소드 안에 구현되는 findByUsernameOrEmail는 사용자 이름 또는 이메일을 사용해 로그인 할 수 있도록 도와주는 기능을 담당한다.

두번째로 구현된 loadUserById()는 JWTAuthenticationFilter에 의해 사용될 것이다.

<br>

<br>

##### 4. JWT 생성 및 검증을 위한 utility 클래스

다음 클래스는, 사용자가 성공적으로 로그인에 성공한 후, JWT를 생성하고 Request 시 Authorization Header에서 보낸 JWT의 유효성을 검사하는데 사용한다.

`com/example/polls/security/JwtTokenProvider.java`

```java
package com.example.polls.security;

import io.jsonwebtoken.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;
import java.util.Date;

@Component
public class JwtTokenProvider {

    private static final Logger logger = LoggerFactory.getLogger(JwtTokenProvider.class);

    @Value("${app.jwtSecret}")
    private String jwtSecret;

    @Value("${app.jwtExpirationInMs}")
    private int jwtExpirationInMs;

    public String generateToken(Authentication authentication) {

        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();

        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationInMs);

        return Jwts.builder()
                .setSubject(Long.toString(userPrincipal.getId()))
                .setIssuedAt(new Date())
                .setExpiration(expiryDate)
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }

    public Long getUserIdFromJWT(String token) {
        Claims claims = Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token)
                .getBody();

        return Long.parseLong(claims.getSubject());
    }

    public boolean validateToken(String authToken) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(authToken);
            return true;
        } catch (SignatureException ex) {
            logger.error("Invalid JWT signature");
        } catch (MalformedJwtException ex) {
            logger.error("Invalid JWT token");
        } catch (ExpiredJwtException ex) {
            logger.error("Expired JWT token");
        } catch (UnsupportedJwtException ex) {
            logger.error("Unsupported JWT token");
        } catch (IllegalArgumentException ex) {
            logger.error("JWT claims string is empty.");
        }
        return false;
    }
}
```

이 클래스는, properties로부터 JWT의 비밀 키와 만료 시간을 읽는다.

`application.properties` 파일에다가 jwtSecret과 jwtExpirationlnMs 속성을 추가하자

##### JWT 등록 정보

`src/main/resources/application.properties`

```
## App Properties
app.jwtSecret= JWTSuperSecretKey
app.jwtExpirationInMs = 604800000
```

<br>

<br>

##### 5. 사용자 정의 Spring Security 인증 필터

JwtAuthenticationFilter에서 Request시 JWT 토큰을 가져와서 유효성을 검사하고, 토큰과 연관된 사용자를 로드하여 Spring Security에 전달한다.

`com/example/polls/security/JwtAuthenticationFilter.java`

```java
package com.example.polls.security;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenProvider tokenProvider;

    @Autowired
    private CustomUserDetailsService customUserDetailsService;

    private static final Logger logger = LoggerFactory.getLogger(JwtAuthenticationFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = getJwtFromRequest(request);

            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                Long userId = tokenProvider.getUserIdFromJWT(jwt);

                UserDetails userDetails = customUserDetailsService.loadUserById(userId);
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication in security context", ex);
        }

        filterChain.doFilter(request, response);
    }

    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7, bearerToken.length());
        }
        return null;
    }
}
```

filter의 Authorization Header에서 가져온 JWT를 파싱하고, 사용자 ID를 얻는다.

그 이후 데이터베이스에서 사용자의 세부정보를 로드하고 Spring Security Context 내에서 인증을 설정한다.

JWT 클레임 내에서 사용자의 사용자 이름과 역할을 인코딩하고 UserDetails를 파싱하여 개체를 만들 수 있는데, 이를 통해 데이터베이스의 손상을 피할 수 있게 된다.

<br>

<br>

##### 6. 현재 로그인 한 사용자에게 접근할 수 있는 사용자 정의 어노테이션

Spring Security는 컨트롤러에서 인증된 사용자에게 접근할 수 있는 @AuthenticationPrincipal 어노테이션을 제공한다.

CurrentUser 어노테이션은 @AuthenticationPrincipal 어노테이션으로 감싸져 있다.

`com/example/polls/security/CurrentUser.java`

```java
package com.example.polls.security;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import java.lang.annotation.*;

@Target({ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@AuthenticationPrincipal
public @interface CurrentUser {

}
```

우리는 이 클래스를 통해 프로젝트에서 Spring Security와 관련된 어노테이션을 너무 많이 사용하지 않도록 구성했다. 의존성을 줄일 수 있고, Spring Security 제거 시 단순히 CurrentUser 어노테이션을 변경하는 방법으로 수행할 수 있게 된다.

<br>

<br>

이제 프로젝트에서 필요한 Spring Security의 구성이 완료되었다. 상당히 복잡하다.. 하지만 모두 중요한 로직이고 기본적인 것으로 프로젝트를 짜면서 이해하도록 노력하자!

<br>

<br>

<br>

### 로그인 및 회원가입 API 작성

---

API를 정의하기 전에, API에서 사용할 Request와 Response 페이로드를 정의하자

해당 클래스들은 `com.example.polls.payload` 패키지 안에 넣는다.

<br>

#### Request 페이로드

1.LoginRequest

`com/example/polls/payload/LoginRequest.java`

```java
package com.example.polls.payload;

import javax.validation.constraints.NotBlank;

public class LoginRequest {
    @NotBlank
    private String usernameOrEmail;

    @NotBlank
    private String password;

    public String getUsernameOrEmail() {
        return usernameOrEmail;
    }

    public void setUsernameOrEmail(String usernameOrEmail) {
        this.usernameOrEmail = usernameOrEmail;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

<br>

2.SignUpRequest

`com/example/polls/payload/SignUpRequest.java`

```java
package com.example.polls.payload;

import javax.validation.constraints.*;

public class SignUpRequest {
    @NotBlank
    @Size(min = 4, max = 40)
    private String name;

    @NotBlank
    @Size(min = 3, max = 15)
    private String username;

    @NotBlank
    @Size(max = 40)
    @Email
    private String email;

    @NotBlank
    @Size(min = 6, max = 20)
    private String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

<br>

<br>

#### Response 페이로드

1.JwtAuthenticationResponse

`com/example/polls/payload/JwtAuthenticationResponse.java`

```java
package com.example.polls.payload;

public class JwtAuthenticationResponse {
    private String accessToken;
    private String tokenType = "Bearer";

    public JwtAuthenticationResponse(String accessToken) {
        this.accessToken = accessToken;
    }

    public String getAccessToken() {
        return accessToken;
    }

    public void setAccessToken(String accessToken) {
        this.accessToken = accessToken;
    }

    public String getTokenType() {
        return tokenType;
    }

    public void setTokenType(String tokenType) {
        this.tokenType = tokenType;
    }
}
```

<br>

2.ApiResponse

`com/example/polls/payload/ApiResponse.java`

```java
package com.example.polls.payload;

public class ApiResponse {
    private Boolean success;
    private String message;

    public ApiResponse(Boolean success, String message) {
        this.success = success;
        this.message = message;
    }

    public Boolean getSuccess() {
        return success;
    }

    public void setSuccess(Boolean success) {
        this.success = success;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

<br>

<br>

#### 커스텀 비즈니스 Exceptions

Request가 유효하지 않거나, 예기치 않은 상황이 발생하면 API가 예외를 발생시켜야 한다.

이를 포함한 다른 예외사항에 대해서도 서로 다른 HTTP 상태 코드로 Response하려고 한다.

이러한 예외는 클래스에서 @ResponseStatus와 함께 정의된다.

> 예외 클래스는 `com.example.polls.exception` 패키지에 넣도록 하자

<br>

1.AppException

`com/example/polls/exception/AppException.java`

```java
package com.example.polls.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public class AppException extends RuntimeException {
    public AppException(String message) {
        super(message);
    }

    public AppException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

<br>

2.BadRequestException

`com/example/polls/exception/BadRequestException.java`

```java
package com.example.polls.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.BAD_REQUEST)
public class BadRequestException extends RuntimeException {

    public BadRequestException(String message) {
        super(message);
    }

    public BadRequestException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

<br>

3.ResourceNotFoundException

`com/example/polls/exception/ResourceNotFoundException.java`

```java
package com.example.polls.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    private String resourceName;
    private String fieldName;
    private Object fieldValue;

    public ResourceNotFoundException( String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s : '%s'", resourceName, fieldName, fieldValue));
        this.resourceName = resourceName;
        this.fieldName = fieldName;
        this.fieldValue = fieldValue;
    }

    public String getResourceName() {
        return resourceName;
    }

    public String getFieldName() {
        return fieldName;
    }

    public Object getFieldValue() {
        return fieldValue;
    }
}
```

에러 처리를 진행하는 클래스들 구현이 끝났다. 다음으로는 전체적인 인증을 관리하는 컨트롤러를 구현해보자

<br>

<br>

#### 인증 컨트롤러

앞으로 구현할 AuthController에서는 로그인과 회원가입을 위한 API가 포함된 전체 코드가 구성된다. 

> 컨트롤러에 해당하는 클래스는 모두 `com.example.polls.controller` 패키지에 넣자

`com/example/polls/controller/AuthController.java`

```java
package com.example.polls.controller;

import com.example.polls.exception.AppException;
import com.example.polls.model.Role;
import com.example.polls.model.RoleName;
import com.example.polls.model.User;
import com.example.polls.payload.ApiResponse;
import com.example.polls.payload.JwtAuthenticationResponse;
import com.example.polls.payload.LoginRequest;
import com.example.polls.payload.SignUpRequest;
import com.example.polls.repository.RoleRepository;
import com.example.polls.repository.UserRepository;
import com.example.polls.security.JwtTokenProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import javax.validation.Valid;
import java.net.URI;
import java.util.Collections;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    AuthenticationManager authenticationManager;

    @Autowired
    UserRepository userRepository;

    @Autowired
    RoleRepository roleRepository;

    @Autowired
    PasswordEncoder passwordEncoder;

    @Autowired
    JwtTokenProvider tokenProvider;

    @PostMapping("/signin")
    public ResponseEntity<?> authenticateUser(@Valid @RequestBody LoginRequest loginRequest) {

        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        loginRequest.getUsernameOrEmail(),
                        loginRequest.getPassword()
                )
        );

        SecurityContextHolder.getContext().setAuthentication(authentication);

        String jwt = tokenProvider.generateToken(authentication);
        return ResponseEntity.ok(new JwtAuthenticationResponse(jwt));
    }

    @PostMapping("/signup")
    public ResponseEntity<?> registerUser(@Valid @RequestBody SignUpRequest signUpRequest) {
        if(userRepository.existsByUsername(signUpRequest.getUsername())) {
            return new ResponseEntity(new ApiResponse(false, "Username is already taken!"),
                    HttpStatus.BAD_REQUEST);
        }

        if(userRepository.existsByEmail(signUpRequest.getEmail())) {
            return new ResponseEntity(new ApiResponse(false, "Email Address already in use!"),
                    HttpStatus.BAD_REQUEST);
        }

        // Creating user's account
        User user = new User(signUpRequest.getName(), signUpRequest.getUsername(),
                signUpRequest.getEmail(), signUpRequest.getPassword());

        user.setPassword(passwordEncoder.encode(user.getPassword()));

        Role userRole = roleRepository.findByName(RoleName.ROLE_USER)
                .orElseThrow(() -> new AppException("User Role not set."));

        user.setRoles(Collections.singleton(userRole));

        User result = userRepository.save(user);

        URI location = ServletUriComponentsBuilder
                .fromCurrentContextPath().path("/api/users/{username}")
                .buildAndExpand(result.getUsername()).toUri();

        return ResponseEntity.created(location).body(new ApiResponse(true, "User registered successfully"));
    }
}
```

<br>

<br>

#### CORS 사용

---

클라이언트로부터 API에 접근이 가능하도록 개발 서버를 실행해야 한다.

cross origin requests를 config 패키지 안에서 WebMvcConfig 클래스를 생성해서 만들자.

`com/example/polls/config/WebMvcConfig.java`

```java
package com.example.polls.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    private final long MAX_AGE_SECS = 3600;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("HEAD", "OPTIONS", "GET", "POST", "PUT", "PATCH", "DELETE")
                .maxAge(MAX_AGE_SECS);
    }
}
```

<br>

<br>

사용자 인증 과정을 처리하는 구현 과정이 상당히 긴 시간이었다..

지금까지 잘 따라왔으면 프로젝트 구조가 아래와 같이 돼야한다.

<br>

<사진>

<br>

이제 프로젝트의 루트 디렉토리에서 다시 서버를 실행해보자

```
mvnw spring-boot:run
```

<br>

<br>

#### 로그인 / 회원가입 테스트

postman을 통해 로그인과 회원가입 API를 테스트해보자

##### 회원가입

<사진>

##### 로그인

<사진>

##### 보호된 API 호출

로그인 API를 활용해 액세스 토큰을 얻은 후에 Authorization은 Request의 Header에 accessToken을 전달하여 보호된 API를 호출할 수 있다.

```
Authorization: Bearer <accessToken>
```

JwtAuthenticationFilter는 Header에서 accessToken을 읽은 후에 해당 API에 대한 접근을 허락/제한할 수 있다.

