
# Day18 - 登录认证与会话管理

## 一、登录认证

### 1.1 实现思路

前端传递 username 和 password → 服务器查询数据库 → 校验通过后生成 JWT 令牌返回。

### 1.2 LoginController

```java
@PostMapping("/login")
public Result login(@RequestBody Emp emp) {
    Login login = empService.login(emp);
    if (login != null) {
        return Result.success(login);
    }
    return Result.error("用户名或密码错误");
}
```

### 1.3 Service：查询 + 生成令牌

```java
public Login login(Emp emp) {
    Emp e = empMapper.selectByUsernameAndPassword(emp);
    if (e != null) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("id", e.getId());
        claims.put("username", e.getUsername());
        String jwt = JwtUtils.generateToken(claims);
        return new Login(e.getId(), e.getUsername(), e.getName(), jwt);
    }
    return null;
}
```

- 密码校验在 SQL 层完成：`where username=#{username} and password=#{password}`
- 返回的 `Login` 实体包含 `id / username / name / token`

---

## 二、会话技术

**会话**：用户打开浏览器访问服务器，直到一方断开连接。一次会话可包含多次请求和响应。

**会话跟踪**：服务器识别多次请求是否来自同一浏览器，以便同一次会话的多次请求间共享数据。

### 2.1 Cookie

| 特性 | 说明 |
|------|------|
| 存储位置 | 客户端浏览器 |
| 生命周期 | `set-cookie` 响应头设置，浏览器自动存储 |
| 携带方式 | 每次请求自动通过 `cookie` 请求头携带 |
| 格式 | key-value |

**优点**：HTTP 协议原生支持。

**缺点**：
- 移动端无法使用 Cookie
- 不安全，存储在客户端
- Cookie 不能跨域（协议 / IP / 端口任一维度不同即为跨域）

### 2.2 Session

| 特性 | 说明 |
|------|------|
| 存储位置 | 服务器端 |
| 流程 | 首次请求创建 Session → 响应 `set-cookie: JSESSIONID` → 后续请求通过 ID 找到 Session |
| 格式 | key-value |

**优点**：存储在服务端，安全。

**缺点**：
- 服务器集群下无法使用（Session 只存在首次请求的服务器上，负载均衡可能分发到其他节点）
- 底层依赖 Cookie，具有 Cookie 的同样缺点

### 2.3 Token（令牌）—— 推荐方案

浏览器登录成功 → 服务端生成令牌 → 返回给客户端 → 后续请求携带令牌 → 服务端校验。

| 特性 | 说明 |
|------|------|
| 本质 | 用户合法身份凭证，一个字符串 |
| 存储位置 | 客户端（可存 localStorage / 移动端存储） |
| 校验方式 | 服务端解析令牌，无需查询存储 |

**优点**：
- 支持 PC 端、移动端
- 解决集群环境认证问题（无状态，任意节点均可校验）
- 减轻服务器存储压力（无需服务端存储会话）

---

## 三、JWT 令牌

**JWT（JSON Web Token）**：以 JSON 格式传输信息的令牌标准。

### 3.1 令牌结构

| 部分 | 说明 |
|------|------|
| Header | 令牌类型、签名算法（如 HS256） |
| Payload | 自定义信息（载荷），如用户 id / username |
| Signature | 签名，由 header + payload + 密钥加密生成，保证防篡改 |

- 三部分以 `.` 分隔，Base64 编码
- `=` 是 Base64 的补位符号
- 密钥不对、令牌被篡改、令牌过期 → 解析时均会报错

### 3.2 JwtUtils 工具类

```java
public class JwtUtils {
    private static final String SECRET_KEY = 
        Base64.getEncoder().encodeToString("tianxing".getBytes());
    private static final long EXPIRE_TIME = 12 * 60 * 60 * 1000L;  // 12小时

    // 生成令牌
    public static String generateToken(Map<String, Object> claims) {
        return Jwts.builder()
                .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
                .setClaims(claims)
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE_TIME))
                .compact();
    }

    // 解析令牌
    public static Claims parseToken(String token) {
        return Jwts.parser()
                .setSigningKey(SECRET_KEY)
                .parseClaimsJws(token)
                .getBody();
    }
}
```

| 方法                                        | 说明          |
| ----------------------------------------- | ----------- |
| `signWith(SignatureAlgorithm.HS256, key)` | 指定签名算法和密钥   |
| `setClaims(map)`                          | 添加自定义信息（载荷） |
| `setExpiration(date)`                     | 设置过期时间      |
| `compact()`                               | 生成最终令牌字符串   |

### 3.3 整体流程

```
前端登录 → LoginController → EmpService.login()
  → 查询数据库校验用户名密码
  → 调用 JwtUtils.generateToken(claims)
  → 返回 Login{token} 给前端
```

