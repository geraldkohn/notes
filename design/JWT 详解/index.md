# JWT 详解

### 什么是 JSON Web 令牌？

JSON Web Token (JWT) 是一个开放标准 ( [RFC 7519](https://tools.ietf.org/html/rfc7519) )，它定义了一种紧凑且自包含的方式，用于在各方之间以 JSON 对象的形式安全传输信息。此信息可以验证和信任，因为它是数字签名的。JWT 可以使用密钥（使用**HMAC算法）或使用** **RSA**或**ECDSA**的公钥/私钥对进行签名。

虽然 JWT 可以加密以在各方之间提供保密性，但我们将专注于*签名*令牌。签名的令牌可以验证其中包含的声明的*完整性*，而加密的令牌会向其他方*隐藏*这些声明。当使用公钥/私钥对对令牌进行签名时，签名还证明只有持有私钥的一方才是签署它的一方。

## 什么时候应该使用 JWT ？

* **授权**：这是使用 JWT 最常见的场景。用户登录后，每个后续请求都将包含 JWT，从而允许用户访问该令牌允许的路由、服务和资源。单点登录是当今广泛使用 JWT 的一项功能，因为它的开销很小并且能够在不同的域中轻松使用。

* **信息交换**：JSON Web 令牌是在各方之间安全传输信息的好方法。因为可以对 JWT 进行签名（例如，使用公钥/私钥对），所以您可以确定发件人就是他们所说的那个人。此外，由于使用标头和有效负载计算签名，您还可以验证内容没有被篡改。

## JWT 的结构是什么？

在其紧凑的形式中，JSON Web Tokens 由以点 ( `.`) 分隔的三部分组成，它们是：

- 标题
- 有效载荷
- 签名

因此，JWT 通常如下所示。

`xxxxx.yyyyy.zzzzz`

让我们分解不同的部分。

### 标题

标头*通常*由两部分组成：令牌的类型，即 JWT，以及正在使用的签名算法，例如 HMAC SHA256 或 RSA。

例如：

```
{
  "alg": "SH256",
  "typ": "JWT"
}
```

然后，这个 JSON 被**Base64Url**编码以形成 JWT 的第一部分。

### 有效载荷

令牌的第二部分是有效负载。

一个示例有效载荷可能是：

```
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后对有效负载进行**Base64Url**编码以形成 JSON Web 令牌的第二部分。

> 请注意，对于已签名的令牌，此信息虽然受到保护以防篡改，但任何人都可以读取。除非已加密，否则请勿将机密信息放入 JWT 的有效负载或标头元素中。

### 签名

要创建签名部分，您必须获取编码的标头、编码的有效负载、秘密、标头中指定的算法，并对其进行签名。

例如，如果您想使用 HMAC SHA256 算法，签名将通过以下方式创建：

> sha256是不可逆的。因为sha256是一个确定的单向哈希函数，是美国国家安全局开发的SHA-2加密哈希函数的成员之一。也就是说sha256是一个数学函数，接受任意大小的输入，但返回固定大小的输入，就像文件或字符串的数字指纹。
> 
> 同时，它也是确定性的，因为相同的输入总是产生相同的输出。所谓不可逆，就是当你知道x的HASH值，无法求出x；所谓无冲突，就是当你知道x，无法求出一个y， 使x与y的HASH值相同。
> 
> 不管输入长度多少，它都会返回一个64个字符的字符串。
> 
> sha256很安全，原因是：只有输入相同的文件或字符串才能获得相同哈希值，即使是小小的调整也会完全改变输出的哈希值。sha256是单向哈希函数，因此是不可逆。

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

签名用于验证消息在此过程中没有被更改，并且在使用私钥签名的令牌的情况下，它还可以验证 JWT 的发送者就是它所说的那个人。

### 把所有的放在一起

输出是三个用点分隔的 Base64-URL 字符串，可以在 HTML 和 HTTP 环境中轻松传递，同时与基于 XML 的标准（如 SAML）相比更紧凑。

`header.payload.secret`

## JWT 令牌如何工作 ？

### 生成 Token

```go
type MyClaims struct {
   Username string `json:"username"`
   jwt.StandardClaims
}

// 定义过期时间
const TokenExpireDuration = time.Hour * 2

//定义secret（私钥）
var MySecret = []byte("这是一段用于生成token的密钥")

//生成jwt
func GenToken(username string) (string, error) {
    // 这是payload（负载）
   c := MyClaims{
      username,
      jwt.StandardClaims{
         // 这里设置过期时间，是这个中间件自带的功能，不是JWT的统一实现
         ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(),
         Issuer:    "my-project",
      },
   }

   //使用指定的签名方法创建签名对象
   // 这里定义加密方法，也就是header中的alg为“HS256"
   token := jwt.NewWithClaims(jwt.SigningMethodHS256, c)

   //使用指定的secret签名并获得完成的编码后的字符串token
   // 这里生成公钥，也就是签名secret
   return token.SignedString(MySecret)
}
```

### 解析 Token

```go
//解析JWT
func ParseToken(tokenString string) (*MyClaims, error) {
   //解析token
   // 这里通过
   token, err := jwt.ParseWithClaims(tokenString, &MyClaims{}, func(token *jwt.Token) (i interface{}, err error) {
      return MySecret, nil
   })
   // 这里判断签名是否被修改
   if err != nil {
      return nil, err
   }
   // 这里判断负载中的 Token 是否过期了
   if claims, ok := token.Claims.(*MyClaims); ok && token.Valid {
      return claims, nil
   }
   return nil, errors.New("invalid token")
}
```

### 编写基于JWT认证的中间件

```go
//基于JWT认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
   return func(c *gin.Context) {
      authHeader := c.Request.Header.Get("Authorization")
      if authHeader == "" {
         c.JSON(http.StatusOK, gin.H{
            "code": 2003,
            "msg":  "请求头中的auth为空",
         })
         //阻止调用后续的函数
         c.Abort()
         return
      }
      parts := strings.SplitN(authHeader, " ", 2)

      if !(len(parts) == 2 && parts[0] == "Bearer") {
         c.JSON(http.StatusOK, gin.H{
            "code": 2004,
            "msg":  "请求头中的auth格式错误",
         })
         //阻止调用后续的函数
         c.Abort()
         return
      }
      mc, err := ParseToken(parts[1])
      if err != nil {
         c.JSON(http.StatusOK, gin.H{
            "code": 2005,
            "msg":  "无效的token",
         })
         //阻止调用后续的函数
         c.Abort()
         return
      }
      //将当前请求的username信息保存到请求的上下文c上
      c.Set("username", mc.Username)
      //后续的处理函数可以通过c.Get("username")来获取请求的用户信息
      c.Next()
   }
}
```

### 注册一条获取 Token 的路由

```go
type UserInfo struct {
   Username string `json:"username"`
   Password string `json:"password"`
}

func authHandler(c *gin.Context) {
   var user UserInfo
   err := c.ShouldBind(&user)
   if err != nil {
      c.JSON(http.StatusOK, gin.H{
         "code": 2001,
         "msg":  "无效的参数",
      })
      return
   }

   if user.Username == "cyl" && user.Password == "123456" {
      //生成token
      tokenString, _ := GenToken(user.Username)
      c.JSON(http.StatusOK, gin.H{
         "code": 200,
         "msg":  "success",
         "data": gin.H{"token": tokenString},
      })
      return
   }

   c.JSON(http.StatusOK, gin.H{
      "code": 2002,
      "msg":  "鉴权失败",
   })
   return
}
```

## 为什么后端要使用 Redis 存储 Token ？

> 这里不需要鉴权，因为Token能存放在Redis中的前提就是这个Token就是Server生成的，而不是前端传递的。使用Redis存放Token有一个好处就是不用判断Token是否被修改，因为如果被修改了，那么ZSet一定没有对应的（Token，TimeStamp），就需要用密码登录了。

ZSet（Token，TimeStamp）

String: (UserID, Token)  作用：在使用密码登录的时候将旧的Token从ZSet中删除。

### 用户登录时：

* 传 Token 还是传 密码
  
  * 密码：生成新 Token，根据UserID找到（UserID， Token）中的旧Token，然后删除，将新的Token放入ZSet中，然后设置一个过期时间。更新（UserID，Token）中的Token为新Token。
  
  * Token： 如下。

* ZSet 是否存在 Token 这个 Key
  
  * 存在：更新ZSet中的Token对应的过期时间戳。
  
  * 不存在：可能设备是旧设备或者Token过期，需要重新使用密码登录。

### 用户鉴权时：

* 传 Token

* ZSet 中是否存在 Token 这个 Key
  
  * 存在：开始鉴权
  
  * 不存在：可能设备是旧设备或者Token过期，需要重新使用密码登录。

### 定时任务

用一个定时任务来清理掉ZSet中过期的Token。

## 怎样实现单点登录？
