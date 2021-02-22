JSON Web Tokens are an open, industry standard [RFC 7519](https://tools.ietf.org/html/rfc7519) method for representing claims securely between two parties.



JWT（JSON Web Token）

client-side based stateless session

stateless and distributed architecture microservices



serveral parties communicating with each other



>1. header
>   alg（sign alg）、typ（JWT）
>
>2. payload
>   contain the *claims* of the token
>
>3. signature
>   signature = Crypto(secret, base64(header), base64(payload))
>
>

![](imgs\jwt\payload_standard_fields.png)