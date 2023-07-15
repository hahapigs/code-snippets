## Jwt token

Jwt token 无状态工具类实现，有状态形式可以配合redis实现

#### maven

``` xml
<dependency>
  <groupId>com.auth0</groupId>
  <artifactId>java-jwt</artifactId>
  <version>4.3.0</version>
</dependency>
```

#### yaml

``` yaml
spring:
	token:
  	secret: canary
  	expires: 7200s
```

#### TokenProperties

``` java
package com.example.canary.core.token;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.time.Duration;

/**
 * token properties
 *
 * @since 1.0
 * @author zhaohongliang
 */
@Setter
@Getter
@ConfigurationProperties(prefix = "token" )
public class TokenProperties {

    private TokenProperties() {}

    /**
     * 密钥
     */
    private String secret;

    /**
     * 到期时间
     */
    private Duration expires;
}

```

#### JwtUtils

``` java
package com.example.canary.util;

import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTCreator;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import com.auth0.jwt.interfaces.Claim;
import com.auth0.jwt.interfaces.DecodedJWT;
import com.example.canary.core.token.JwtConstant;
import org.springframework.util.CollectionUtils;

import java.time.Duration;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * jwt工具类
 *
 * @since 1.0
 * @author zhaohongliang
 */
public class JwtUtils {

    private JwtUtils() {

    }

    /**
     * 创建token
     *
     * @param secret 密钥
     * @param expires 过期间隔
     * @param claim 载荷
     * @param audience aud
     * @return
     */
    public static String createJwtToken(String secret, Duration expires, String claim, String... audience) {
        // 有效起始时间
        Date beginTime = new Date();
        // 有效结束时间
        Date endTime = new Date(System.currentTimeMillis() + expires.toMillis());

        // header
        Map<String, Object> header = new HashMap<>();
        header.put("typ", "JWT");
        header.put("alg", "HS256");

        // 加密算法
        Algorithm algorithm = Algorithm.HMAC256(secret);

        return JWT.create()
                // header
                .withHeader(header)
                // payload
                .withAudience(audience)
                .withIssuedAt(beginTime)
                .withExpiresAt(endTime)
                .withClaim(JwtConstant.CLAIM_DATA, claim)
                // sign
                .sign(algorithm);
    }

    /**
     * 创建token
     *
     * @param secret 密钥
     * @param expires 过期时间
     * @param claimMap 载荷map
     * @param audience aud
     * @return
     */
    public static String createJwtToken(String secret, Duration expires, Map<String, String> claimMap, String... audience) {
        // 有效起始时间
        Date beginTime = new Date();
        // 有效结束时间
        Date endTime = new Date(System.currentTimeMillis() + expires.toMillis());

        // header
        Map<String, Object> header = new HashMap<>();
        header.put("typ", "JWT");
        header.put("alg", "HS256");

        // 加密算法
        Algorithm algorithm = Algorithm.HMAC256(secret);

        JWTCreator.Builder builder = JWT.create();
        builder
                // header
                .withHeader(header)
                // payload
                .withAudience(audience)
                .withIssuedAt(beginTime)
                .withExpiresAt(endTime);
        if (!CollectionUtils.isEmpty(claimMap)) {
            claimMap.forEach(builder::withClaim);
        }
        // sign
        return builder.sign(algorithm);

    }

    /**
     * 校验token
     *
     * @param secret
     * @param token
     * @return
     */
    public static boolean verify(String secret, String token) {
        Algorithm algorithm = Algorithm.HMAC256(secret);
        JWTVerifier verifier = JWT.require(algorithm).build();
        try {
            verifier.verify(token);
        } catch (JWTVerificationException e) {
            return false;
        }
        return true;
    }

    /**
     * 校验是否过期
     *
     * @param token
     * @return
     */
    public static boolean isExpired(String token) {
        DecodedJWT decodedJwt = JWT.decode(token);
        // 如果过期时间小于当前时间，则表示已过期
        return decodedJwt.getExpiresAt().getTime() < System.currentTimeMillis();
    }

    /**
     * 校验是否过期
     *
     * @param expiresAt
     * @return
     */
    public static boolean isExpired(Date expiresAt) {
        // 如果过期时间在当前日期之前，则表示已过期
        return expiresAt.before(new Date());
    }

    /**
     * 获取 aud
     *
     * @param token
     * @param index
     * @return
     */
    public static String getAudience(String token, int index) {
        DecodedJWT decodedJwt = JWT.decode(token);
        List<String> audiences = decodedJwt.getAudience();
        if (CollectionUtils.isEmpty(audiences)) {
            return null;
        }
        return decodedJwt.getAudience().get(index);
    }

    /**
     * 获取载荷
     *
     * @param token
     * @param keyName
     * @return
     */
    public static Claim getClaim(String token, String keyName) {
        DecodedJWT decodedJwt = JWT.decode(token);
        return decodedJwt.getClaim(keyName);
    }

    /**
     * 获取载荷字符串
     *
     * @param token
     * @param keyName
     * @return
     */
    public static String getClaimStr(String token, String keyName) {
        return JwtUtils.getClaim(token, keyName).asString();
    }

    /**
     * 获取过期时间
     *
     * @param token
     * @return
     */
    public static Date getExpiresAt(String token) {
        return JWT.decode(token).getExpiresAt();
    }

    public static void main(String[] args) throws InterruptedException {

        String secret = "test1234";
        Duration expires = Duration.ofMillis(1000);

        String token = JwtUtils.createJwtToken(secret, expires, "test","123456", "000");
        System.out.println(token);
        Thread.sleep(2000);
        System.out.println(JwtUtils.isExpired(token));
        System.out.println(JwtUtils.verify(secret, token));
        System.out.println(JwtUtils.getClaim(token, JwtConstant.CLAIM_DATA));
    }
}

```

