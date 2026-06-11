# Multi-Instance ZITADEL With One Login V2 Frontend

This guide starts from the ZITADEL docker compose setup in this directory and
turns it into a local multi-instance setup where:

- `localhost` serves the first/default instance.
- `auth2.local` serves a second virtual instance.
- One `zitadel-login` container serves Login V2 for both instances.
- API/admin automation and Login V2 use separate system users and key pairs.

The important application rule is: do not pin Login V2 URLs to one host. Use
relative Login V2 paths and pass the current instance host through the proxy.

The docker-compose.yml and the .env.example are gotton from [here](https://zitadel.com/docs/self-hosting/deploy/compose#stage-1-quickstart-2-minutes). Updated on June 11 2026.

This is a test setup for understanding how multi-instance zitadel works. Do not use this as-is in production.

## 1. Local DNS

Add the second domain to `/etc/hosts`:

```text
127.0.0.1 auth2.local
```

The examples below assume ZITADEL is published on local port `8080`:

```text
http://localhost:8080
http://auth2.local:8080
```

## 2. Create Two System Key Pairs

Create one key pair for administrative API calls and one key pair for the Login
V2 frontend.

```sh
openssl genrsa -traditional -out admin-api.pem 2048
openssl rsa -in admin-api.pem -outform PEM -pubout -out admin-api.pub

openssl genrsa -traditional -out login-client.pem 2048
openssl rsa -in login-client.pem -outform PEM -pubout -out login-client.pub
```

The `.pem` files are private keys. Keep them local. The `.pub` files are mounted
into ZITADEL so it can verify system-user JWTs.

## 3. Add ZITADEL Runtime Config

Create `config.yml` next to `docker-compose.yml`:

```yaml
SystemAPIUsers:
  - admin-api:
      Path: /secrets/admin-api/admin-api.pub
      Memberships:
        - MemberType: System
          Roles:
            - "SYSTEM_OWNER"
            - "IAM_OWNER"
  - login-client:
      Path: /secrets/login-client/login-client.pub
      Memberships:
        - MemberType: System
          Roles:
            - "IAM_LOGIN_CLIENT"
```

`admin-api` is intentionally separate from `login-client`.

- `SYSTEM_OWNER` lets the admin key create and manage virtual instances.
- `IAM_OWNER` lets the admin key update instance-level feature settings such as
  `loginV2.baseUri`.
- `IAM_LOGIN_CLIENT` is the role the Login V2 frontend needs to read login
  settings and work with auth requests.

## 4. Create and update `.env`

```sh
cp .env.example .env
```

Update the version of Zitadel:

```sh
ZITADEL_VERSION=v4.14.0
```

Add the Login V2 private key as a single-line base64 value:

```sh
# Append it to your .env file
echo "# Private key for Login V2" >> .env
echo LOGIN_SYSTEM_USER_PRIVATE_KEY=$(base64 -w0 login-client.pem) >> .env
```

Do not put the `admin-api.pem` private key in `.env`; use it only when creating
short-lived JWTs for API calls.

## 5. Update `zitadel-api`

In `docker-compose.yml`, under `zitadel-api`:

Change the command so ZITADEL loads `config.yml`:

```yaml
command: start-from-init --masterkey "${ZITADEL_MASTERKEY}" --config /extraconfig.yml
```

Mount the config and both public keys:

```yaml
volumes:
  - zitadel-bootstrap:/zitadel/bootstrap:rw # already exists and correct
  - ./config.yml:/extraconfig.yml:ro
  - ./admin-api.pub:/secrets/admin-api/admin-api.pub:ro
  - ./login-client.pub:/secrets/login-client/login-client.pub:ro
```

Make Login V2 settings relative, not host-pinned:

```yaml
environment:
  ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_REQUIRED: true # already exists and correct
  ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_BASEURI: /ui/v2/login/
  ZITADEL_OIDC_DEFAULTLOGINURLV2: /ui/v2/login/login?authRequest=
  ZITADEL_OIDC_DEFAULTLOGOUTURLV2: /ui/v2/login/logout?post_logout_redirect=
  ZITADEL_SAML_DEFAULTLOGINURLV2: /ui/v2/login/login?samlRequest=
```

These values are critical. If `ZITADEL_OIDC_DEFAULTLOGINURLV2` or an instance's
stored `loginV2.baseUri` is `http://localhost:8080/...`, then an auth flow that
starts on `auth2.local` can be redirected back to `localhost`.

## 6. Update `zitadel-login`

In `docker-compose.yml`, under `zitadel-login`:

Do not use the bootstrapped PAT for a shared multi-instance Login V2 frontend.
That PAT belongs to the first instance. Use the dedicated `login-client` system
user instead:

```yaml
environment:
  ZITADEL_API_URL: http://zitadel-api:8080 # Already exists and good
  NEXT_PUBLIC_BASE_PATH: /ui/v2/login # Already exists and good
  SYSTEM_USER_ID: login-client
  SYSTEM_USER_PRIVATE_KEY: ${LOGIN_SYSTEM_USER_PRIVATE_KEY}
  AUDIENCE: http://localhost:8080
  CUSTOM_REQUEST_HEADERS: "" # Already exists, but needs updating
```

Remove:

```yaml
ZITADEL_SERVICE_USER_TOKEN_FILE: /zitadel/bootstrap/login-client.pat
```

The proxy will inject the instance headers per request instead.

## 7. Add Traefik Routing For The Second Host

The ZITADEL API/UI service must receive all non-Login V2 paths for both hosts.
The Login V2 service must receive `/ui/v2/login` and root redirects for both
hosts.

Keep the existing `localhost` routers, then add equivalent `auth2.local`
routers.

### ZITADEL API/UI routers

Add these labels to `zitadel-api`:

```yaml
labels:
  - traefik.http.routers.zitadel-api-auth2-web.rule=Host(`auth2.local`) && PathPrefix(`/api`)
  - traefik.http.routers.zitadel-api-auth2-web.entrypoints=web
  - traefik.http.routers.zitadel-api-auth2-web.middlewares=zitadel-strip-api
  - traefik.http.routers.zitadel-api-auth2-web.service=zitadel-api
  - traefik.http.routers.zitadel-api-auth2-web.priority=200

  - traefik.http.routers.zitadel-auth2-web.rule=Host(`auth2.local`) && !PathPrefix(`/ui/v2/login`) && !PathPrefix(`/api`) && !Path(`/`)
  - traefik.http.routers.zitadel-auth2-web.entrypoints=web
  - traefik.http.routers.zitadel-auth2-web.service=zitadel-api
  - traefik.http.routers.zitadel-auth2-web.priority=100
```

If you enable TLS locally, add matching `websecure` routers as you do for the
default host. For development this is not really needed (unless you need stuff like passkeys to work).

### Login V2 routers and headers

Add per-host middleware and routers to `zitadel-login`:

```yaml
labels:
  - traefik.http.middlewares.zitadel-localhost-headers.headers.customrequestheaders.X-Zitadel-Instance-Host=localhost:8080
  - traefik.http.middlewares.zitadel-localhost-headers.headers.customrequestheaders.X-Zitadel-Public-Host=localhost:8080
  - traefik.http.middlewares.zitadel-auth2-headers.headers.customrequestheaders.X-Zitadel-Instance-Host=auth2.local:8080
  - traefik.http.middlewares.zitadel-auth2-headers.headers.customrequestheaders.X-Zitadel-Public-Host=auth2.local:8080

  - traefik.http.routers.zitadel-root-web.middlewares=zitadel-root-rewrite,zitadel-localhost-headers
  - traefik.http.routers.zitadel-login-web.middlewares=zitadel-localhost-headers

  - traefik.http.routers.zitadel-root-auth2-web.rule=Host(`auth2.local`) && Path(`/`)
  - traefik.http.routers.zitadel-root-auth2-web.entrypoints=web
  - traefik.http.routers.zitadel-root-auth2-web.middlewares=zitadel-root-rewrite,zitadel-auth2-headers
  - traefik.http.routers.zitadel-root-auth2-web.service=zitadel-login
  - traefik.http.routers.zitadel-root-auth2-web.priority=400

  - traefik.http.routers.zitadel-login-auth2-web.rule=Host(`auth2.local`) && PathPrefix(`/ui/v2/login`)
  - traefik.http.routers.zitadel-login-auth2-web.entrypoints=web
  - traefik.http.routers.zitadel-login-auth2-web.middlewares=zitadel-auth2-headers
  - traefik.http.routers.zitadel-login-auth2-web.service=zitadel-login
  - traefik.http.routers.zitadel-login-auth2-web.priority=250
```

The key headers are:

```text
X-Zitadel-Instance-Host: localhost:8080
X-Zitadel-Instance-Host: auth2.local:8080
```

Without them, the Login V2 server-side API calls can use the wrong instance
context even though direct browser calls to ZITADEL work.

## 8. Start ZITADEL

Validate and start:

```sh
docker compose config -q
docker compose up -d
```

Open the first instance:

```text
http://localhost:8080/ui/console
```

Default credentials:
Username: `zitadel-admin@zitadel.localhost`
Password: `Password1!`

## 9. Create An Admin API Token

Use the admin key for setup calls:

```sh
export ZITADEL_TOKEN=$(zitadel-tools key2jwt \
  --audience=http://localhost:8080 \
  --key=admin-api.pem \
  --issuer=admin-api)
```

The `issuer` must match the system user name in `config.yml`.

## 10. Create The Second Instance

Create an instance with `auth2.local` as its custom domain:

```sh
curl --request POST \
  --url http://localhost:8080/system/v1/instances/_create \
  --header "Authorization: Bearer $ZITADEL_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "instanceName": "Instance 2",
    "firstOrgName": "Instance 2",
    "customDomain": "auth2.local",
    "human": {
      "userName": "admin",
      "profile": {
        "firstName": "Local",
        "lastName": "Admin"
      },
      "email": {
        "email": "admin@example.com",
        "isEmailVerified": true
      },
      "password": {
        "password": "Password1!",
        "passwordChangeRequired": false
      }
    }
  }' | jq
```

Check that both hosts resolve to different ZITADEL instances:

```sh
curl --request POST \
  --url http://localhost:8080/zitadel.instance.v2.InstanceService/GetInstance \
  --header "Authorization: Bearer $ZITADEL_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{}'  | jq

curl --request POST \
  --url http://auth2.local:8080/zitadel.instance.v2.InstanceService/GetInstance \
  --header "Authorization: Bearer $ZITADEL_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{}' | jq
```

## 11. Set Login V2 Feature Values On Both Instances

Existing or newly initialized instances can have instance-level feature values
stored in ZITADEL. Set both instances to use the shared Login V2 frontend with a
relative base URI:

This should not be needed, the default is already set this way.

```sh
curl --request PUT \
  --url http://localhost:8080/v2/features/instance \
  --header "Authorization: Bearer $ZITADEL_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "loginV2": {
      "required": true,
      "baseUri": "/ui/v2/login/"
    }
  }' | jq

curl --request PUT \
  --url http://auth2.local:8080/v2/features/instance \
  --header "Authorization: Bearer $ZITADEL_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "loginV2": {
      "required": true,
      "baseUri": "/ui/v2/login/"
    }
  }' | jq
```

Verify:

```sh
curl --request GET \
  --url http://auth2.local:8080/v2/features/instance \
  --header "Authorization: Bearer $ZITADEL_TOKEN" | jq
```

The response should contain:

```json
"loginV2": {
  "required": true,
  "baseUri": "/ui/v2/login/"
}
```

## 12. Test The Login Redirect

Open:

```text
http://auth2.local:8080/ui/console
```

The browser should go to:

```text
http://auth2.local:8080/oauth/v2/authorize?...
```

Then ZITADEL should redirect to a relative Login V2 URL:

```text
/ui/v2/login/login?authRequest=V2_...
```

If the redirect location is `http://localhost:8080/ui/v2/login/...`, then one
of these values is still host-pinned:

- `ZITADEL_OIDC_DEFAULTLOGINURLV2`
- `ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_BASEURI`
- the stored instance feature `loginV2.baseUri`

## Helm Notes

The same logic applies outside Docker Compose:

- Run one Login V2 deployment.
- Use a Login V2 system user with only `IAM_LOGIN_CLIENT`.
- Use a separate admin/setup system user for instance creation and feature
  changes.
- Preserve or inject `x-zitadel-instance-host` per external hostname before the
  request reaches Login V2.
- Keep Login V2 URLs relative unless each instance deliberately has its own
  dedicated login host.
