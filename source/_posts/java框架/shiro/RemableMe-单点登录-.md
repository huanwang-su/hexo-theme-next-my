---
title: shiro RemableMe-单点登录
date: 2018/3/16 08:28:25
category:
- java框架
- shiro
tag:
- shiro
comments: true  
---

# RememberMe
Shiro 提供了记住我（RememberMe）的功能，比如访问如淘宝等一些网站时，关闭了浏览 器下次再打开时还是能记住你是谁，下次访问时无需再登录即可访问，基本流程如下

## RememberMe 配置 
### spring-shiro-web.xml 配置：         
	<!-- 会话 Cookie 模板 --> 
	<bean id="sessionIdCookie" class="org.apache.shiro.web.servlet.SimpleCookie">     
		<constructor-arg value="sid"/>     
		<property name="httpOnly" value="true"/>     
		<property name="maxAge" value="-1"/> 
	</bean> 
	<bean id="rememberMeCookie" class="org.apache.shiro.web.servlet.SimpleCookie">     
		<constructor-arg value="rememberMe"/>     
		<property name="httpOnly" value="true"/>     
		<property name="maxAge" value="2592000"/><!-- 30 天 --> 
	</bean>
	
- sessionIdCookie：maxAge=-1 表示浏览器关闭时失效此 Cookie； 
- rememberMeCookie：即记住我的 Cookie，保存时长 30 天；	

	<!-- rememberMe 管理器 --> 
	<bean id="rememberMeManager"  class="org.apache.shiro.web.mgt.CookieRememberMeManager">     
		<property name="cipherKey" value=" #{T(org.apache.shiro.codec.Base64).decode('4AvVhmFLUs0KTA3Kprsdag==')}"/>      
		<property name="cookie" ref="rememberMeCookie"/> 
	</bean> 
	
rememberMe 管理器，cipherKey 是加密 rememberMe Cookie 的密钥；默认 AES 算法
	
	<!-- 安全管理器 --> 
	<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		……     
		<property name="rememberMeManager" ref="rememberMeManager"/> 
	</bean> 
	
另外对于过滤器，一般这样使用： 
- 访问一般网页，如个人在主页之类的，我们使用 user 拦截器即可，user 拦截器只要用户登 录(isRemembered()==true or isAuthenticated()==true)过即可访问成功； 
- 访问特殊网页，如我的订单，提交订单页面，我们使用 authc 拦截器即可，authc 拦截器会 判断用户是否是通过 Subject.login（isAuthenticated()==true）登录的，如果是才放行，否则会跳转到登录页面叫你重新登录
	
因此 RememberMe 使用过程中，需要配合相应的拦截器来实现相应的功能，用错了拦截器 可能就不能满足你的需求了。 
	
# SSL 	
对于 SSL 的支持，Shiro 只是判断当前 url 是否需要 SSL 登录，如果需要自动重定向到 https 进行访问。 	

...略

# 单点登录	
Shiro 1.2 开始提供了 Jasig CAS 单点登录的支持，单点登录主要用于多系统集成，即在多个 系统中，用户只需要到一个中央服务器登录一次即可访问这些系统中的任何一个，无须多次登录。
	
大体流程如下：
1. 访问客户端需要登录的页面 http://localhost:9080/ client/，此时会跳到单点登录服务器 https://localhost:8443/ server/login?service=https://localhost:9443/ client/cas； 
2. 如果此时单点登录服务器也没有登录的话，会显示登录表单页面，输入用户名/密码进 行登录； 
3. 登录成功后服务器端会回调客户端传入的地址： https://localhost:9443/client/cas?ticket=ST-1-eh2cIo92F9syvoMs5DOg-cas01.example.org，且带 着一个 ticket； 
4. 客户端会把 ticket 提交给服务器来验证 ticket 是否有效；如果有效服务器端将返回用户 身份； 
5. 客户端可以再根据这个用户身份获取如当前系统用户/角色/权限信息。

