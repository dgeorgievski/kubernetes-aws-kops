Dex SSO for Kubernetes CLI access
-----------------------------------

Dex is OpenID Connect Identity (OIDC) and OAuth 2.0 Provider with Pluggable Connectors.

Use Dex with GitHub connector to authenticate users and authorize them using RBAC + GitHub teams.

# Installation

## Dex and Gangway
Deploy Dex and Gangway Helm charts. 

Dex is using ingress that terminates the SSL connections with letsencrypt
certificates. Therefore it is not required to provide dex's certificate to the
API server for verification.

https://dex.dev.edpub.wiley.com/

## Configure K8S API for OIDC

Add the following settings to `kubeAPIServer` section:
```
kubeAPIServer:
  # https://github.com/heptiolabs/gangway
  # https://github.com/dexidp/dex/blob/master/Documentation/kubernetes.md
  oidcIssuerURL: https://dex.dev.example.com/dex
  oidcClientID: gangway
  oidcUsernameClaim: email
  oidcGroupsClaim: groups
```

Update k8s cluster following the kops update instructions.

## Accessing Gangway for authentication
1. Open in the browser https://k8slogin.dev.example.com/
2. Select GitHub and authenticate
3. If authentication was successful follow the instructions to setup
   the local kube config context for `WPNG Dev` cluster.
