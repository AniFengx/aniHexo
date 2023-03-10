---
title: 前端使用iframe嵌套我方h5页面，出现session跨域问题如何解决
date: 2023-03-10 09:12:10
tags: [java,cors]
categories: [cors]
---

### 问题描述

提供给客户对接的H5页面正常都是通过webview或者页面跳转的形式来做，部分客户可能会采用iframe嵌套的形式，如果是H5页面用户信息采用token的形式，不会有影响，但是采用session的形式，会引起session跨域从而导致浏览器无法拿到对应的用户信息

<!-- more -->

### 问题解决

#### SameSite

> Chrome 51 开始，浏览器的 Cookie 新增加了一个 SameSite 属性，用来防止 CSRF 攻击和用户追踪。Cookie 的 SameSite 属性用来限制第三方 Cookie，从而减少安全风险

目前SameSite有三个常用的值``Strict``, ``Lax``, ``None``，他们的作用是对跨站请求的cookie做不同程度的限制，而根据请求的不同，其对应关系如下：

| 请求类型 | 示例 | 默认 | Lax | Strict |
| ---- | ---- | ---- | ---- | ---- | 
| 链接 | ``<a href="..."></a>`` | 发送 Cookie | 发送 Cookie | 不发送 |
| 预加载 | ``<link rel="prerender" href="..."/>`` | 发送 Cookie | 发送 Cookie | 不发送 |
| GET 表单 | ``<form method="GET" action="...">`` | 发送 Cookie | 发送 Cookie | 不发送 |
| POST 表单 | ``<form method="POST" action="...">`` | 发送 Cookie | 不发送 | 不发送 |
| iframe | ``<iframe src="..."></iframe>`` | 发送 Cookie | 不发送 | 不发送 |
| AJAX | ``$.get("...")`` | 发送 Cookie | 不发送 | 不发送 |
| Image | ``<img src="...">`` | 发送 Cookie | 不发送 | 不发送 |

从Chrome 80开始，默认会对cookie设置SameSite=Lax，ChromeLab对此有比较清楚的说明：地址，目前项目中大部分cookie是没有设置任何的SameSite属性，所以可能出现跨站cookie无法传递的情况，需要对这部分内容进行处理，否则，可能出现cookie无法传递导致类似登陆失败问题。

所以默认较新浏览器版本下，没有正常声明处理SameSite属性的系统，浏览器都会默认设置成Lax从而导致iframe中失效

我们只需要设置SameSite=None，同时，从Chrome从76版本开始，开始对设置为None必须和Secure配合使用，所以最后设置内容是

```
SameSite=None;Secure;
```

#### 解决措施

1. 创建自定义CookieSerializer，修改设置SameSite属性的逻辑

