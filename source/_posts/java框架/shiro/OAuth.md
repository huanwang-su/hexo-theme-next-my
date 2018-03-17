---
title: OAuth
date: 2018/3/16 08:28:25
category:
- java框架
- shiro
tag:
- shiro
- OAuth
comments: true  
---
# OAuth2 集成 
目前很多开放平台如新浪微博开放平台都在使用提供开放 API 接口供开发者使用，随之带 来了第三方应用要到开放平台进行授权的问题，OAuth 就是干这个的，OAuth2 是 OAuth 协议的下一个版本，相比 OAuth1，OAuth2 整个授权流程更简单安全了，但不兼容 OAuth1， 具 体 可以到 OAuth2 官 网 http://oauth.net/2/ 查看，OAuth2 协 议 规 范 可以参考 http://tools.ietf.org/html/rfc6749。目前有好多参考实现供选择，可以到其官网查看下载。 

### OAuth 角色 
- 资源拥有者（resource owner）：能授权访问受保护资源的一个实体，可以是一个人，那我 们称之为最终用户；如新浪微博用户 zhangsan； 
- 资源服务器（resource server）：存储受保护资源，客户端通过 access token 请求资源，资源服务器响应受保护资源给客户端；存储着用户 zhangsan 的微博等信息。
- 授权服务器（authorization server）：成功验证资源拥有者并获取授权之后，授权服务器 颁发授权令牌（Access Token）给客户端。 
- 客户端（client）：如新浪微博客户端weico、微格等第三方应用，也可以是它自己的官方应用；其本身不存储资源，而是资源拥有者授权通过后，使用它的授权（授权令牌）访问受保护资源，然后客户端把相应的数据展示出来/提交到服务器。“客户端”术语不代表任 何特定实现（如应用运行在一台服务器、桌面、手机或其他设备）。 

