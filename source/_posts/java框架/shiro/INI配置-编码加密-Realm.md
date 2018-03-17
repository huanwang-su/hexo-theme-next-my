---
title: shiro配置-编码-加密 
date: 2018/3/16 08:28:25
category:
- java框架
- shiro
tag:
- shiro
comments: true  
---
# INI配置

Shiro是从根对象SecurityManager进行身份验证和授权的；也就是所有操作都是自它开始的，这个对象是线程安全且真个应用只需要一个即可

### java代码初始化 
```
DefaultSecurityManager securityManager = new DefaultSecurityManager(); 

//设置 authenticator 
ModularRealmAuthenticator authenticator = new ModularRealmAuthenticator(); 
authenticator.setAuthenticationStrategy(new AtLeastOneSuccessfulStrategy()); 
securityManager.setAuthenticator(authenticator); 
 
//设置 authorizer 
ModularRealmAuthorizer authorizer = new ModularRealmAuthorizer(); 
authorizer.setPermissionResolver(new WildcardPermissionResolver()); 
securityManager.setAuthorizer(authorizer); 
 
//设置 Realm 
DruidDataSource ds = new DruidDataSource(); 
ds.setDriverClassName("com.mysql.jdbc.Driver"); 
ds.setUrl("jdbc:mysql://localhost:3306/shiro"); 
ds.setUsername("root"); ds.setPassword(""); 
JdbcRealm jdbcRealm = new JdbcRealm(); 
jdbcRealm.setDataSource(ds); 
jdbcRealm.setPermissionsLookupEnabled(true); 
securityManager.setRealms(Arrays.asList((Realm) jdbcRealm));  

//将 SecurityManager 设置到 SecurityUtils 方便全局使用 
SecurityUtils.setSecurityManager(securityManager);  
Subject subject = SecurityUtils.getSubject(); 
```

### 等价INI配置文件（shiro-config.ini） ,类似ioc容器
```
[main]
#authenticator
authenticator=org.apache.shiro.authc.pam.ModularRealmAuthenticator
authenticationStrategy=org.apache.shiro.authc.pam.AtLeastOneSuccessfulStrategy
authenticator.authenticationStrategy=$authenticationStrategy
securityManager.authenticator=$authenticator

#authorizer
authorizer=org.apache.shiro.authz.ModularRealmAuthorizer
permissionResolver=org.apache.shiro.authz.permission.WildcardPermissionResolver
authorizer.permissionResolver=$permissionResolver
securityManager.authorizer=$authorizer

#realm
dataSource=com.alibaba.druid.pool.DruidDataSource
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql://localhost:3306/shiro
dataSource.username=root
dataSource.password=root
jdbcRealm=org.apache.shiro.realm.jdbc.JdbcRealm
jdbcRealm.dataSource=$dataSource
jdbcRealm.permissionsLookupEnabled=true
securityManager.realms=$jdbcRealm

```
### java初始化 
```
Factory<org.apache.shiro.mgt.SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro-config.ini");  // ini 配置文件路径，其支持“classpath:”（类路径）、“file:”（文件系统）、“url:”（网络）三种路径格式，
org.apache.shiro.mgt.SecurityManager securityManager = factory.getInstance();  
//将 SecurityManager设置到SecurityUtils方便全局使用 SecurityUtils.setSecurityManager(securityManager); Subject subject = SecurityUtils.getSubject(); 
```

### INI 配置分类
```
[main] 
#提供了对根对象securityManager及其依赖的配置
securityManager=org.apache.shiro.mgt.DefaultSecurityManager 
………… 
securityManager.realms=$jdbcRealm  

[users] 
#提供了对用户/密码及其角色的配置，用户名=密码，角色1，角色 2 
username=password,role1,role2  

[roles] 
#提供了角色及权限之间关系的配置，角色=权限 1，权限 2 
role1=permission1,permission2  

[urls] 
#用于web，提供了对web url拦截相关的配置，url=拦截器[参数]，拦截器
/index.html = anon 
/admin/** = authc, roles[admin], perms["permission1"] 

```

# 编码/加密 

在涉及到密码存储问题上，应该加密/生成密码摘要存储，而不是存储明文密码