``` java
/**
 * 自定义cookie序列化方式
 *
 * @author anifengx
 * @date 2022-04-02 14:21:21
 */
@Slf4j
public class CustomCookieSerializer extends DefaultCookieSerializer {

    /**
     * http协议
     */
    private static final String HTTP = "http";
    /**
     * 阿里云cdn添加header头帮助判断协议类型
     */
    private static final String X_CLIENT_SCHEME = "X-Client-Scheme";
    /**
     * nginx添加header头帮助判断协议类型
     */
    private static final String X_FORWARDED_PROTO = "X-Forwarded-Proto";

    @Override
    public void writeCookieValue(CookieValue cookieValue) {
        HttpServletRequest request = cookieValue.getRequest();
        HttpServletResponse response = cookieValue.getResponse();
        super.writeCookieValue(cookieValue);
        StringBuffer requestURL = request.getRequestURL();
        String scheme = request.getHeader(X_CLIENT_SCHEME);
        log.info("requestURL X-Client-Scheme : [{}] scheme: [{}]", requestURL, scheme);
        if (StringUtils.isBlank(scheme)) {
            scheme = request.getHeader(X_FORWARDED_PROTO);
            log.info("requestURL X-Forwarded-Proto : [{}] scheme: [{}]", requestURL, scheme);
        }
        if (StringUtils.isBlank(scheme)) {
            scheme = request.getScheme();
            log.info("requestURL schema : [{}] scheme: [{}]", requestURL, scheme);
        }
        if (StringUtils.isBlank(scheme) || HTTP.equals(scheme)) {
            return;
        }
        Collection<String> headers = response.getHeaders(HttpHeaders.SET_COOKIE);
        if (CollectionUtils.isEmpty(headers)) {
            return;
        }
        String userAgent = request.getHeader(HttpHeaders.USER_AGENT);
        log.info("shouldSendSameSiteNone start userAgent: [{}] SET_COOKIE: [{}]", userAgent, headers);
        boolean firstHeader = true;
        // there can be multiple Set-Cookie attributes
        for (String header : headers) {
            if (firstHeader) {
                log.info("shouldSendSameSiteNone userAgent: [{}] SET_COOKIE: [{}]", userAgent, header);
                boolean shouldSendSameSiteNone;
                try {
                    shouldSendSameSiteNone = shouldSendSameSiteNone(userAgent);
                } catch (Exception e) {
                    log.error("shouldSendSameSiteNone error userAgent: [{}] SET_COOKIE: [{}]", userAgent, header, e);
                    return;
                }
                log.info("shouldSendSameSiteNone userAgent: [{}] SET_COOKIE: [{}] boolean [{}]", userAgent, header, shouldSendSameSiteNone);
                if (shouldSendSameSiteNone) {
                    header = header.replaceAll("SameSite=Lax", "SameSite=None;secure;");
                    response.setHeader(HttpHeaders.SET_COOKIE, header);
                }
                log.info("shouldSendSameSiteNone change userAgent: [{}] SET_COOKIE: [{}] boolean [{}] headers [{}] ", userAgent, header, shouldSendSameSiteNone, response.getHeaders(HttpHeaders.SET_COOKIE));
                firstHeader = false;
            }
        }
    }

    boolean shouldSendSameSiteNone(String useragent) {
        return !isSameSiteNoneIncompatible(useragent);
    }

    boolean isSameSiteNoneIncompatible(String useragent) {
        return hasWebKitSameSiteBug(useragent) || dropsUnrecognizedSameSiteCookies(useragent);
    }

    boolean hasWebKitSameSiteBug(String useragent) {
        return isIosVersion(12, useragent) || (isMacosxVersion(10, 14, useragent) && (isSafari(useragent) || isMacEmbeddedBrowser(useragent)));
    }

    boolean dropsUnrecognizedSameSiteCookies(String useragent) {
        if (isUcBrowser(useragent)) {
            return !isUcBrowserVersionAtLeast(12, 13, 2, useragent);
        }
        return isChromiumBased(useragent) && isChromiumVersionAtLeast(51, useragent) && !isChromiumVersionAtLeast(67, useragent);
    }

    boolean isIosVersion(int major, String useragent) {
        String pattern = "\\(iP.+; CPU .*OS (\\d+)[\\d]*.*\\) AppleWebKit\\/";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        if (m.find()) {
            log.info("isIosVersion Found value: " + m.group(0));
            log.info("isIosVersion Found value: " + m.group(1));
            return m.group(1).equals(String.valueOf(major));
        }
        return false;
    }

    boolean isMacosxVersion(int major, int minor, String useragent) {
        String pattern = "\\(Macintosh;.*Mac OS X (\\d+)(\\d+)[\\d]*.*\\) AppleWebKit\\/";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        if (m.find()) {
            log.info("isMacosxVersion Found value: " + m.group(0));
            log.info("isMacosxVersion Found value: " + m.group(1));
            log.info("isMacosxVersion Found value: " + m.group(2));
            return m.group(1).equals(String.valueOf(major)) && m.group(2).equals(String.valueOf(minor));
        }
        return false;
    }

    boolean isSafari(String useragent) {
        String pattern = "Version\\/.* Safari\\/";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        return m.find() && !isChromiumBased(useragent);
    }

    boolean isChromiumBased(String useragent) {
        String pattern = "Chrom(e|ium)";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        return m.find();
    }

    boolean isMacEmbeddedBrowser(String useragent) {
        String pattern = "^Mozilla\\/[\\.\\d]+ \\(Macintosh;.*Mac OS X [\\d]+\\) " + "AppleWebKit\\/[\\.\\d]+ \\(KHTML, like Gecko\\)$";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        return m.find();
    }

    boolean isChromiumVersionAtLeast(int major, String useragent) {
        String pattern = "Chrom[^ \\/]+\\/(\\d+)[\\.\\d]* ";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        if (m.find()) {
            int version = new Integer(m.group(1));
            return version >= major;
        }
        return false;
    }

    boolean isUcBrowser(String useragent) {
        String pattern = "UCBrowser\\/";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        return m.find();
    }

    private boolean isUcBrowserVersionAtLeast(int major, int minor, int build, String useragent) {
        String pattern = "UCBrowser\\/(\\d+)\\.(\\d+)\\.(\\d+)[\\.\\d]* ";
        Pattern r = Pattern.compile(pattern);
        Matcher m = r.matcher(useragent);
        if (m.find()) {
            int majorVersion = new Integer(m.group(1));
            int minorVersion = new Integer(m.group(2));
            int buildVersion = new Integer(m.group(3));
            if (majorVersion != major) {
                return majorVersion > major;
            }
            if (minorVersion != minor) {
                return minorVersion > minor;
            }
            return buildVersion >= build;
        }
        return false;
    }
}
```

2. 在spring中注解式注入自定义CookieSerializer

``` java
@Configuration
public class SpringSessionConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        CustomCookieSerializer cookieSerializer = new CustomCookieSerializer();
        cookieSerializer.setCookieMaxAge(60 * 60 * 24 * 30);
        return cookieSerializer;
    }
}
```