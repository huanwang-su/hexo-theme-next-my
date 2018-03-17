---
title: shiro会话管理-缓存-spring集成
date: 2018/3/16 08:28:25
category:
- java框架
- shiro
tag:
- shiro
comments: true  
---

# 会话管理

显然会话管理原理很简单,通过session管理会话,根据token获取对应会话即可,那token怎么传呢,在tomcat中,session我们通过cookie设置会话id,那么shiro中没有cookie,他是如何做的呢

Shiro提供了完整的企业级会话管理功能，不依赖于底层容器（如web容器tomcat），不管JavaSE还是JavaEE环境都可以使用，提供了会话管理、会话事件监听、会话存储/持久化、容器无关的集群、失效/过期支持、对Web的透明支持、SSO单点登录的支持等特性。即直接使用Shiro的会话管理可以直接替换如Web容器的会话管理。

## 会话 
所谓会话，即用户访问应用时保持的连接关系，在多次交互中应用能够识别出当前访问的 用户是谁，且可以在多次交互中保存一些数据。

	Subject subject = SecurityUtils.getSubject(); 
	Session session = subject.getSession(); 
	
Session接口

	public interface Session {    
		Serializable getId();
		Date getStartTimestamp();
		Date getLastAccessTime();
		long getTimeout() throws InvalidSessionException;
		void setTimeout(long maxIdleTimeInMillis) throws InvalidSessionException;
		String getHost();
		void touch() throws InvalidSessionException; //修改最后一次访问时间
		void stop() throws InvalidSessionException;
		Collection<Object> getAttributeKeys() throws InvalidSessionException;
		Object getAttribute(Object key) throws InvalidSessionException;
		void setAttribute(Object key, Object value) throws InvalidSessionException;
		Object removeAttribute(Object key) throws InvalidSessionException;	
	}

## 会话管理器
会话管理器管理着应用中所有 Subject 的会话的创建、维护、删除、失效、验证等工作。是 Shiro 的核心组件，顶层组件 SecurityManager 直接继承了 SessionManager，且提供了 SessionsSecurityManager 实 现 直 接把会话管理委托给相应的 SessionManager ， DefaultSecurityManager 及 DefaultWebSecurityManager 默认 SecurityManager 都继承了 SessionsSecurityManager。

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fhp9ynny5cj30vx0dnjs7.jpg)

SecurityManager 提供了如下接口： 
	
	Session start(SessionContext context); //启动会话
	Session getSession(SessionKey key) throws SessionException; //根据会话 Key 获取会话 	

另外用于 Web 环境的 WebSessionManager 又提供了如下接口：   

	boolean isServletContainerSessions();//是否使用 Servlet 容器的会话 

Shiro 还提供了 ValidatingSessionManager 用于验资并过期会话： 
	
	void validateSessions();//验证所有会话是否过期
	
## 会话监听器 
会话监听器用于监听会话创建、过期及停止事件

	public interface SessionListener {

		void onStart(Session session);

		void onStop(Session session);

		void onExpiration(Session session);
	}

## 会话存储/持久化 
Shiro 提供 SessionDAO 用于会话的 CRUD，即 DAO（Data Access Object）模式实现
	
	public interface SessionDAO {

		Serializable create(Session session);

		Session readSession(Serializable sessionId) throws UnknownSessionException;

		void update(Session session) throws UnknownSessionException;

		void delete(Session session);
	 
		Collection<Session> getActiveSessions();
	}
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fhpb5w97fxj30nr0eg77i.jpg)
AbstractSessionDAO提供了SessionDAO的基础实现，如生成会话ID等；CachingSessionDAO 提供了对开发者透明的会话缓存的功能，只需要设置相应的 CacheManager 即可； MemorySessionDAO 直接在内存中进行会话维护；而 EnterpriseCacheSessionDAO 提供了缓 存功能的会话维护，默认情况下使用 MapCache 实现，内部使用 ConcurrentHashMap 保存 缓存的会话。 
	
Shiro 提供了使用 Ehcache 进行会话存储，Ehcache 可以配合 TerraCotta 实现容器无关的分布式集群。 
	
	sessionDAO=org.apache.shiro.session.mgt.eis.EnterpriseCacheSessionDAO 
	sessionDAO.activeSessionsCacheName=shiro-activeSessionCache 
	sessionManager.sessionDAO=$sessionDAO 
	cacheManager = org.apache.shiro.cache.ehcache.EhCacheManager 
	cacheManager.cacheManagerConfigFile=classpath:ehcache.xml 
	securityManager.cacheManager = $cacheManager 
	
