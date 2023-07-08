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

#### java

``` java
package com.example.canary.util;

import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTCreator;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import com.auth0.jwt.interfaces.Claim;
import com.auth0.jwt.interfaces.DecodedJWT;
import com.example.canary.core.constant.JwtConstant;
import org.springframework.util.CollectionUtils;

import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * jwt工具类
 *
 * @ClassName JwtUtils
 * @Description jwt工具类
 * @Author zhaohongliang
 * @Date 2023-07-05 21:25
 * @Since 1.0
 */
public class JwtUtils {

    private JwtUtils() {

    }

    /**
     * token有效期，默认为2小时，单位(ms)
     */
    private static final int AVAILABLE_TIME = 1000 * 60 * 60 * 2;

    /**
     * 签名密钥
     */
    private static final String TOKEN_SECRET = "test1234";


    /**
     * 创建token
     *
     * @param claim
     * @param audience
     * @return
     */
    public static String createJwtToken(String claim, String... audience) {
        // 有效起始时间
        Date beginTime = new Date();
        // 有效结束时间
        Date endTime = new Date(System.currentTimeMillis() + AVAILABLE_TIME);

        // header
        Map<String, Object> header = new HashMap<>();
        header.put("typ", "JWT");
        header.put("alg", "HS256");

        // 加密算法
        Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);

        return JWT.create()
                // header
                .withHeader(header)
                // payload
                .withAudience(audience)
                .withIssuedAt(beginTime)
                .withExpiresAt(endTime)
                .withClaim("data", claim)
                // sign
                .sign(algorithm);

    }


    /**
     * 创建token
     *
     * @param claimMap
     * @param audience
     * @return
     */
    public static String createJwtToken(Map<String, String> claimMap, String... audience) {
        // 有效起始时间
        Date beginTime = new Date();
        // 有效结束时间
        Date endTime = new Date(System.currentTimeMillis() + AVAILABLE_TIME);

        // header
        Map<String, Object> header = new HashMap<>();
        header.put("typ", "JWT");
        header.put("alg", "HS256");

        // 加密算法
        Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);

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
     * @param token
     * @return
     */
    public static boolean verify(String token) {
        Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);
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
        Map<String, String> claimMap = new HashMap<>();
        claimMap.put(JwtConstant.CLAIM_DATA, "xxx");
        String token = JwtUtils.createJwtToken(claimMap,"123456", "000");
        System.out.println(token);
        Thread.sleep(2000);
        System.out.println(JwtUtils.isExpired(token));
        System.out.println(JwtUtils.verify(token));
        System.out.println(JwtUtils.getClaim(token, JwtConstant.CLAIM_DATA));
    }
}

```