---

## 四、过滤器 Filter

对请求进行统一拦截，实现登录校验、统一编码处理等功能。

### 4.1 DemoFilter —— Filter 生命周期

```java
@Slf4j
//@WebFilter(urlPatterns = "/*")
public class DemoFilter implements Filter {
    public void init(FilterConfig config)   { /* 初始化 */ }
    public void doFilter(req, resp, chain)  { /* 拦截逻辑 */ chain.doFilter(); }
    public void destroy()                   { /* 销毁 */ }
}
```

| 方法 | 说明 |
|------|------|
| `init()` | 服务器启动时执行一次，初始化 |
| `doFilter()` | 每次请求被拦截时执行 |
| `destroy()` | 服务器关闭时执行，释放资源 |

要激活 `@WebFilter` 注解，需在启动类加 `@ServletComponentScan`。

### 4.2 TokenFilter —— 令牌校验过滤器

```
获取 URL → 判断是否为 /login → 是则放行
  → 获取请求头 token → 为空则 401
  → JwtUtils.parseToken(token) → 失败则 401
  → 校验通过，放行
```

- `filterChain.doFilter()`：放行到下一个过滤器或目标资源
- 校验失败设置 `response.setStatus(401)` 且不调用 `doFilter()`

### 4.3 过滤器链与拦截路径

**执行路径**：放行前逻辑 → `doFilter()` 放行 → 访问资源 → 放行后的逻辑 → 响应给客户端。

| 拦截路径写法 | 含义 |
|------|------|
| `/*` | 拦截所有请求 |
| `/emps/*` | 拦截 /emps/ 下的所有路径 |
| `/login` | 精确匹配 /login |

多个 Filter 组成**过滤器链**，按注册顺序依次执行。

---

## 五、拦截器 Interceptor

拦截器是 Spring 框架提供的拦截机制，需要将类交给 IOC 容器管理。

### 5.1 DemoInterceptor

```java
@Component
public class DemoInterceptor implements HandlerInterceptor {
    public boolean preHandle(...)       { /* 放行前 */ return true; }
    public void postHandle(...)         { /* 放行后 */ }
    public void afterCompletion(...)    { /* 视图渲染完毕后 */ }
}
```

| 方法 | 执行时机 |
|------|----------|
| `preHandle` | Controller 方法执行前，返回 true 放行 |
| `postHandle` | Controller 方法执行后，视图渲染前 |
| `afterCompletion` | 整个请求完成（视图渲染完毕）后 |

### 5.2 TokenInterceptor —— 令牌校验

逻辑与 TokenFilter 完全一致：判断 `/login` → 获取 token → 解析校验 → 401 或放行。

区别在于放行方式：`preHandle` 返回 `true` 放行，`false` 拦截。

### 5.3 WebConfig 注册拦截器

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private TokenInterceptor tokenInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(tokenInterceptor)
                .addPathPatterns("/**");  // 拦截所有路径
    }
}
```

- 实现 `WebMvcConfigurer` 接口，重写 `addInterceptors`
- `addPathPatterns`：指定拦截路径
- `/**` 拦截所有路径，`/*` 只拦截一级路径，`/emp/*` 拦截指定目录一级路径

### 5.4 Filter vs Interceptor

| 对比维度 | Filter | Interceptor |
|------|------|------|
| 归属 | Jakarta Servlet（Web 服务器层） | Spring 框架层 |
| 拦截范围 | 拦截所有请求（含静态资源） | 只拦截进入 Spring 环境的请求 |
| IOC 支持 | 默认不被 Spring 管理 | 可用 `@Component` + `@Autowired` |
| 注册方式 | `@WebFilter` + `@ServletComponentScan` | `WebMvcConfigurer.addInterceptors` |
| 放行方式 | `chain.doFilter()` | `return true` |

---

## 今日总结

| 模块 | 核心要点 |
|------|----------|
| 登录认证 | LoginController → Service 校验用户名密码 → 生成 JWT 返回 |
| 会话技术 | Cookie（客户端，自动携带，不可跨域）→ Session（服务端，集群受限）→ Token（无状态，支持多端+集群） |
| JWT | header + payload + signature 三段式，Base64 编码；JwtUtils 封装生成/解析，HS256 算法签名，12h 过期 |
| Filter | `doFilter()` 拦截逻辑，`chain.doFilter()` 放行；`@WebFilter` + `@ServletComponentScan` 注册 |
| Interceptor | `preHandle` 返回 true/false 控制放行；`WebMvcConfigurer` 注册；只拦截 Spring 环境请求 |
| 拦截路径 | `/*` 所有一级，`/**` 所有层级，`/emps/*` 指定目录一级 |