配置 ehcache.xml

	<cache name="shiro-activeSessionCache"        
		maxEntriesLocalHeap="10000"        
		overflowToDisk="false"        
		eternal="false"        
		diskPersistent="false"        
		timeToLiveSeconds="0"        
		timeToIdleSeconds="0"        
		statistics="true"
	/> 	
	
另外可以通过如下 ini 配置设置会话 ID 生成器：
 
	sessionIdGenerator=org.apache.shiro.session.mgt.eis.JavaUuidSessionIdGenerator 
	sessionDAO.sessionIdGenerator=$sessionIdGenerator 	
	
如果自定义实现 SessionDAO，继承 CachingSessionDAO 即可： 
	
	public class MySessionDAO extends CachingSessionDAO {     
		private JdbcTemplate jdbcTemplate = JdbcTemplateUtils.jdbcTemplate();      
		protected Serializable doCreate(Session session) {         
			Serializable sessionId = generateSessionId(session);        
			assignSessionId(session, sessionId);         
			String sql = "insert into sessions(id, session) values(?,?)";         jdbcTemplate.update(sql, sessionId, SerializableUtils.serialize(session));  
			return session.getId();     
		} 
		protected void doUpdate(Session session) {     
			if(session instanceof ValidatingSession && !((ValidatingSession)session).isValid()) {         
				return; //如果会话过期/停止 没必要再更新了    
			}         
			String sql = "update sessions set session=? where id=?";         
			jdbcTemplate.update(sql, SerializableUtils.serialize(session), session.getId());     
		}     
		protected void doDelete(Session session) {         
			String sql = "delete from sessions where id=?";         
			jdbcTemplate.update(sql, session.getId());     
		}     
		protected Session doReadSession(Serializable sessionId) {         
			String sql = "select session from sessions where id=?";         
			List<String> sessionStrList = jdbcTemplate.queryForList(sql, String.class, sessionId);         
			if(sessionStrList.size() == 0) 
				return null; 
			return SerializableUtils.unSerialize(sessionStrList.get(0));
		}
		
	}
	
Shiro 提供了会话验证调度器，用于定期的验证会话是否已过期，如果过期将停止会话；出 于性能考虑，一般情况下都是获取会话时来验证会话是否过期并停止会话的；但是如在 web 环境中，如果用户不主动退出是不知道会话是否过期的，因此需要定期的检测会话是否过 期，Shiro 提供了会话验证调度器 SessionValidationScheduler 来做这件事情。 

	sessionValidationScheduler=org.apache.shiro.session.mgt.ExecutorServiceSessionValidationScheduler 
	sessionValidationScheduler.interval = 3600000 
	sessionValidationScheduler.sessionManager=$sessionManager 
	sessionManager.globalSessionTimeout=1800000 
	sessionManager.sessionValidationSchedulerEnabled=true 
	sessionManager.sessionValidationScheduler=$sessionValidationScheduler 

Shiro 也提供了使用 Quartz 会话验证调度器

	sessionValidationScheduler=org.apache.shiro.session.mgt.quartz.QuartzSessionValidationScheduler 

## sessionFactory 
sessionFactory 是创建会话的工厂，根据相应的 Subject 上下文信息来创建会话；默认提供了 SimpleSessionFactory 用来创建 SimpleSession 会话。 

可以自定义session

# 缓存管理
Shiro 提供了类似于 Spring 的 Cache 抽象，即 Shiro 本身不实现 Cache，但是对 Cache 进行 了又抽象，方便更换不同的底层 Cache 实现。对于 Cache 的一些概念可以参考《Spring Cache 抽象详解》：http://jinnianshilongnian.iteye.com/blog/2001040。 

Shiro 提供的 Cache 接口：

	public interface Cache<K, V> {     
		//根据 Key 获取缓存中的值 
		public V get(K key) throws CacheException;     
		//往缓存中放入 key-value，返回缓存中之前的值     
		public V put(K key, V value) throws CacheException;      
		//移除缓存中 key 对应的值，返回该值 
		public V remove(K key) throws CacheException;     
		//清空整个缓存     
		public void clear() throws CacheException;     
		//返回缓存大小     
		public int size();     
		//获取缓存中所有的 key     
		public Set<K> keys();     
		//获取缓存中所有的 value     
		public Collection<V> values(); 
	} 
                   
Shiro 提供的 CacheManager 接口：       

	public interface CacheManager {     
		//根据缓存名字获取一个 Cache     
		public <K, V> Cache<K, V> getCache(String name) throws CacheException; 
	}
	