#### TokenInterceptor

``` java
package com.example.canary.core.token;

import com.example.canary.core.constant.HeaderConstant;
import com.example.canary.core.context.CanaryContext;
import com.example.canary.core.context.CurrentUser;
import com.example.canary.core.exception.ResultCodeEnum;
import com.example.canary.core.exception.ResultEntity;
import com.example.canary.sys.entity.UserVO;
import com.example.canary.util.JwtUtils;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.constraints.NotNull;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.util.StringUtils;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import java.io.IOException;
import java.io.Writer;

/**
 * 拦截器
 *
 * @since 1.0
 * @author zhaohongliang
 */
@Slf4j
public class TokenInterceptor implements HandlerInterceptor {

    @Autowired
    private TokenProperties tokenProperties;

    @Override
    public boolean preHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull Object handler) throws Exception {
        // token
        String token = request.getHeader(HeaderConstant.TOKEN);
        // secret
        String secret = tokenProperties.getSecret();

        // 校验token
        if (!StringUtils.hasText(token) || !JwtUtils.verify(secret, token)) {
            setResponse(response, ResultEntity.fail(ResultCodeEnum.TOKEN_ERROR));
            return false;
        }

        // 载荷
        String claimStr = JwtUtils.getClaimStr(token, JwtConstant.CLAIM_DATA);
        if (!StringUtils.hasText(claimStr)) {
            setResponse(response, ResultEntity.fail(ResultCodeEnum.TOKEN_ERROR));
            return false;
        }

        ObjectMapper objectMapper = new ObjectMapper();
        // user
        UserVO userVo = objectMapper.readValue(claimStr, UserVO.class);
        if (userVo == null) {
            setResponse(response, ResultEntity.fail(ResultCodeEnum.TOKEN_ERROR));
            return false;
        }

        CurrentUser<UserVO> currentUser = new CurrentUser<>(userVo);
        CanaryContext.setCurrentUser(currentUser);

        return HandlerInterceptor.super.preHandle(request, response, handler);
    }

    @Override
    public void postHandle(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(@NotNull HttpServletRequest request, @NotNull HttpServletResponse response, @NotNull Object handler, Exception ex) throws Exception {
        CanaryContext.removeCurrentUser();
    }

    private static void setResponse(HttpServletResponse response, ResultEntity<?> resultEntity) {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        try (Writer writer = response.getWriter()) {
            ObjectMapper objectMapper = new ObjectMapper();
            writer.write(objectMapper.writeValueAsString(resultEntity));
            writer.flush();
        } catch (IOException e) {
            log.error("response异常:" + e);
        }
    }
}

```

