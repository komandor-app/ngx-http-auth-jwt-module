# Intro
This is an NGINX module to check for a valid JWT and proxy to an upstream server or redirect to a login page.

## Building and testing
To build the Docker image, start NGINX, and run our Bash test against it, run
```bash
make
```

When you make a change to the module, run `make rebuild-nginx`.

When you make a change to `test.sh`, run `make rebuild-test-runner`.

| Command                    | Description                                 |
| -------------------------- |:-------------------------------------------:|
| `make build-nginx`         | Builds the NGINX image                      |
| `make rebuild-nginx`       | Re-builds the NGINX image                   |
| `make build-test-runner`   | Builds the image that will run `test.sh`    |
| `make rebuild-test-runner` | Re-builds the image that will run `test.sh` |
| `make start-nginx`         | Starts the NGINX container                  |
| `make stop-nginx`          | Stops the NGINX container                   |
| `make test`                | Runs `test.sh` against the NGINX container  |

You can re-run tests as many times as you like while NGINX is up.
When you're done running tests, make sure to stop the NGINX container.

The Dockerfile builds all of the dependencies as well as the module,
downloads a binary version of NGINX, and runs the module as a dynamic module.

Tests get executed in containers. This project is 100% Docker-ized.

## Dependencies
This module depends on the [JWT C Library](https://github.com/benmcollins/libjwt)

Transitively, that library depends on a JSON Parser called
[Jansson](https://github.com/akheron/jansson) as well as the OpenSSL library.

## NGINX Directives
This module requires several new `nginx.conf` directives,
which can be specified in on the `main` `server` or `location` level.

```
auth_jwt_key "00112233445566778899AABBCCDDEEFF00112233445566778899AABBCCDDEEFF";
auth_jwt_enabled on;
auth_jwt_algorithm HS256; # or RS256
auth_jwt_validate_email on;  # or off
```

So, a typical use would be to specify the key on the main level and then only 
turn on the locations that you want to secure (not the login page).  Unauthorized 
requests are given 401 "Unauthorized" responses, you can redirect them with the 
nginx's `error_page` directive.

```
location @login_redirect {
  allow all;
  return 302 https://yourdomain.com/loginpage;
}

location /secure-location/ {
  auth_jwt_enabled   on;
  error_page  401 = @login_redirect;
}
```

```
auth_jwt_validation_type AUTHORIZATION;
auth_jwt_validation_type COOKIE=rampartjwt;
```
By default the authorization header is used to provide a JWT for validation.
However, you may use the `auth_jwt_validation_type` configuration to specify the name of a cookie that provides the JWT.



The default algorithm is 'HS256', for symmetric key validation.
Also supported is 'RS256', for RSA 256-bit public key validation.

If using "auth_jwt_algorithm RS256;", then the 'auth_jwt_key' field must be set to your public key.
That is the public key, rather than a PEM certificate.  I.e.:

```
auth_jwt_key "-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0aPPpS7ufs0bGbW9+OFQ
RvJwb58fhi2BuHMd7Ys6m8D1jHW/AhDYrYVZtUnA60lxwSJ/ZKreYOQMlNyZfdqA
rhYyyUkedDn8e0WsDvH+ocY0cMcxCCN5jItCwhIbIkTO6WEGrDgWTY57UfWDqbMZ
4lMn42f77OKFoxsOA6CVvpsvrprBPIRPa25H2bJHODHEtDr/H519Y681/eCyeQE/
1ibKL2cMN49O7nRAAaUNoFcO89Uc+GKofcad1TTwtTIwmSMbCLVkzGeExBCrBTQo
wO6AxLijfWV/JnVxNMUiobiKGc/PP6T5PI70Uv67Y4FzzWTuhqmREb3/BlcbPwtM
oQIDAQAB
-----END PUBLIC KEY-----";
```

By default, the module will attempt to validate the email address field of the JWT, then set the x-email header of the
session, and will log an error if it isn't found.  To disable this behavior, for instance if you are using a different
user identifier property such as 'sub', set:

```
auth_jwt_validate_email off;
```