Shiro 还提供了 CacheManagerAware 用于注入 CacheManager： 
 
	public interface CacheManagerAware {     
		//注入 CacheManager     
		void setCacheManager(CacheManager cacheManager); 
	}
 
Shiro 内部相应的组件（DefaultSecurityManager）会自动检测相应的对象（如 Realm）是否 实现了 CacheManagerAware 并自动注入相应的 CacheManager。

## Realm 缓存
Shiro 提供了 CachingRealm，其实现了 CacheManagerAware 接口，提供了缓存的一些基础 实现；另外 AuthenticatingRealm 及 AuthorizingRealm 分别提供了对 AuthenticationInfo 和 AuthorizationInfo 信息的缓存。

但缓存也可以使用spring cache或者自己通过AOP实现的cache

## Session缓存
如 securityManager 实现了 SessionsSecurityManager，其会自动判断 SessionManager 是否实 现了 CacheManagerAware 接口，如果实现了会把 CacheManager 设置给它。然后 sessionManager 会判断相应的 sessionDAO（如继承自 CachingSessionDAO）是否实现了 CacheManagerAware，如果实现了会把 CacheManager 设置给它

# spring集成
Shiro 的组件都是 JavaBean/POJO 式的组件，所以非常容易使用 Spring 进行组件管理

	<!-- 缓存管理器 使用 Ehcache 实现 --> 
	<bean id="cacheManager" class="org.apache.shiro.cache.ehcache.EhCacheManager">     
		<property name="cacheManagerConfigFile" value="classpath:ehcache.xml"/> 
	</bean>  

	<!-- 凭证匹配器 --> 
	<bean id="credentialsMatcher" class=" com.github.zhangkaitao.shiro.chapter12.credentials.RetryLimitHashedCredentialsMatcher">     
		<constructor-arg ref="cacheManager"/>     
		<property name="hashAlgorithmName" value="md5"/>     
		<property name="hashIterations" value="2"/>     
		<property name="storedCredentialsHexEncoded" value="true"/> 
	</bean>  

	<!-- Realm 实现 --> 
	<bean id="userRealm" class="com.github.zhangkaitao.shiro.chapter12.realm.UserRealm">     
		<property name="userService" ref="userService"/>     
		<property name="credentialsMatcher" ref="credentialsMatcher"/>     
		<property name="cachingEnabled" value="true"/>     
		<property name="authenticationCachingEnabled" value="true"/>
		<property name="authenticationCacheName" value="authenticationCache"/>     
		<property name="authorizationCachingEnabled" value="true"/>     
		<property name="authorizationCacheName" value="authorizationCache"/> 
	</bean> 
	
	<!-- 会话 ID 生成器 --> 
	<bean id="sessionIdGenerator"  class="org.apache.shiro.session.mgt.eis.JavaUuidSessionIdGenerator"/> 
	
	<!-- 会话 DAO --> 
	<bean id="sessionDAO"  class="org.apache.shiro.session.mgt.eis.EnterpriseCacheSessionDAO">     
		<property name="activeSessionsCacheName" value="shiro-activeSessionCache"/>     
		<property name="sessionIdGenerator" ref="sessionIdGenerator"/> 
	</bean> 
	
	<!-- 会话验证调度器 --> 
	<bean id="sessionValidationScheduler"  class="org.apache.shiro.session.mgt.quartz.QuartzSessionValidationScheduler">     
		<property name="sessionValidationInterval" value="1800000"/>     
		<property name="sessionManager" ref="sessionManager"/> 
	</bean> 
	
	<!-- 会话管理器 --> 
	<bean id="sessionManager" class="org.apache.shiro.session.mgt.DefaultSessionManager">     
		<property name="globalSessionTimeout" value="1800000"/>     
		<property name="deleteInvalidSessions" value="true"/>     
		<property name="sessionValidationSchedulerEnabled" value="true"/>    
		<property name="sessionValidationScheduler" ref="sessionValidationScheduler"/>     
		<property name="sessionDAO" ref="sessionDAO"/> 
	</bean> 
	
	<!-- 安全管理器 --> 
	<bean id="securityManager" class="org.apache.shiro.mgt.DefaultSecurityManager">     
		<property name="realms">         
			<list>
				<ref bean="userRealm"/>
			</list>     
		</property>     
		<property name="sessionManager" ref="sessionManager"/>     
		<property name="cacheManager" ref="cacheManager"/> 
	</bean> 
	
	<!-- 相当于调用 SecurityUtils.setSecurityManager(securityManager) --> <bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
		<property name="staticMethod"  value="org.apache.shiro.SecurityUtils.setSecurityManager"/>     
		<property name="arguments" ref="securityManager"/> 
	</bean> 
	
	<!-- Shiro 生命周期处理器--> 
	<bean id="lifecycleBeanPostProcessor"  class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/> 
	
