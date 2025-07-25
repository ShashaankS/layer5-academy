---
docType: "Chapter"
chapterTitle: "Request Routing and Canary Testing"
description: "In this chapter, we are going to get our hands on some of the traffic management capabilities of Istio."
lectures: 12
weight: 5
title: "Request Routing and Canary Testing"
---

{{< chapterstyle >}}
In this chapter, we are going to get our hands on some of the traffic management capabilities of Istio.

### Apply default destination rules

Before we start playing with Istio's traffic management capabilities, we need to define the available versions of the deployed services. In Istio parlance, versions are called subsets. Subsets are defined in destination rules.

Run the following in the custom yaml section to create default destination rules for the Bookinfo services:

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
subsets: - name: v1
labels:
version: v1 - name: v2
labels:
version: v2 - name: v3
labels:
version: v3

---

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
name: ratings
spec:
host: ratings
subsets: - name: v1
labels:
version: v1 - name: v2
labels:
version: v2 - name: v2-mysql
labels:
version: v2-mysql - name: v2-mysql-vm
labels:
version: v2-mysql-vm

---

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
name: details
spec:
host: details
subsets: - name: v1
labels:
version: v1 - name: v2
labels:
version: v2

---

```

Using Meshery, navigate to the Istio management page:

1. Enter `default` in the `Namespace` field.
1. Click the (+) icon on the `Apply Service Mesh Configuration` card and select `Bookinfo subsets` from the list.

This will deploy the destination rules for all the Book info services defining their subsets. Verify the destination rules created by using the command below:

```sh
kubectl get destinationrules


kubectl get destinationrules -o yaml
```

### Configure the default route for all services to V1

As part of the bookinfo sample app, there are multiple versions of reviews service. When we load the `/productpage` in the browser multiple times we have seen the reviews service round robin between v1, v2 or v3. As the first exercise, let us first restrict traffic to just V1 of all the services.

Using Meshery, navigate to the Istio management page:

1. Enter `default` in the `Namespace` field.
2. Click the (+) icon on the `Apply Custom Configuration` card and paste the configuration below.

To view the applied rule:

```sh
kubectl get virtualservice
```

To take a look at a specific one:

```sh
kubectl get virtualservice reviews -o yaml
```

_Please note:_ In the place of the above command, we can either use kubectl or istioctl.

Config:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
---
```

Now when we reload the `/productpage` several times, we will ONLY be viewing the data from v1 of all the services, which means we will not see any ratings (any stars).

### Content-based routing

Let's replace our first rules with a new set. Enable the `ratings` service for a user `jason`
by routing `productpage` traffic to `reviews` v2:

Using Meshery, navigate to the Istio management page:

1. Enter `default` in the `Namespace` field.
2. Click the (+) icon on the `Apply Custom Configuration` card and paste the configuration below.

This will update the existing virtual service definition for reviews to route all traffic for user `jason` to review V2.

In a few, we should be able to verify the virtual service by using the command below:

```sh
kubectl get virtualservice reviews -o yaml
```

Config:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
---
```

Now if we login as your `jason`, you will be able to see data from `reviews` v2. While if you NOT logged in or logged in as a different user, you will see data from `reviews` v1.

### Canary Testing - Traffic Shifting


#### Canary testing w/50% load

To start canary testing, let's begin by transferring 50% of the traffic from reviews:v1 to reviews:v3 with the following command:

Using Meshery, navigate to the Istio management page:

1. Enter `default` in the `Namespace` field.
2. Click the (+) icon on the `Apply Custom Configuration` card and paste the configuration below.

This will update the existing virtual service definition for reviews to route 50% of all traffic to review V3.

In a few, we should be able to verify the virtual service by using the command below:

```sh
kubectl get virtualservice reviews -o yaml
```

Config:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
---
```

Now, if we reload the `/productpage` in your browser several times, you should now see red-colored star ratings approximately 50% of the time.

#### Shift 100% to v3

When version v3 of the reviews microservice is considered stable, we can route 100% of the traffic to reviews:v3:

Using Meshery, navigate to the Istio management page:

1. Enter `default` in the `Namespace` field.
2. Click the (+) icon on the `Apply Custom Configuration` card and paste the configuration below.

This will update the existing virtual service definition for reviews to route 100% of all traffic to review V3.

In a few, we should be able to verify the virtual service by using the command below:

```sh
kubectl get virtualservice reviews -o yaml
```

Config:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v3
---
```

Now, if we reload the `/productpage` in your browser several times, you should now see red-colored star ratings 100% of the time.


#### Alternative: Manual installation
Follow these steps if the above steps did not work

##### Default destination rules

Run the following command to create default destination rules for the Bookinfo services:

```sh
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

##### Route all traffic to version V1 of all services

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

##### Route all traffic to version V2 of reviews for user Jason

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

##### Route 50% of traffic to version V3 of reviews service

```sh
kubectl apply -f  samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

##### Route 100% of traffic to version V3 of reviews service

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

{{< /chapterstyle >}}