# 并发登录人数控制
如果同时 有多人登录：要么不让后者登录；要么踢出前者登录（强制退出）。比如 spring security 就 直接提供了相应的功能；Shiro 的话没有提供默认实现，不过可以很容易的在 Shiro 中加入 这个功能。  

	<bean id="kickoutSessionControlFilter" 
	class="com.github.zhangkaitao.shiro.chapter18.web.shiro.filter.KickoutSessionControlFilter">
		<property name="cacheManager" ref="cacheManager"/>
		<property name="sessionManager" ref="sessionManager"/>

		<property name="kickoutAfter" value="false"/>
		<property name="maxSession" value="2"/>
		<property name="kickoutUrl" value="/login?kickout=1"/>
	</bean>
	
- cacheManager：使用cacheManager获取相应的cache来缓存用户登录的会话；用于保存用户—会话之间的关系的；
- sessionManager：用于根据会话ID，获取会话进行踢出操作的；
- kickoutAfter：是否踢出后来登录的，默认是false；即后者登录的用户踢出前者登录的用户；
- maxSession：同一个用户最大的会话数，默认1；比如2的意思是同一个用户允许最多同时两个人登录；
- kickoutUrl：被踢出后重定向到的地址；

### KickoutSessionControlFilter核心代码：
	public class KickoutSessionControlFilter  extends AccessControlFilter{  
		private String kickoutUrl; //踢出后到的地址  
		private boolean kickoutAfter; //踢出之前登录的/之后登录的用户 默认踢出之前登录的用户  
		private int maxSession; //同一个帐号最大会话数 默认1  
		private SessionManager sessionManager;  
		private Cache<String, Deque<Serializable>> cache;  
	  
		public void setKickoutUrl(String kickoutUrl) {  
			this.kickoutUrl = kickoutUrl;  
		}  
	  
		public void setKickoutAfter(boolean kickoutAfter) {  
			this.kickoutAfter = kickoutAfter;  
		}  
	  
		public void setMaxSession(int maxSession) {  
			this.maxSession = maxSession;  
		}  
	  
		public void setSessionManager(SessionManager sessionManager) {  
			this.sessionManager = sessionManager;  
		}  
	  
		public void setCacheManager(CacheManager cacheManager) {  
			this.cache = cacheManager.getCache("shiro-activeSessionCache");  
		}  
		 /** 
		  * 是否允许访问，返回true表示允许 
		  */  
		@Override  
		protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {  
			return false;  
		}  
		/** 
		 * 表示访问拒绝时是否自己处理，如果返回true表示自己不处理且继续拦截器链执行，返回false表示自己已经处理了（比如重定向到另一个页面）。 
		 */  
		@Override  
		protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {  
			Subject subject = getSubject(request, response);  
			if(!subject.isAuthenticated() && !subject.isRemembered()) {  
				//如果没有登录，直接进行之后的流程  
				return true;  
			}  
	  
			Session session = subject.getSession();  
			String username = (String) subject.getPrincipal();  
			Serializable sessionId = session.getId();  
	  
			// 初始化用户的队列放到缓存里  
			Deque<Serializable> deque = cache.get(username);  
			if(deque == null) {  
				deque = new LinkedList<Serializable>();  
				cache.put(username, deque);  
			}  
	  
			//如果队列里没有此sessionId，且用户没有被踢出；放入队列  
			if(!deque.contains(sessionId) && session.getAttribute("kickout") == null) {  
				deque.push(sessionId);  
			}  
	  
			//如果队列里的sessionId数超出最大会话数，开始踢人  
			while(deque.size() > maxSession) {  
				Serializable kickoutSessionId = null;  
				if(kickoutAfter) { //如果踢出后者  
					kickoutSessionId=deque.getFirst();  
					kickoutSessionId = deque.removeFirst();  
				} else { //否则踢出前者  
					kickoutSessionId = deque.removeLast();  
				}  
				try {  
					Session kickoutSession = sessionManager.getSession(new DefaultSessionKey(kickoutSessionId));  
					if(kickoutSession != null) {  
						//设置会话的kickout属性表示踢出了  
						kickoutSession.setAttribute("kickout", true);  
					}  
				} catch (Exception e) {//ignore exception  
					e.printStackTrace();  
				}  
			}  
	  
			//如果被踢出了，直接退出，重定向到踢出后的地址  
			if (session.getAttribute("kickout") != null) {  
				//会话被踢出了  
				try {  
					subject.logout();  
				} catch (Exception e) {   
				}  
				WebUtils.issueRedirect(request, response, kickoutUrl);  
				return false;  
			}  
			return true;  
		}  
	}  