Web集成

	<!-- 会话 Cookie 模板 --> 
	<bean id="sessionIdCookie" class="org.apache.shiro.web.servlet.SimpleCookie">     
		<constructor-arg value="sid"/>     
		<property name="httpOnly" value="true"/>     
		<property name="maxAge" value="180000"/> 
	</bean> 
	
	<!-- 会话管理器 --> 
	<bean id="sessionManager"  class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">     
		<property name="globalSessionTimeout" value="1800000"/>     
		<property name="deleteInvalidSessions" value="true"/>     
		<property name="sessionValidationSchedulerEnabled" value="true"/>     
		<property name="sessionValidationScheduler" ref="sessionValidationScheduler"/>     
		<property name="sessionDAO" ref="sessionDAO"/>     
		<property name="sessionIdCookieEnabled" value="true"/>     
		<property name="sessionIdCookie" ref="sessionIdCookie"/> 
	</bean> 
	
	<!-- 安全管理器 --> 
	<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager"> 
		<property name="realm" ref="userRealm"/>     
		<property name="sessionManager" ref="sessionManager"/> 
		<property name="cacheManager" ref="cacheManager"/> 
	</bean> 
	
1. sessionIdCookie 是用于生产 Session ID Cookie 的模板； 
2. 会话管理器使用用于 web 环境的 DefaultWebSessionManager； 
3. 安全管理器使用用于 web 环境的 DefaultWebSecurityManager。 
	
身份验证过滤器

	<!-- Shiro 的 Web 过滤器 --> 
	<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">     
		<property name="securityManager" ref="securityManager"/>     
		<property name="loginUrl" value="/login.jsp"/>     
		<property name="unauthorizedUrl" value="/unauthorized.jsp"/>     
		<property name="filterChainDefinitions">         
			<value>             
				/index.jsp = anon             
				/unauthorized.jsp = anon             
				/login.jsp = authc             
				/logout = logout             
				/** = user         
			</value>     
		</property> 
	</bean>
	
shiroFilter：此处使用 ShiroFilterFactoryBean 来创建 ShiroFilter 过滤器；filterChainDefinitions 用于声明 url 和 filter 的关系
	
接着需要在 web.xml 中进行如下配置： 

	<filter>     
		<filter-name>shiroFilter</filter-name>     	
		<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>     
		<init-param>         
			<param-name>targetFilterLifecycle</param-name>         
			<param-value>true</param-value>     
		</init-param> 
	</filter> 
	
	<filter-mapping>     
		<filter-name>shiroFilter</filter-name>     
		<url-pattern>/*</url-pattern> 
	</filter-mapping> 

DelegatingFilterProxy 会自动到 Spring 容器中查找名字为 shiroFilter 的 bean 并把 filter 请求 交给它处理。 

## Shiro 权限注解
Shiro 提供了相应的注解用于权限控制，如果使用这些注解就需要使用 AOP 的功能来进行 判断，如 Spring AOP；Shiro 提供了 Spring AOP 集成用于权限注解的解析和验证。 

在 spring-mvc.xml 配置文件添加 Shiro Spring AOP 权限注解的支持

	<aop:config proxy-target-class="true"></aop:config> 
	<bean class=" org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">     
		<property name="securityManager" ref="securityManager"/> 
	</bean> 

如上配置用于开启 Shiro Spring AOP 权限注解的支持；<aop:config proxy-target-class="true">
	
接着就可以在相应的控制器（AnnotationController）中使用如下方式进行注解：

	@RequiresRoles("admin") 
	@RequestMapping("/hello2") 
	public String hello2() {     
		return "success"; 
	} 
	
当验证失败，其会抛出UnauthorizedException异常，此时可以使用Spring的ExceptionHandler （DefaultExceptionHandler）来进行拦截处理： 
	
### 权限注解
 
- 表示当前 Subject 已经通过 @RequiresAuthentication 
- 表示当前 Subject 已经身份验证或者通过记住我登录的 @RequiresUser   
- 表示当前 Subject 没有身份验证或通过记住我登录过，即是游客身份。@RequiresGuest   
- 表示当前 Subject 需要角色 admin 和 user。 @RequiresRoles(value={“admin”, “user”}, logical= Logical.AND)   
- 表示当前 Subject 需要权限 user:a 或 user:b。 @RequiresPermissions (value={“user:a”, “user:b”}, logical= Logical.OR) 
  




	
	
	
	
	
	
	




