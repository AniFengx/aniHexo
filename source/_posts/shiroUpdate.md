---
title: shiro框架版本升级出现请求400拦截处理
date: 2023-01-18 10:35:03
tags: [java,shiro]
categories: [shiro]
---

### 问题描述

因阿里云服务器安全提示，项目中的shiro框架近期做了一次升级，升级后检查项目运行无任何问题，几天后运营突然反应说管理平台图片全部无法展示，进行了初步排查，发现在请求后端获取图片的请求全部返回400被拦截了，结合最近更新内容锁定目标在shiro版本升级上

{% note info %} 运营需要在管理平台查看图片后直接右键另存为对应中文名的文件，并且要求是原本文件名称。图片默认都上传到了oss，如果直接右键下载会是乱序英文数字文件名，所以在管理平台后端做了一次转发并设置了中文文件名称 {% endnote %}

<!-- more -->

### 问题解决

实际问题是由于shiro新框架新增了几个默认filter，其中`InvalidRequestFilter`会拦截url上的特殊字符，所以只需要修改`InvalidRequestFilter`的方法跳过拦截即可

#### 解决办法

重写`ShiroFilterFactoryBean`

``` Java
public class CustomShiroFilterFactoryBean extends ShiroFilterFactoryBean {

    private static final Logger logger = LoggerFactory.getLogger(CustomShiroFilterFactoryBean.class);

    @Override
    public Class getObjectType() {
        return CustomSpringShiroFilter.class;
    }

    @Override
    protected AbstractShiroFilter createInstance() throws Exception {
        logger.debug("Creating Shiro Filter instance.");

        SecurityManager securityManager = getSecurityManager();
        if (securityManager == null) {
            String msg = "SecurityManager property must be set.";
            throw new BeanInitializationException(msg);
        }

        if (!(securityManager instanceof WebSecurityManager)) {
            String msg = "The security manager does not implement the WebSecurityManager interface.";
            throw new BeanInitializationException(msg);
        }

        FilterChainManager manager = createFilterChainManager();
        //Expose the constructed FilterChainManager by first wrapping it in a
        // FilterChainResolver implementation. The AbstractShiroFilter implementations
        // do not know about FilterChainManagers - only resolvers:
        PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
        chainResolver.setFilterChainManager(manager);

        // URL携带中文400，servletPath中文校验bug
        Map<String, Filter> filterMap = manager.getFilters();
        Filter invalidRequestFilter = filterMap.get(DefaultFilter.invalidRequest.name());
        if (invalidRequestFilter instanceof InvalidRequestFilter) {
            ((InvalidRequestFilter) invalidRequestFilter).setBlockNonAscii(false);
        }
        //Now create a concrete ShiroFilter instance and apply the acquired SecurityManager and built
        //FilterChainResolver.  It doesn't matter that the instance is an anonymous inner class
        //here - we're just using it because it is a concrete AbstractShiroFilter instance that accepts
        //injection of the SecurityManager and FilterChainResolver:
        return new CustomSpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
    }

}
```

将重写的`ShiroFilterFactoryBean`注入进shiro中

``` java
    @Bean(name = "shiroFilter")
    public BaseShiroFactoryBean shiroFilter(@Qualifier(value = "securityManager") SecurityManager securityManager) {
        //根据配置加载是否启用redis
        Boolean redis = customerConfig.getRedis();
        BaseShiroFactoryBean shiroFilterFactoryBean = new CustomShiroFilterFactoryBean();
        Map<String, Filter> filters = shiroFilterFactoryBean.getFilters();
        filters.put(Constants.CUSTOMER_SHIRO_FILTER, customRolesAuthorizationFilter());
        shiroFilterFactoryBean.setFilters(filters);
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //加载默认配置.
        Map<String, String> filterChainDefinitionMap = new HashMap<>(16);
        defaultFilterChain(filterChainDefinitionMap);
        //加载自定义配置
        loadCustomerChain(filterChainDefinitionMap);
        //登录地址
        if (StringUtils.isBlank(customerConfig.getLoginUrl())) {
            shiroFilterFactoryBean.setLoginUrl(Constants.DEFAULT_LOGIN_URL);
        } else {
            shiroFilterFactoryBean.setLoginUrl(customerConfig.getLoginUrl());
        }
        //登录成功后要跳转的链接
        if (StringUtils.isBlank(customerConfig.getSuccessUrl())) {
            shiroFilterFactoryBean.setSuccessUrl(Constants.DEFAULT_INDEX);
        } else {
            shiroFilterFactoryBean.setSuccessUrl(customerConfig.getSuccessUrl());
        }
        //未授权界面;
        if (StringUtils.isBlank(customerConfig.getUnauthorizedUrl())) {
            shiroFilterFactoryBean.setUnauthorizedUrl(Constants.DEFAULT_403);
        } else {
            shiroFilterFactoryBean.setUnauthorizedUrl(customerConfig.getUnauthorizedUrl());
        }
        //加载静态权限链
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        //加载动态权限链
        shiroFilterFactoryBean.setFilterChainDefinitions("");
        return shiroFilterFactoryBean;
    }
```

至此问题解决，一开始曾尝试设置filter不校验的url地址，但是查看源码后发现不支持这么设置，修改源码影响范围较大，遂决定全局关闭BlockNonAscii设置

### 参考文章

[关于springboot:在SpringBoot下升级shiro之后导致中文路径400非法的解决方案](https://lequ7.com/guan-yu-springboot-zai-springboot-xia-sheng-ji-shiro-zhi-hou-dao-zhi-zhong-wen-lu-jing-400-fei-fa-de-jie-jue-fang-an.html "关于springboot:在SpringBoot下升级shiro之后导致中文路径400非法的解决方案")

{% note warning %} 尝试了上述方法，发现未生效 {% endnote %}

[SHIRO1.6升级到1.7后的问题处理](https://www.freesion.com/article/89131489103/ "SHIRO1.6升级到1.7后的问题处理")

主要参考了上述文章