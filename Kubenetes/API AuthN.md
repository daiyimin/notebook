# API Authentication [^1]
[TOC]
API requests are tied to either a normal user or a service account, or are treated as anonymous requests.
## Users in K8S
Users are humans that sends request to API server.

Kubernetes does not have objects which represent normal user accounts. Normal users cannot be added to a cluster through an API call.

Kubenetes supports following ways of authentication users:
* X509 Certificate
* Static Tokens
* TokenRequest API tokens
* OpenId Connect Tokens
* Webhook Token Authentication

### X509 Certificate
Even though a normal user cannot be added via an API call, any user that presents a valid certificate signed by the cluster's certificate authority (CA) is considered authenticated. In this configuration, Kubernetes determines the username from the common name field in the 'subject' of the cert (e.g., "/CN=bob"). From there, the role based access control (RBAC) sub-system would determine whether the user is authorized to perform a specific operation on a resource. 

#### user group in X509 Certificate
Kubernetes determines the user group from orgnaization field in the 'subject' of the cert (e.g., "/O=system:master). In this example, the user of this certificate belongs to system:master group. 
system:master group is a reserved group. It is bond to cluster admin role. See API AuthZ.

### Static Tokens
If API server is started with -token-auth-file=SOMEFILE option on the command line, it reads allowed bearer tokens from the file.
The token file is a csv file with a minimum of 3 columns: token, user name, user uid, followed by optional group names. For example:
```
token,user,uid,"group1,group2,group3"
```
When using bearer token authentication from an http client, the API server expects an Authorization header with a value of Bearer \<token>.
```
Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
```
API server shall match the bearer token with allowed tokens. Kubenetes determines the username from user name column of the matched token.

### OpenID Connect Tokens
Open ID is an extension of OAuth2. In the OAuth2 response, an id_token containing info of the authenticated user is returned to client.
When using OpenID connect token to autenticate an API user, http client shall first get the id_token from OpenID connect server.
Next, http client shall send base64 encoded id_token as Bearer \<token> value in Authorization header.
The raw id_token (base64 decoded) is a JWT compliant token. The minimum valid JWT payload must contain the following claims:
```
{
  "iss": "https://example.com",   // must match the issuer.url
  "aud": ["my-app"],              // at least one of the entries in issuer.audiences must match the "aud" claim in presented JWTs.
  "exp": 1234567890,              // token expiration as Unix time (the number of seconds elapsed since January 1, 1970 UTC)
  "<username-claim>": "user"      // this is the username claim configured in the claimMappings.username.claim or claimMappings.username.expression
}
```
The id_token is signed by the issuer (OpenID Connect server). Kubenetes shall verify the the signature to prevent spoofing attack on id_tokien.
#### Configuration
API server shall be configured to enable OpenID Connect token authenticator.
The configuration looks like:
```
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
jwt:
- issuer:
    url: https://example.com
    audiences:
    - my-app
  claimMappings:
    username:
      expression: 'claims.username + ":external-user"'
    groups:
      expression: 'claims.roles.split(",")'
    uid:
      expression: 'claims.sub'
    extra:
    - key: 'example.com/tenant'
      valueExpression: 'claims.tenant'
  userValidationRules:
  - expression: "!user.username.startsWith('system:')" # the expression will evaluate to true, so validation will succeed.
    message: 'username cannot used reserved system: prefix'
```
- Note: the issuer url is OpenID discovery URL.

The token payload looks like:
```
  {
    "aud": "kubernetes",
    "exp": 1703232949,
    "iat": 1701107233,
    "iss": "https://example.com",
    "jti": "7c337942807e73caa2c30c868ac0ce910bce02ddcbfebe8c23b8b5f27ad62873",
    "nbf": 1701107233,
    "roles": "user,admin",
    "sub": "auth",
    "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db4a",
    "username": "foo"
  }
```
The token with the above AuthenticationConfiguration will produce the following UserInfo object and successfully authenticate the user.
```
{
    "username": "foo:external-user",
    "uid": "auth",
    "groups": [
        "user",
        "admin"
    ],
    "extra": {
        "example.com/tenant": "72f988bf-86f1-41af-91ab-2d7cd011db4a"
    }
}
```
#### OpenID Discory URL
The public key to verify the signature is discovered from the issuer's public endpoint using OIDC discovery. If you're using the authentication configuration file, the identity provider doesn't need to publicly expose the discovery endpoint. You can host the discovery endpoint at a different location than the issuer (such as locally in the cluster) and specify the issuer.discoveryURL in the configuration file.

### Webhook Token Authentication 
Webhook authentication is a hook for verifying bearer tokens.
The configuration file uses the kubeconfig file format. Within the file, clusters refers to the remote service and users refers to the API server webhook. An example would be:
```
# Kubernetes API version
apiVersion: v1
# kind of the API object
kind: Config
# clusters refers to the remote service.
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # CA for verifying the remote service.
      server: https://authn.example.com/authenticate # URL of remote service to query. 'https' recommended for production.

# users refers to the API server's webhook configuration.
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
      client-key: /path/to/key.pem          # key matching the cert

# kubeconfig files require a context. Provide one for the API server.
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-server
  name: webhook
```
When a client attempts to authenticate with the API server using a bearer token as discussed above, the authentication webhook POSTs a JSON-serialized <b>TokenReview</b> object containing the token to the remote service. In the example, opaque token is sent to webhook service.
```
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    # Opaque bearer token sent to the API server
    "token": "014fbff9a07c...",

    "audiences": ["https://myserver.example.com", "https://myserver.internal.example.com"]
  }
}
```
The remote service is expected to fill the status field of the request to indicate the success of the login. In the example, user info is returned in the status field.
```
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": ["developers", "qa"]
    },
    "audiences": ["https://myserver.example.com"]
  }
}
```
Webhook authentication is often used together with [client-go credential plugins](#client-go-credential-plugin). Client-go credential plugin enables Kubenets to integrate with external authN protocols like LDAP, SAML. The plugin implements the protocol specific logic, then returns opaque credentials to use. Almost all credential plugin use cases require a server side component with support for the webhook token authenticator to interpret the credential format produced by the client plugin.


## Service Account in K8S
A ServiceAccount provides an identity for processes that run in a Pod.
A process inside a Pod can use the identity of its associated service account to authenticate to the cluster's API server.
Service accounts are usually created automatically by the API server and associated with pods running in the cluster through the ServiceAccount Admission Controller. Bearer tokens are mounted into pods at well-known locations, and allow in-cluster processes to talk to the API server. Accounts may be explicitly associated with pods using the serviceAccountName field of a PodSpec.
In API server, service account authenticator is an automatically enabled authenticator that uses signed bearer tokens to verify requests. 

在 Kubernetes 中，服务帐户是一种为 Pod 内的进程提供的特殊的类型的凭据。这些凭据由 Kubernetes 自动管理，并允许 Pod 内的进程与 Kubernetes API 服务器进行交互。通过使用服务帐户，你可以更容易地为你的应用程序或工作负载提供必要的权限，同时保持对这些权限的严格控制。这有助于防止一个应用程序的不当行为或安全漏洞影响到其他应用程序。

### TokenRequest API tokens
From K8S 1.23, TokenRequest API tokens have replaced Service Account Secret and becomes the recommended way to obtain short-lived token for Service Account. See [Reference](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#manual-secret-management-for-serviceaccounts).

[ServiceAccount token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection) describes how to project a Service Account token into POD.
First, define a Pod using service account token projection in pods/pod-projected-svc-token.yaml.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: build-robot
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```
Then, deploy the POD
```
kubectl create -f https://k8s.io/examples/pods/pod-projected-svc-token.yaml
```
The kubelet will: request and store the token on behalf of the Pod; make the token available to the Pod at a configurable file path; and refresh the token as it approaches expiration. The kubelet proactively requests rotation for the token if it is older than 80% of its total time-to-live (TTL), or if the token is older than 24 hours.

The application is responsible for reloading the token when it rotates. It's often good enough for the application to load the token on a schedule (for example: once every 5 minutes), without tracking the actual expiry time.


## Anonymous requests
When enabled, requests that are not rejected by other configured authentication methods are treated as anonymous requests, and given a username of system:anonymous and a group of system:unauthenticated.

## Client-go credential plugin
Client-go credential plugin enables kubenetes to integrate with not natively supported authentication protocols, like LDAP, SAML etc.
The process of authenticate against API using  client-go credential plugin is:
* The user issues a kubectl command. 
* kubectl checks kubectl config file for credential plugin, in this case it's an LDAP credential plugin. kubectl execute the plugin command with an ExternalCredential input object.
* Credential plugin prompts the user for LDAP credentials, exchanges credentials with external service for a token.
* Credential plugin returns token to client-go in an ExternalCredential output object, which uses it as a bearer token against the API server.
* API server uses the webhook token authenticator to submit a TokenReview to the external service.
* External service verifies the signature on the token and returns the user's username and groups.

### Configuration
Credential plugins are configured through kubectl config files as part of the user fields. 
kubectl checks kubectl config file for credential plugin and call the plugin to acquisite token from external service.
### Token acquisition
After receiving request from kubectl, credential plugin will get opaque token from external service, such as LDAP, SAML etc.
Then the opaque token is returned to kubectl in an ExternalCredential output object.
### Token verification
kubectl call API with the opaque token. API server uses the [webhook token authenticator](#webhook-token-authentication) to submit a TokenReview to the external service. External service verifies the signature on the token and returns the user's username and groups.


[^1]: https://kubernetes.io/docs/reference/access-authn-authz/authentication/