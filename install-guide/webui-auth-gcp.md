# Google OAuth 2.0 or OIDC

These instructions explain how to set up the Nephio WebUI to use Google OAuth
2.0 for authentication, or using OIDC backed by Google authentication. When used
with the Web UI running in a GKE cluster, the users authorization roles will be
automatically syncrhonized based upon their IAM roles in GCP.

If you are not exposing the webui on a load balancer IP address, but are instead
using `kubectl port-forward`, you should use `localhost` and `7007` for the
`HOSTNAME` and `PORT`; otherwise, use the DNS name and port as it will be seen by
your browser.

## Creating an OAuth 2.0 Client ID

Google OAuth 2.0 requires a client ID and allows you to authenticate
against the GCP identity service. To create your client ID and secret:

1. Sign in to the [Google Console](https://console.cloud.google.com)
2. Select or create a new project from the dropdown menu on the top bar
3. Navigate to
   [APIs & Services > Credentials](https://console.cloud.google.com/apis/credentials)
4. Click **Create Credentials** and choose `OAuth client ID`
5. Configure an OAuth consent screen, if required
   - For scopes, select `openid`, `auth/userinfo.email`,
     `auth/userinfo.profile`, and `auth/cloud-platform`.
   - Add any users that will want access to the UI if using External user type
6. Set **Application Type** to `Web Application` with these settings:
   - *Name*: Nephio Web UI (or any other name you wish)
   - *Authorized JavaScript origins*: http://`HOSTNAME`:`PORT`
   - *Authorized redirect URIs*:
     http://`HOSTNAME`:`PORT`/api/auth/google/handler/frame
7. Click Create
8. Copy the client ID and client secret displayed

## Create the Secret in the Cluster

The client ID and client secret need to be added as a Kubernetes secret to your
cluster. This can be done via Secrets Manager or Vault integrations, or you can
manully provision the secret (replacing the placeholders here):

```bash
kubectl create ns nephio-webui
kubectl create secret generic -n nephio-webui nephio-google-oauth-client --from-literal=client-id=CLIENT_ID_PLACEHOLDER --from-literal=client-secret=CLIENT_SECRET_PLACEHOLDER
```

## Enable Google OAuth

The webui package has a function that will configure the package for
authentication with different services. Edit the `set-auth.yaml` file to set the
`authProvider` field to `google` or run this command:

```bash
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-name set-auth -- 'by-path=authProvider' 'put-value=google'
```
## Enable OIDC with Google

The webui package has a function that will configure the package for
authentication with different services. Edit the `set-auth.yaml` file to set the
`authProvider` field to `oidc` and the `oidcTokenProvider` to `google`, or run
these commands:

```bash
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-name set-auth -- 'by-path=authProvider' 'put-value=oidc'
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-name set-auth -- 'by-path=oidcTokenProvider' 'put-value=google'
```