## 编码/解码 
Shiro提供了base64和16进制字符串编码/解码的API支持，方便一些编码解码操作。Shiro内部的一些数据的存储/表示都使用了base64和16进制字符串。  
```
//Base64
String str = "hello"; 
String base64Encoded = Base64.encodeToString(str.getBytes()); 
String str2 = Base64.decodeToString(base64Encoded); 
Assert.assertEquals(str, str2); 

//16进制 
String str = "hello"; 
String hexEncoded = Hex.encodeToString(str.getBytes()); 
String str2 = new String(Hex.decode(hexEncoded.getBytes())); Assert.assertEquals(str, str2); 
```
还有一个可能经常用到的类CodecSupport，提供了toBytes(str,"utf-8")/toString(bytes,"utf-8")用于在byte数组/String 之间转换。

## 散列算法 
用于生成数据的摘要信息，是一种不可逆的算法,如MD5,SHA等。一般进行散列时最好提供一个salt（盐），这样生成的散列值相对来说更难破解

```
String str = "hello";
String salt = "123";
String md5 = new Md5Hash(str, salt).toString();// 还可以转换为toBase64()/toHex(),new Md5Hash(str, salt, 2).toString()
String sha1 = new Sha256Hash(str,salt).toString();//另外还有如 SHA1、SHA512 算法

//Shiro还提供了通用的散列支持：内部使用MessageDigest  
String simpleHash = new SimpleHash("SHA-1", str, salt).toString(); 
```
为了方便使用，Shiro 提供了 HashService，默认提供了 DefaultHashService 实现
```
DefaultHashService hashService = new DefaultHashService(); // 默认算法 SHA-512
hashService.setHashAlgorithmName("SHA-512");
hashService.setPrivateSalt(new SimpleByteSource("123")); // 私盐,默认无,其在散列时自动与用户传入的公盐混合产生一个新盐； 
String 
hashService.setGeneratePublicSalt(true);// 是否生成公盐，默认 false
hashService.setRandomNumberGenerator(new SecureRandomNumberGenerator());//用于生成公盐。默认就这个
hashService.setHashIterations(1);//生成Hash值的迭代次数
HashRequest request = new HashRequest.Builder().setAlgorithmName("MD5")
		.setSource(ByteSource.Util.bytes("hello")).setSalt(ByteSource.Util.bytes("123")).setIterations(2)
		.build();
String hex = hashService.computeHash(request).toHex();
```
## 加密/解密
Shiro还提供对称式加密/解密算法的支持，如AES、Blowfish 等；

AES例子
```
AesCipherService aesCipherService = new AesCipherService(); aesCipherService.setKeySize(128); //设置 key 长度 
//生成 key 
Key key = aesCipherService.generateNewKey(); 
String text = "hello"; //加密 
String encrptText =  aesCipherService.encrypt(text.getBytes(), key.getEncoded()).toHex(); 
//解密 
String text2 =  new String(aesCipherService.decrypt(Hex.decode(encrptText), key.getEncoded()).getBytes());  
Assert.assertEquals(text, text2); 
```

## PasswordService/CredentialsMatcher
Shiro 提供了 PasswordService 及 CredentialsMatcher 用于提供加密密码及验证密码服务。 

Shiro 默认提供了 PasswordService 实现 DefaultPasswordService；CredentialsMatcher 实现 PasswordMatcher 及 HashedCredentialsMatcher（更强大）。 

### DefaultPasswordService 配合 PasswordMatcher 实现简单的密码加密与验证服务 

1. 定义Realm
```
public class MyRealm extends AuthorizingRealm {     
	private PasswordService passwordService;    
	public void setPasswordService(PasswordService passwordService) { 
		this.passwordService = passwordService;    
	}      
	//省略 doGetAuthorizationInfo，具体看代码      
	@Override    
	protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
		return new SimpleAuthenticationInfo("wu",passwordService.encryptPassword("123"),getName());
	}
} 
```
为了方便，直接注入一个 passwordService 来加密密码，实际使用时需要在 Service 层使用 passwordService 加密密码并存到数据库。 

2. 配置ini
```
passwordService=org.apache.shiro.authc.credential.DefaultPasswordService 
hashService=org.apache.shiro.crypto.hash.DefaultHashService 
passwordService.hashService=$hashService

passwordMatcher=org.apache.shiro.authc.credential.PasswordMatcher 
passwordMatcher.passwordService=$passwordService

myRealm=com.github.zhangkaitao.shiro.chapter5.hash.realm.MyRealm
myRealm.passwordService=$passwordService
myRealm.credentialsMatcher=$passwordMatcher  
```

