
# 客户端认证
客户端的入口函数为 KerberosAuthenticator.authenticate函数

## 连接HTTP服务端

```java
HttpURLConnection conn = token.openConnection(url, connConfigurator);
conn.setRequestMethod(AUTH_HTTP_METHOD);
conn.connect();
```

## SPNEGO认证
对于普通的HTTP的kerberos认证(SPNEGO)，需要现在客户端登录KDC服务。核心代码在函数doSpnegoSequence里面。

首先需要登录KDC服务端：

```java
subject = new Subject();
LoginContext login = new LoginContext("", subject,
    null, new KerberosConfiguration());
// 登录KDC服务
login.login();
```
下面主要是开始认证的时候的逻辑。核心处理逻辑流程图如下：

![pic](https://pan.zeekling.cn/zeekling/hadoop/common/Hadoop_auth_client_spnego.png)

主要代码逻辑如下。

```java
GSSManager gssManager = GSSManager.getInstance();
// 设置服务端的域名，由于是HTTP协议，所以当前要求principal的格式为：HTTP/HOST_NAME的方式。
String servicePrincipal = KerberosUtil.getServicePrincipal("HTTP",
    KerberosAuthenticator.this.url.getHost());
Oid oid = KerberosUtil.NT_GSS_KRB5_PRINCIPAL_OID;
GSSName serviceName = gssManager.createName(servicePrincipal,
                                            oid);
oid = KerberosUtil.GSS_KRB5_MECH_OID;
// 创建获取token的上下文信息。
gssContext = gssManager.createContext(serviceName, oid, null,
                                      GSSContext.DEFAULT_LIFETIME);
gssContext.requestCredDeleg(true);
gssContext.requestMutualAuth(true);

byte[] inToken = new byte[0];
byte[] outToken;
boolean established = false;

// Loop while the context is still not established
while (!established) {
  HttpURLConnection conn =
      token.openConnection(url, connConfigurator);
  // 获取客户端的token。对于第一次的场景，inToken为空。
  // 对于中间过程，需要将服务端给的token传进去校验。
  outToken = gssContext.initSecContext(inToken, 0, inToken.length);
  if (outToken != null) {
	// 将token发送给服务端
    sendToken(conn, outToken);
  }

  if (!gssContext.isEstablished()) {
    // 读取服务端发送的token。
	inToken = readToken(conn);
  } else {
	// 认证完成,认证结束
    established = true;
  }
}

```


## 认证完成/无须认证

如果HTTP服务端返回的是HTTP_OK，则认为服务端是不需要认证的，或者是已经认证完成了。在已经完成认证的场景下，
需要解析Token。

```java
AuthenticatedURL.extractToken(conn, token);
if (isTokenKerberos(token)) {
  return;
}
needFallback = true;
```


## 其他自定义

其他自定义认证的场景，可以通过指定Authenticator的方式实现，具体实现可以自定义，主要保证服务端和客户端一致即可。

```java
// 当前主要适用于对认证方式需要扩展的场景。
Authenticator auth = getFallBackAuthenticator();
auth.setConnectionConfigurator(connConfigurator);
auth.authenticate(url, token);
```


# 服务端认证


## 初始化

服务端初始化的入口函数类为AuthenticationFilter， 可以在web启动的时候将当前filter配置为AuthenticationFilter。用来实现服务端认证功能。

其中函数init为filter的初始化函数，主要作用是加载authHandler以及其他参数。在加载完authHandler之后会调用authHandler的init函数。
用来初始化authHandler。KerberosAuthenticationHandler就是一种authHandler。在初始化的时候会加载keytab和principal等信息。
并且初试化gssManager。

## 认证

认证的入口函数filter函数。对于大部分的authHandler，最终认证都是调用authHandler的authenticate函数。针对KerberosAuthenticationHandler来讲。

在需要认证的时候返回401，如下代码：
```java
response.setHeader(WWW_AUTHENTICATE, KerberosAuthenticator.NEGOTIATE);
response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
```

通过下面代码获取客户端Token。客户端的Token对应的keytab必须包含HTTP/*

```java
authorization = authorization.substring(KerberosAuthenticator.NEGOTIATE.length()).trim();
final Base64 base64 = new Base64(0);
final byte[] clientToken = base64.decode(authorization);
```

在与客户端的认证的过程中，可能会需要多次交换token（参考客户端流程），如果是在交互过程中，http请求的返回码是401。如果认证成功，状态码则是200.




