# Kong / Keycloak Only support kong 2.x.x 

## Credits

[Securing APIs with Kong and Keycloak - Part 1](https://www.jerney.io/secure-apis-kong-keycloak-1/) by Joshua A Erney and 
[Example Thank you very much](https://github.com/d4rkstar/kong-konga-keycloak)

## Installed versions

- Kong 2.8.3 - alpine
- Keycloak 20.0.1

## Noted

- only support kong 2.x.x 

## 1. Create the image of Kong + Oidc

create dockerfile and use lua rock with [kong-oidc](https://github.com/nokia/kong-oidc)

### 1.1 Construction of the docker image

build image for kong-oidc

```bash
docker-compose build kong
```

## 2. Kong DB + Database Migrations


Start kong-db service:

```bash
docker-compose up -d kong-db
```

migrations:

```bash
docker-compose run --rm kong kong migrations bootstrap
```

:raised_hand: In case you're upgrading kong from previous versions, probably you may need to run migrations. In this case, you can give this command:

```bash
docker-compose run --rm kong kong migrations up
```

start kong:

```bash
docker-compose up -d kong
```

check service running:

```bash
docker-compose ps
```

check plug in available:

```bash
curl -s http://localhost:8001 | jq .plugins.available_on_server.oidc
```

The result of this call should be `true`. The presence of the plugin does not indicate that it is
already active.

## 3. Creation of a service and a route

create service for mock api

```bash
$ curl -s -X POST http://localhost:8001/services \
    -d name=mock-service \
    -d url=https://{{GUID}}.mockapi.io/api/v1/:{{endpoint}}
```

copy service id and create route to the service

```bash
$ curl -s -X POST http://localhost:8001/services/{{service id}}/routes -d "paths[]=/mock"
```

Test:

```bash
$ curl -s http://localhost:8000/mock
```

# 4. Keycloak containers

Create keycloak service:

```bash
docker-compose up -d keycloak-db
```

```bash
docker-compose up -d keycloak
```

Keycloak will be available at the url [http://localhost:8180](http://localhost:8180).

config Realm, client secret

## 5. Kong configuration as Keycloak client

Config oidc plugin 

```bash
$ curl -s -X POST http://localhost:8001/plugins \
  -d name=oidc \
  -d config.client_id=${CLIENT_ID} \
  -d config.client_secret=${CLIENT_SECRET} \
  -d config.bearer_only=yes \
  -d config.realm=${REALM} \
  -d config.introspection_endpoint=http://${HOST_IP}:8180/realms/${REALM}/protocol/openid-connect/token/introspect \
  -d config.discovery=http://${HOST_IP}:8180/auth/realms/${REALM}/.well-known/openid-configuration \
```

Get access token

```bash
TOKEN=$(curl -s -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "username=demouser" \
        -d "password=demouser" \
        -d 'grant_type=password' \
        -d "client_id=myapp" \
        http://${HOST_IP}:8180/realms/${REALM}/protocol/openid-connect/token \
        |jq . )

echo $TOKEN
```

```bash
export TKN=$(echo $RAWTKN | jq -r '.access_token')
echo $TKN
```

Test access api with token
```bash
curl "http://${HOST_IP}:8000/mock" \
-H "Accept: application/json" \
-H "Authorization: Bearer $TKN"
```