为什么要设置credentialsMatcher?

myRealm间接继承了AuthenticatingRealm，将credentialsMatcher赋值给myRealm，其在调用getAuthenticationInfo方法获取到AuthenticationInfo信息后，会使用credentialsMatcher来验证凭据是否匹配，如果不匹配将抛出IncorrectCredentialsException异常。  

### HashedCredentialsMatcher 实现密码验证服务 
Shiro提供了CredentialsMatcher的散列实现HashedCredentialsMatcher，和之前的PasswordMatcher不同的是，它只用于密码验证，且可以提供自己的盐，而不是随机生成盐
```
SimpleAuthenticationInfo ai = new SimpleAuthenticationInfo(username, password, getName());     ai.setCredentialsSalt(ByteSource.Util.bytes(username+salt2)); //盐是用户名+随机数 
```
通过SimpleAuthenticationInfo的credentialsSalt设置盐，HashedCredentialsMatcher 会自动识别这个盐。 

### 密码重试次数限制 
```
public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
	String username = (String) token.getPrincipal(); // retry count + 1
	Element element = passwordRetryCache.get(username);
	if (element == null) {
		element = new Element(username, new AtomicInteger(0));
		passwordRetryCache.put(element);
	}
	AtomicInteger retryCount = (AtomicInteger) element.getObjectValue();
	if (retryCount.incrementAndGet() > 5) { // if retry count > 5 throw
		throw new ExcessiveAttemptsException();
	}

	boolean matches = super.doCredentialsMatch(token, info);
	if (matches) { // clear retry count
		passwordRetryCache.remove(username);
	}
	return matches;
}
```

# Realm
定义实体及关系 

即用户-角色之间是多对多关系，角色-权限之间是多对多关系；且用户和权限之间通过角 色建立关系


```
public class UserRealm extends AuthorizingRealm {
    private UserService userService = new UserServiceImpl();

    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        String username = (String) principals.getPrimaryPrincipal();
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        authorizationInfo.setRoles(userService.findRoles(username));
        authorizationInfo.setStringPermissions(userService.findPermissions(username));
        return authorizationInfo;
    }

    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        String username = (String) token.getPrincipal();
        User user = userService.findByUsername(username);
        if (user == null) {
            throw new UnknownAccountException();//没找到帐号
        }
        if (Boolean.TRUE.equals(user.getLocked())) {
            throw new LockedAccountException(); //帐号锁定
        }         
		//交给 AuthenticatingRealm 使用 CredentialsMatcher 进行密码匹配，如果觉得人家 的不好可以在此判断或自定义实现
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(user.getUsername(), //用户名
                user.getPassword(), //密码
                ByteSource.Util.bytes(user.getCredentialsSalt()),//salt=username+salt
                getName()  //realm name
        );
        return authenticationInfo;
    }
} 
```
UserRealm父类AuthorizingRealm将获取Subject相关信息分成两步：获取身份验证信息（doGetAuthenticationInfo）及授权信息（doGetAuthorizationInfo）； 

- doGetAuthenticationInfo 获取身份验证相关信息:首先根据传入的用户名获取 User 信 息；然后如果 user 为空，那么抛出没找到帐号异常 UnknownAccountException；如果 user 找到但锁定了抛出锁定异常 LockedAccountException；最后生成 AuthenticationInfo 信息， 交给间接父类 AuthenticatingRealm 使用 CredentialsMatcher 进行判断密码是否匹配，如果不 匹配将抛出密码错误异常 IncorrectCredentialsException；另外如果密码重试此处太多将抛出 超出重试次数异常 ExcessiveAttemptsException；在组装 SimpleAuthenticationInfo 信息时， 需要传入：身份信息（ 用户名）、凭据（ 密 文 密码）、盐（username+salt），CredentialsMatcher 使用盐加密传入的明文密码和此处的密文密码进行匹配。 - 
- doGetAuthorizationInfo 获取授权信息：PrincipalCollection 是一个身份集合，因为我们 现在就一个 Realm，所以直接调用 getPrimaryPrincipal 得到之前传入的用户名即可；然后根 据用户名调用 UserService 接口获取角色及权限信息。  






