![](http://dl2.iteye.com/upload/attachment/0095/4801/3e97c6f9-41a7-398e-9cdb-b60afda794e9.png)

1. 客户端从资源拥有者那请求授权。授权请求可以直接发给资源拥有者，或间接的通过授权服务器这种中介，后者更可取。 
2. 客户端收到一个授权许可，代表资源服务器提供的授权。 
3. 客户端使用它自己的私有证书及授权许可到授权服务器验证。 
4. 如果验证成功，则下发一个访问令牌。 
5. 客户端使用访问令牌向资源服务器请求受保护资源。 
6. 资源服务器会验证访问令牌的有效性，如果成功则下发受保护资源

### 授权码类型的开放授权
![](http://www.sinaimg.cn/blog/developer/wiki/oAuth2_02.gif)

1. Client初始化协议的执行流程。首先通过HTTP 302来重定向RO用户代理到AS。Client在redirect_uri中应包含如下参数：client_id, scope (描述被访问的资源), redirect_uri (即Client的URI), state (用于抵制CSRF攻击). 此外，请求中还可以包含access_type和approval_prompt参数。当approval_prompt=force时，AS将提供交互页面，要求RO必须显式地批准（或拒绝）Client的此次请求。如果没有approval_prompt参数，则默认为RO批准此次请求。当access_type=offline时，AS将在颁发access_token时，同时还会颁发一个refresh_token。因为access_token的有效期较短（如3600秒），为了优化协议执行流程，offline方式将允许Client直接持refresh_token来换取一个新的access_token。
2. AS认证RO身份，并提供页面供RO决定是否批准或拒绝Client的此次请求（当approval_prompt=force时）。
3. 若请求被批准，AS使用步骤(1)中Client提供的redirect_uri重定向RO用户代理到Client。redirect_uri须包含authorization_code，以及步骤1中Client提供的state。若请求被拒绝，AS将通过redirect_uri返回相应的错误信息。
4. Client拿authorization_code去访问AS以交换所需的access_token。Client请求信息中应包含用于认证Client身份所需的认证数据，以及上一步请求authorization_code时所用的redirect_uri。
5. AS在收到authorization_code时需要验证Client的身份，并验证收到的redirect_uri与第3步请求authorization_code时所使用的redirect_uri相匹配。如果验证通过，AS将返回access_token，以及refresh_token（若access_type=offline）。

## 服务器端
### POM 依赖
此处我们使用 apache oltu oauth2 服务端实现，需要引入 authzserver（授权服务器依赖）和 resourceserver（资源服务器依赖）

	<dependency>     			
		<groupId>org.apache.oltu.oauth2</groupId>     		
		<artifactId>org.apache.oltu.oauth2.authzserver</artifactId>    
		<version>0.31</version> 
	</dependency> 
	<dependency>     
		<groupId>org.apache.oltu.oauth2</groupId>     	<artifactId>org.apache.oltu.oauth2.resourceserver</artifactId> 
		<version>0.31</version> 
	</dependency>

### 实体类
### 表和Sql
### DAO
### Service
	public interface UserService {  
		public User createUser(User user);// 创建用户  
		public User updateUser(User user);// 更新用户  
		public void deleteUser(Long userId);// 删除用户  
		public void changePassword(Long userId, String newPassword); //修改密码  
		User findOne(Long userId);// 根据id查找用户  
		List<User> findAll();// 得到所有用户  
		public User findByUsername(String username);// 根据用户名查找用户  
	}  

	public interface ClientService {  
		public Client createClient(Client client);// 创建客户端  
		public Client updateClient(Client client);// 更新客户端  
		public void deleteClient(Long clientId);// 删除客户端  
		Client findOne(Long clientId);// 根据id查找客户端  
		List<Client> findAll();// 查找所有  
		Client findByClientId(String clientId);// 根据客户端id查找客户端  
		Client findByClientSecret(String clientSecret);//根据客户端安全KEY查找客户端  
	}  

	此处通过OAuthService实现进行auth code和access token的维护。

	public interface OAuthService {  
	   public void addAuthCode(String authCode, String username);// 添加 auth code  
	   public void addAccessToken(String accessToken, String username); // 添加 access token  
	   boolean checkAuthCode(String authCode); // 验证auth code是否有效  
	   boolean checkAccessToken(String accessToken); // 验证access token是否有效  
	   String getUsernameByAuthCode(String authCode);// 根据auth code获取用户名  
	   String getUsernameByAccessToken(String accessToken);// 根据access token获取用户名  
	   long getExpireIn();//auth code / access token 过期时间  
	   public boolean checkClientId(String clientId);// 检查客户端id是否存在  
	   public boolean checkClientSecret(String clientSecret);// 坚持客户端安全KEY是否存在  
	}   

### 控制器 
#### AuthorizeController 

	@Controller
	public class AuthorizeController {
	  @Autowired
	  private OAuthService oAuthService;
	  @Autowired
	  private ClientService clientService;
	  @RequestMapping("/authorize")
	  public Object authorize(Model model,  HttpServletRequest request)
			throws URISyntaxException, OAuthSystemException {
		try {
		  //构建OAuth 授权请求
		  OAuthAuthzRequest oauthRequest = new OAuthAuthzRequest(request);
		  //检查传入的客户端id是否正确
		  if (!oAuthService.checkClientId(oauthRequest.getClientId())) {
			OAuthResponse response = OAuthASResponse
				 .errorResponse(HttpServletResponse.SC_BAD_REQUEST)
				 .setError(OAuthError.TokenResponse.INVALID_CLIENT)
				 .setErrorDescription(Constants.INVALID_CLIENT_DESCRIPTION)
				 .buildJSONMessage();
			return new ResponseEntity(
			   response.getBody(), HttpStatus.valueOf(response.getResponseStatus()));
		  }

		  Subject subject = SecurityUtils.getSubject();
		  //如果用户没有登录，跳转到登陆页面
		  if(!subject.isAuthenticated()) {
			if(!login(subject, request)) {//登录失败时跳转到登陆页面
			  model.addAttribute("client",    
				  clientService.findByClientId(oauthRequest.getClientId()));
			  return "oauth2login";
			}
		  }

		  String username = (String)subject.getPrincipal();
		  //生成授权码
		  String authorizationCode = null;
		  //responseType目前仅支持CODE，另外还有TOKEN
		  String responseType = oauthRequest.getParam(OAuth.OAUTH_RESPONSE_TYPE);
		  if (responseType.equals(ResponseType.CODE.toString())) {
			OAuthIssuerImpl oauthIssuerImpl = new OAuthIssuerImpl(new MD5Generator());
			authorizationCode = oauthIssuerImpl.authorizationCode();
			oAuthService.addAuthCode(authorizationCode, username);
		  }
		  //进行OAuth响应构建
		  OAuthASResponse.OAuthAuthorizationResponseBuilder builder =
			OAuthASResponse.authorizationResponse(request, 
											   HttpServletResponse.SC_FOUND);
		  //设置授权码
		  builder.setCode(authorizationCode);
		  //得到到客户端重定向地址
		  String redirectURI = oauthRequest.getParam(OAuth.OAUTH_REDIRECT_URI);

		  //构建响应
		  final OAuthResponse response = builder.location(redirectURI).buildQueryMessage();
		  //根据OAuthResponse返回ResponseEntity响应
		  HttpHeaders headers = new HttpHeaders();
		  headers.setLocation(new URI(response.getLocationUri()));
		  return new ResponseEntity(headers, HttpStatus.valueOf(response.getResponseStatus()));
		} catch (OAuthProblemException e) {
		  //出错处理
		  String redirectUri = e.getRedirectUri();
		  if (OAuthUtils.isEmpty(redirectUri)) {
			//告诉客户端没有传入redirectUri直接报错
			return new ResponseEntity(
			  "OAuth callback url needs to be provided by client!!!", HttpStatus.NOT_FOUND);
		  }
		  //返回错误消息（如?error=）
		  final OAuthResponse response =
				  OAuthASResponse.errorResponse(HttpServletResponse.SC_FOUND)
						  .error(e).location(redirectUri).buildQueryMessage();
		  HttpHeaders headers = new HttpHeaders();
		  headers.setLocation(new URI(response.getLocationUri()));
		  return new ResponseEntity(headers, HttpStatus.valueOf(response.getResponseStatus()));
		}
	  }

	  private boolean login(Subject subject, HttpServletRequest request) {
		if("get".equalsIgnoreCase(request.getMethod())) {
		  return false;
		}
		String username = request.getParameter("username");
		String password = request.getParameter("password");

		if(StringUtils.isEmpty(username) || StringUtils.isEmpty(password)) {
		  return false;
		}

		UsernamePasswordToken token = new UsernamePasswordToken(username, password);
		try {
		  subject.login(token);
		  return true;
		} catch (Exception e) {
		  request.setAttribute("error", "登录失败:" + e.getClass().getName());
		  return false;
		}
	  }
	};
	
1. 首先通过如http://localhost:8080/chapter17-server/authorize
?client_id=c1ebe466-1cdc-4bd3-ab69-77c3561b9dee&response_type=code&redirect_uri=http://localhost:9080/chapter17-client/oauth2-login访问授权页面；
2. 该控制器首先检查clientId是否正确；如果错误将返回相应的错误信息；
3. 然后判断用户是否登录了，如果没有登录首先到登录页面登录；
4. 登录成功后生成相应的auth code即授权码，然后重定向到客户端地址，如http://localhost:9080/chapter17-client/oauth2-login?code=52b1832f5dff68122f4f00ae995da0ed；在重定向到的地址中会带上code参数（授权码），接着客户端可以根据授权码去换取access token。	
	
#### AccessTokenController  
	
	@RestController
	public class AccessTokenController {
	  @Autowired
	  private OAuthService oAuthService;
	  @Autowired
	  private UserService userService;
	  @RequestMapping("/accessToken")
	  public HttpEntity token(HttpServletRequest request)
			  throws URISyntaxException, OAuthSystemException {
		try {
		  //构建OAuth请求
		  OAuthTokenRequest oauthRequest = new OAuthTokenRequest(request);

		  //检查提交的客户端id是否正确
		  if (!oAuthService.checkClientId(oauthRequest.getClientId())) {
			OAuthResponse response = OAuthASResponse
					.errorResponse(HttpServletResponse.SC_BAD_REQUEST)
					.setError(OAuthError.TokenResponse.INVALID_CLIENT)
					.setErrorDescription(Constants.INVALID_CLIENT_DESCRIPTION)
					.buildJSONMessage();
		   return new ResponseEntity(
			 response.getBody(), HttpStatus.valueOf(response.getResponseStatus()));
		  }

		// 检查客户端安全KEY是否正确
		  if (!oAuthService.checkClientSecret(oauthRequest.getClientSecret())) {
			OAuthResponse response = OAuthASResponse
				  .errorResponse(HttpServletResponse.SC_UNAUTHORIZED)
				  .setError(OAuthError.TokenResponse.UNAUTHORIZED_CLIENT)
				  .setErrorDescription(Constants.INVALID_CLIENT_DESCRIPTION)
				  .buildJSONMessage();
		  return new ResponseEntity(
			  response.getBody(), HttpStatus.valueOf(response.getResponseStatus()));
		  }
	  
		  String authCode = oauthRequest.getParam(OAuth.OAUTH_CODE);
		  // 检查验证类型，此处只检查AUTHORIZATION_CODE类型，其他的还有PASSWORD或REFRESH_TOKEN
		  if (oauthRequest.getParam(OAuth.OAUTH_GRANT_TYPE).equals(
			 GrantType.AUTHORIZATION_CODE.toString())) {
			 if (!oAuthService.checkAuthCode(authCode)) {
				OAuthResponse response = OAuthASResponse
					.errorResponse(HttpServletResponse.SC_BAD_REQUEST)
					.setError(OAuthError.TokenResponse.INVALID_GRANT)
					.setErrorDescription("错误的授权码")
				  .buildJSONMessage();
			   return new ResponseEntity(
				 response.getBody(), HttpStatus.valueOf(response.getResponseStatus()));
			 }
		  }

		  //生成Access Token
		  OAuthIssuer oauthIssuerImpl = new OAuthIssuerImpl(new MD5Generator());
		  final String accessToken = oauthIssuerImpl.accessToken();
		  oAuthService.addAccessToken(accessToken,
			  oAuthService.getUsernameByAuthCode(authCode));

		  //生成OAuth响应
		  OAuthResponse response = OAuthASResponse
				  .tokenResponse(HttpServletResponse.SC_OK)
				  .setAccessToken(accessToken)
				  .setExpiresIn(String.valueOf(oAuthService.getExpireIn()))
				  .buildJSONMessage();

		  //根据OAuthResponse生成ResponseEntity
		  return new ResponseEntity(
			  response.getBody(), HttpStatus.valueOf(response.getResponseStatus()));
		} catch (OAuthProblemException e) {
		  //构建错误响应
		  OAuthResponse res = OAuthASResponse
				  .errorResponse(HttpServletResponse.SC_BAD_REQUEST).error(e)
				  .buildJSONMessage();
		 return new ResponseEntity(res.getBody(), HttpStatus.valueOf(res.getResponseStatus()));
	   }
	 }
	}
	
1. 首先通过如http://localhost:8080/chapter17-server/accessToken，POST提交如下数据：client_id= c1ebe466-1cdc-4bd3-ab69-77c3561b9dee& client_secret= d8346ea2-6017-43ed-ad68-19c0f971738b&grant_type=authorization_code&code=828beda907066d058584f37bcfd597b6&redirect_uri=http://localhost:9080/chapter17-client/oauth2-login访问；
2. 该控制器会验证client_id、client_secret、auth code的正确性，如果错误会返回相应的错误；
3. 如果验证通过会生成并返回相应的访问令牌access token。

#### 资源控制器
@RestController
	public class UserInfoController {
	  @Autowired
	  private OAuthService oAuthService;

	  @RequestMapping("/userInfo")
	  public HttpEntity userInfo(HttpServletRequest request) throws OAuthSystemException {
		try {
		  //构建OAuth资源请求
		  OAuthAccessResourceRequest oauthRequest = 
				new OAuthAccessResourceRequest(request, ParameterStyle.QUERY);
		  //获取Access Token
		  String accessToken = oauthRequest.getAccessToken();

		  //验证Access Token
		  if (!oAuthService.checkAccessToken(accessToken)) {
			// 如果不存在/过期了，返回未验证错误，需重新验证
		  OAuthResponse oauthResponse = OAuthRSResponse
				  .errorResponse(HttpServletResponse.SC_UNAUTHORIZED)
				  .setRealm(Constants.RESOURCE_SERVER_NAME)
				  .setError(OAuthError.ResourceResponse.INVALID_TOKEN)
				  .buildHeaderMessage();

			HttpHeaders headers = new HttpHeaders();
			headers.add(OAuth.HeaderType.WWW_AUTHENTICATE, 
			  oauthResponse.getHeader(OAuth.HeaderType.WWW_AUTHENTICATE));
		  return new ResponseEntity(headers, HttpStatus.UNAUTHORIZED);
		  }
		  //返回用户名
		  String username = oAuthService.getUsernameByAccessToken(accessToken);
		  return new ResponseEntity(username, HttpStatus.OK);
		} catch (OAuthProblemException e) {
		  //检查是否设置了错误码
		  String errorCode = e.getError();
		  if (OAuthUtils.isEmpty(errorCode)) {
			OAuthResponse oauthResponse = OAuthRSResponse
				   .errorResponse(HttpServletResponse.SC_UNAUTHORIZED)
				   .setRealm(Constants.RESOURCE_SERVER_NAME)
				   .buildHeaderMessage();

			HttpHeaders headers = new HttpHeaders();
			headers.add(OAuth.HeaderType.WWW_AUTHENTICATE, 
			  oauthResponse.getHeader(OAuth.HeaderType.WWW_AUTHENTICATE));
			return new ResponseEntity(headers, HttpStatus.UNAUTHORIZED);
		  }

		  OAuthResponse oauthResponse = OAuthRSResponse
				   .errorResponse(HttpServletResponse.SC_UNAUTHORIZED)
				   .setRealm(Constants.RESOURCE_SERVER_NAME)
				   .setError(e.getError())
				   .setErrorDescription(e.getDescription())
				   .setErrorUri(e.getUri())
				   .buildHeaderMessage();

		  HttpHeaders headers = new HttpHeaders();
		  headers.add(OAuth.HeaderType.WWW_AUTHENTICATE, 、
			oauthResponse.getHeader(OAuth.HeaderType.WWW_AUTHENTICATE));
		  return new ResponseEntity(HttpStatus.BAD_REQUEST);
		}
	  }
	};
	
1. 首先通过如http://localhost:8080/chapter17-server/userInfo? access_token=828beda907066d058584f37bcfd597b6进行访问；
2. 该控制器会验证access token的有效性；如果无效了将返回相应的错误，客户端再重新进行授权；
3. 如果有效，则返回当前登录用户的用户名。

### Spring配置文件
只列举spring-config-shiro.xml中的shiroFilter的filterChainDefinitions属性：  

	<property name="filterChainDefinitions">
		<value>
		  / = anon
		  /login = authc
		  /logout = logout

		  /authorize=anon
		  /accessToken=anon
		  /userInfo=anon

		  /** = user
		</value>
	</property>

对于oauth2的几个地址/authorize、/accessToken、/userInfo都是匿名可访问的

### 服务器维护
客户端管理就是进行客户端的注册，如新浪微博的第三方应用就需要到新浪微博开发平台进行注册；用户管理就是进行如新浪微博用户的管理。

对于授权服务和资源服务的实现可以参考新浪微博开发平台的实现：
http://open.weibo.com/wiki/授权机制说明 
http://open.weibo.com/wiki/微博API 
 
## 客户端
客户端流程：如果需要登录首先跳到oauth2服务端进行登录授权，成功后服务端返回auth code，然后客户端使用auth code去服务器端换取access token，最好根据access token获取用户信息进行客户端的登录绑定。这个可以参照如很多网站的新浪微博登录功能，或其他的第三方帐号登录功能。
### POM依賴 
<dependency>  
  <groupId>org.apache.oltu.oauth2</groupId>  
  <artifactId>org.apache.oltu.oauth2.client</artifactId>  
  <version>0.31</version>  
</dependency>

### OAuth2Token
类似于UsernamePasswordToken和CasToken；用于存储oauth2服务端返回的auth code。  

	public class OAuth2Token implements AuthenticationToken {  
		private String authCode;  
		private String principal;  
		public OAuth2Token(String authCode) {  
			this.authCode = authCode;  
		}  
		//省略getter/setter  
	}   

### OAuth2AuthenticationFilter
用于oauth2客户端的身份验证控制；如果当前用户还没有身份验证，首先会判断url中是否有code（服务端返回的auth code），如果没有则重定向到服务端进行登录并授权，然后返回auth code；接着OAuth2AuthenticationFilter会用auth code创建OAuth2Token，然后提交给Subject.login进行登录；接着OAuth2Realm会根据OAuth2Token进行相应的登录逻辑。  

	public class OAuth2AuthenticationFilter extends AuthenticatingFilter {
		//oauth2 authc code参数名
		private String authcCodeParam = "code";
		//客户端id
		private String clientId;
		//服务器端登录成功/失败后重定向到的客户端地址
		private String redirectUrl;
		//oauth2服务器响应类型
		private String responseType = "code";
		private String failureUrl;
		//省略setter
		protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception {
			HttpServletRequest httpRequest = (HttpServletRequest) request;
			String code = httpRequest.getParameter(authcCodeParam);
			return new OAuth2Token(code);
		}
		protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
			return false;
		}
		protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
			String error = request.getParameter("error");
			String errorDescription = request.getParameter("error_description");
			if(!StringUtils.isEmpty(error)) {//如果服务端返回了错误
				WebUtils.issueRedirect(request, response, failureUrl + "?error=" + error + "error_description=" + errorDescription);
				return false;
			}
			Subject subject = getSubject(request, response);
			if(!subject.isAuthenticated()) {
				if(StringUtils.isEmpty(request.getParameter(authcCodeParam))) {
					//如果用户没有身份验证，且没有auth code，则重定向到服务端授权
					saveRequestAndRedirectToLogin(request, response);
					return false;
				}
			}
			//执行父类里的登录逻辑，调用Subject.login登录
			return executeLogin(request, response);
		}

		//登录成功后的回调方法 重定向到成功页面
		protected boolean onLoginSuccess(AuthenticationToken token, Subject subject, ServletRequest request,  ServletResponse response) throws Exception {
			issueSuccessRedirect(request, response);
			return false;
		}

		//登录失败后的回调 
		protected boolean onLoginFailure(AuthenticationToken token, AuthenticationException ae, ServletRequest request,
										 ServletResponse response) {
			Subject subject = getSubject(request, response);
			if (subject.isAuthenticated() || subject.isRemembered()) {
				try { //如果身份验证成功了 则也重定向到成功页面
					issueSuccessRedirect(request, response);
				} catch (Exception e) {
					e.printStackTrace();
				}
			} else {
				try { //登录失败时重定向到失败页面
					WebUtils.issueRedirect(request, response, failureUrl);
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
			return false;
		}
	}

该拦截器的作用：
1. 首先判断有没有服务端返回的error参数，如果有则直接重定向到失败页面；
2. 接着如果用户还没有身份验证，判断是否有auth code参数（即是不是服务端授权之后返回的），如果没有则重定向到服务端进行授权；
3. 否则调用executeLogin进行登录，通过auth code创建OAuth2Token提交给Subject进行登录；
4. 登录成功将回调onLoginSuccess方法重定向到成功页面；
5. 登录失败则回调onLoginFailure重定向到失败页面。

### OAuth2Realm 
	public class OAuth2Realm extends AuthorizingRealm {
		private String clientId;
		private String clientSecret;
		private String accessTokenUrl;
		private String userInfoUrl;
		private String redirectUrl;
		//省略setter
		public boolean supports(AuthenticationToken token) {
			return token instanceof OAuth2Token; //表示此Realm只支持OAuth2Token类型
		}
		protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
			SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
			return authorizationInfo;
		}
		protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
			OAuth2Token oAuth2Token = (OAuth2Token) token;
			String code = oAuth2Token.getAuthCode(); //获取 auth code
			String username = extractUsername(code); // 提取用户名
			SimpleAuthenticationInfo authenticationInfo =
					new SimpleAuthenticationInfo(username, code, getName());
			return authenticationInfo;
		}
		private String extractUsername(String code) {
			try {
				OAuthClient oAuthClient = new OAuthClient(new URLConnectionClient());
				OAuthClientRequest accessTokenRequest = OAuthClientRequest
						.tokenLocation(accessTokenUrl)
						.setGrantType(GrantType.AUTHORIZATION_CODE)
						.setClientId(clientId).setClientSecret(clientSecret)
						.setCode(code).setRedirectURI(redirectUrl)
						.buildQueryMessage();
				//获取access token
				OAuthAccessTokenResponse oAuthResponse = 
					oAuthClient.accessToken(accessTokenRequest, OAuth.HttpMethod.POST);
				String accessToken = oAuthResponse.getAccessToken();
				Long expiresIn = oAuthResponse.getExpiresIn();
				//获取user info
				OAuthClientRequest userInfoRequest = 
					new OAuthBearerClientRequest(userInfoUrl)
						.setAccessToken(accessToken).buildQueryMessage();
				OAuthResourceResponse resourceResponse = oAuthClient.resource(
					userInfoRequest, OAuth.HttpMethod.GET, OAuthResourceResponse.class);
				String username = resourceResponse.getBody();
				return username;
			} catch (Exception e) {
				throw new OAuth2AuthenticationException(e);
			}
		}
	}

此Realm首先只支持OAuth2Token类型的Token；然后通过传入的auth code去换取access token；再根据access token去获取用户信息（用户名），然后根据此信息创建AuthenticationInfo；如果需要AuthorizationInfo信息，可以根据此处获取的用户名再根据自己的业务规则去获取。

### Spring shiro配置（spring-config-shiro.xml）  
	<bean id="oAuth2Realm" 
		class="com.github.zhangkaitao.shiro.chapter18.oauth2.OAuth2Realm">
	  <property name="cachingEnabled" value="true"/>
	  <property name="authenticationCachingEnabled" value="true"/>
	  <property name="authenticationCacheName" value="authenticationCache"/>
	  <property name="authorizationCachingEnabled" value="true"/>
	  <property name="authorizationCacheName" value="authorizationCache"/>
	  <property name="clientId" value="c1ebe466-1cdc-4bd3-ab69-77c3561b9dee"/>
	  <property name="clientSecret" value="d8346ea2-6017-43ed-ad68-19c0f971738b"/>
	  <property name="accessTokenUrl" 
		 value="http://localhost:8080/chapter17-server/accessToken"/>
	  <property name="userInfoUrl" value="http://localhost:8080/chapter17-server/userInfo"/>
	  <property name="redirectUrl" value="http://localhost:9080/chapter17-client/oauth2-login"/>
	</bean>;

此OAuth2Realm需要配置在服务端申请的clientId和clientSecret；及用于根据auth code换取access token的accessTokenUrl地址；及用于根据access token换取用户信息（受保护资源）的userInfoUrl地址。

