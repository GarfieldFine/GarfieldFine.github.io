---
title: Java 使用 JWT（JSON Web Token）实现认证与授权
date: 2025-04-21 20:04:03
top_group_index: 1
categories: 后端
tags: [java, jwt]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/B6A82F123DFB8E13B891DADCE9633317.jpg
---
# Java 使用 JWT（JSON Web Token）实现认证与授权

## **1. 什么是 JWT？**

JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在网络应用间安全地传输信息。它由三部分组成：

- **Header**（头部）：包含 token 类型（如 `JWT`）和签名算法（如 `HS256`、`RS256`）。
- **Payload**（载荷）：存放实际数据，如用户 ID、角色、过期时间等。
- **Signature**（签名）：用于验证 token 的合法性，防止数据篡改。

JWT 通常用于：

- **用户认证**：用户登录后，服务器返回 JWT，客户端后续请求携带该 token 进行身份验证。
- **信息交换**：安全地在不同系统间传递数据。

------

## **2. Java 实现 JWT（使用 JJWT 库）**

JJWT（Java JWT）是一个流行的 Java 库，用于创建和解析 JWT。下面我们一步步实现 JWT 的生成、验证和解析。

### **2.1 添加依赖**

在 `pom.xml `中添加 JJWT 依赖：

#### **Maven**

```XML
 <!--jwt maven坐标-->
        <!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-api -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-impl -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.2</version>
            <scope>runtime</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-jackson -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.2</version>
            <scope>runtime</scope>
        </dependency>
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **2.2 生成 JWT**

```XML
     *
     * @param secretKey 私钥
     * @param ttlMillis jwt有效时间
     * @param map 携带内容
     * @return jwt
     */
    public static String createJwt(String secretKey, long ttlMillis, Map<String,Object> map){
        // 设置签名算法
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;

        // 设置过期时间
        long overTtlMills = System.currentTimeMillis()+ttlMillis;

        // 将过期时间转换为Date类型
        Date date = new Date(overTtlMills);
        // 将密钥转换为SecretKey类型
        SecretKey key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));

        // 创建JWT构建器
        JwtBuilder jwtBuilder = Jwts.builder()
                // 设置JWT中的声明
                .setClaims(map)
                // 设置签名密钥
                .signWith(key, signatureAlgorithm)
                // 设置过期时间
                .setExpiration(date);

        // 返回JWT字符串
        return jwtBuilder.compact();

    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **2.3 解析并验证 JWT** 

```XML
 public static Claims parseJwt(String secretKey, String token){

        return Jwts.parserBuilder()
                .setSigningKey(secretKey.getBytes(StandardCharsets.UTF_8))
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **2.4 编写拦截器**

```XML
@Component
public class JwtTokenInterceptor implements HandlerInterceptor {


    @Autowired
    private JwtProperties jwtProperties;


    @Override
    public boolean preHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull Object handler) throws Exception {

        System.out.println(handler.getClass()); // 输出可能是 $ProxyXX

        // 判断handler是否为HandlerMethod类型
        if(!(handler instanceof HandlerMethod)){
            return true;
        }

        try {
            // 获取请求头中的jwt
            String jwt = request.getHeader(jwtProperties.tokenName);

            // 解析jwt
            Claims claims = JwtUtil.parseJwt(jwtProperties.secretKey, jwt);

            // 获取用户id
            long id = Long.parseLong(claims.get(UserConstant.USERID).toString());

            // 将用户id存入ThreadLocal中
            ThreadStoreUtil.setThreadLocal(id);

            return true;
        } catch (Exception e) {
            // 设置响应状态码为401
            response.setStatus(401);
            // 抛出自定义异常
            throw new BaseException(ErrorCode.User_NotLogin_Error,"用户未登录");
        }
    }

    @Override
    public void afterCompletion(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull Object handler, Exception ex) throws Exception {
        ThreadStoreUtil.removeThreadLocal();
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **2.5** ThreadStoreUtil工具类的定义

```XML
public class ThreadStoreUtil {
    public static ThreadLocal<Long> threadLocal = new ThreadLocal<>();

    // 设置ThreadLocal的值
    public static void setThreadLocal(Long id){
        threadLocal.set(id);
    }

    // 获取ThreadLocal的值
    public static Long getThreadLocal(){
        return threadLocal.get();
    }

    // 移除ThreadLocal的值
    public static void removeThreadLocal(){
        threadLocal.remove();
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**3. JWT 最佳实践**

1. **密钥安全**：
   - 不要硬编码密钥，推荐从环境变量或配置中心读取。
   - 使用 `Keys.secretKeyFor(SignatureAlgorithm.HS256)` 自动生成安全密钥。
2. **Token 有效期**：
   - 设置合理的过期时间（如 24 小时），避免长期有效。
   - 可以使用 Refresh Token 机制延长会话。
3. **HTTPS 传输**：
   - JWT 应通过 HTTPS 传输，防止中间人攻击。
4. **存储方式**：
   - 前端可存于 `localStorage` 或 `HttpOnly Cookie`（防 XSS）。
5. **黑名单机制**：
   - 如果需要强制失效某些 Token，可结合 Redis 实现黑名单。

------

## **4. 总结**

JWT 是一种轻量级、安全的身份认证方案，适用于分布式系统和前后端分离架构。通过 JJWT 库，我们可以方便地在 Java 中生成、解析和验证 JWT。在实际项目中，应结合业务需求合理设计 Token 的存储、刷新和安全策略。
