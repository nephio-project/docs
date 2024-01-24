---
title: WebUI OIDC authenticaiton with Okta
description: >
   These instructions explain how to set up the Nephio WebUI to use OIDC with Okta for authentication.
weight: 6
---

If you are not exposing the webui on a load balancer IP address, but are instead using `kubectl port-forward`, you
should use `localhost` and `7007` for the `HOSTNAME` and `PORT`; otherwise, use the DNS name and port as it will be seen
by your browser.

## Creating an Okta Application

Adapted from the [Backstage](https://backstage.io/docs/auth/okta/provider#create-an-application-on-okta)
documentation:

1. Log into Okta (generally company.okta.com)
2. Navigate to Menu >> Applications >> Applications >> Create App Integration
3. Fill out the Create a new app integration form:

   - Sign-in method: OIDC - OpenID Connect
   - Application type: Web Application
   - Click Next

4. Fill out the New Web App Integration form:

   - App integration name: Nephio Web UI (or any other name you wish)
   - Grant type: Authorization Code & Refresh Token
   - Sign-in redirect URIs: http://HOSTNAME:PORT/api/auth/okta/handler/frame
   - Sign-out redirect URIs: http://HOSTNAME:PORT
   - Controlled access: (select as appropriate)
   - Click Save

## Create the Secret in the Cluster

The values created for the Okta application must be added to a Kubernetes Secret to that they can be added to the
container environment.

In the secret, use these keys:

| Key            | Description                                                 |
| -------------- | ----------------------------------------------------------- |
| client-id      | The client ID that you generated on Okta, e.g. 3abe134ejxzF21HU74c1 |
| client-secret  | The client secret shown for the Application.                |
| audience       | The Okta domain shown for the Application, e.g. https://www.okta.com/company/ |
| auth-server-id | The authorization server ID for the Application (optional)  |
| idp            | The identity provider for the application, e.g. 0oaulob4BFVa4zQvt0g3 (optional) |

This can be done via a secrets manager or by manually provision the secret (replacing the placeholders here):

```bash
kubectl create ns nephio-webui
kubectl create secret generic -n nephio-webui nephio-okta-oauth-client \
   --from-literal=client-id=CLIENT_ID \
   --from-literal=client-secret=CLIENT_SECRET \
   --from-literal=audience=AUDIENCE \
   --from-literal=auth-server-id=AUTH_SERVER_ID \
   --from-literal=idp=IDP
```

## Enable the WebUI Auth Provider

The webui package has a function that will configure the package for authentication with different services. Edit the
`set-auth.yaml` file to set the `authProvider` field to `oidc` and the `oidcTokenProvider` to `okta`, or run these
commands:

```bash
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-name set-auth -- 'by-path=authProvider' 'put-value=oidc'
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-name set-auth -- 'by-path=oidcTokenProvider' 'put-value=okta'
```
