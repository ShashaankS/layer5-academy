---
docType: "Chapter"
chapterTitle: "Service security capabilities"
description: "In this chapter, you will learn how to secure your microservices using Istio's service security capabilities, including access control, identity verification, and mutual TLS."
lectures: 12
weight: 6
id: "service-security-capabilities"
title: "Service Security Capabilities of Istio"
---

{{< chapterstyle >}}

### Access Control

You will start by denying all traffic.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec: {}
```

#### And then begin poking holes in your service mesh "firewall".

```yaml
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "allow-get"
  namespace: default
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
    - to:
        - operation:
            methods: ["GET"]
```

#### Create AuthorizationPolicy for each BookInfo service.

```yaml
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "view-productpage"
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
    - to:
        - operation:
            methods: ["GET", "POST"] # try login with just GET (fails)
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "view-details"
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
      to:
        - operation:
            methods: ["GET", "POST"]
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "view-reviews"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
      to:
        - operation:
            methods: ["GET", "POST"]
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "view-ratings"
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
      to:
        - operation:
            methods: ["GET", "POST"]
```

#### Allow per user access

```yaml
---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "view-reviews"
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
      to:
        - operation:
            methods: ["GET"]
      when:
        - key: request.headers[end-user]
          values: ["naruto"]
```

#### Reset BookInfo Subsets (reset destination rules)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
    - name: v1
      labels:
        version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v2-mysql
      labels:
        version: v2-mysql
    - name: v2-mysql-vm
      labels:
        version: v2-mysql-vm
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
```

### Identity Verification

Note: this lab uses the sample application HTTPbin.

Using Meshery, deploy the HTTPbin sample application.

#### Add Claims

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
      when:
        - key: request.auth.claims[groups]
          values: ["group1"]
```

#### Def

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt"
  namespace: default
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
    - issuer: "testing@secure.istio.io"
      jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.7/security/tools/jwt/samples/jwks.json"
```

### Mutual TLS

Using Meshery, you can change mTLS enforcement for a namespace.

To configure mTLS on more selective level, you can change and apply this configuration:

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  # selector:
  #   matchLabels:
  #     app: httpbin
  mtls:
    mode: STRICT #ISTIO_MUTUAL,DISABLE
  # portLevelMtls:
  #   80:
  #     mode: DISABLE
```

{{< /chapterstyle >}}