---
title: Call a registered external service from Kyma
type: Getting Started
---

This guide shows how to call a registered external service from Kyma using a simple lambda function.


## Prerequisites

- A Remote Environment (RE) bound to the `production` Environment
- Client certificates generated for the connected RE.
- Map the `my-lambda-production.kyma.local` to your Minikube IP to call the lambda function on a local Kyma deployment.


## Steps

1. Register a service with the following specification to the desired Remote Environment.
>**NOTE:** To learn how to register a service, see the **Register a service** Getting Started Guide.
```json
{
  "name": "my-service",
  "provider": "myCompany",
  "Identifier": "identifier",
  "description": "This is some service",
  "api": {
    "targetUrl": "http://httpbin.org/",
    "spec": {
      "swagger":"2.0"
    }
  }
}
```

2. Get the `externalName` of the Service Class of the registered service.
```
kubectl -n production get serviceclass {SERVICE_ID}  -o jsonpath='{.spec.externalName}'
```

3. Create a Service Instance for the registered service.
```
cat <<EOF | kubectl apply -f -
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceInstance
metadata:
  name: my-service-instance-name
  namespace: production
spec:
  serviceClassExternalName: {EXTERNAL_NAME}
EOF
```

4. Create a lambda function that sends a request to the registered service with an additional path of `/uuid`. A successful response returns a UUID generated by `httpbin.org`. To create and register the lambda function in the `production` Environment, run:
```
cat <<EOF | kubectl apply -f -
apiVersion: kubeless.io/v1beta1
kind: Function
metadata:
  name: my-lambda
  namespace: production
spec:
  deployment:
    spec:
      template:
        metadata:
          labels:
            re-{RE_NAME}-{SERVICE_ID}: "true"
        spec:
          containers:
          - env:
            - name: GATEWAY_URL
              value: re-{RE_NAME}-{SERVICE_ID}.kyma-integration
  deps: |-
    {
        "name": "example-1",
        "version": "0.0.1",
        "dependencies": {
          "request": "^2.85.0"
        }
    }
  function: |-
    const request = require('request');

    module.exports = { main: function (event, context) {
        return new Promise((resolve, reject) => {
            const url = \`http://\${process.env.GATEWAY_URL}/uuid\`;
            const options = {
                url: url,
            };
              
            sendReq(url, resolve, reject)
        })
    } }

    function sendReq(url, resolve, reject) {
        request.get(url, { json: true }, (error, response, body) => {
            if(error){
                resolve(error);
            }
            resolve(response.body);
        })
    }
  function-content-type: text
  handler: handler.main
  horizontalPodAutoscaler:
    spec:
      maxReplicas: 0
  runtime: nodejs8
  service:
    ports:
    - name: http-function-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      created-by: kubeless
      function: my-lambda
  timeout: ""
  topic: http
EOF
```

5. Create a ServiceBinding and a ServiceBindingUsage to bind the Service Instance to the lambda function.

```
cat <<EOF | kubectl apply -f -
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceBinding
metadata:
  labels:
    Function: my-lambda
  name: my-service-binding
  namespace: production
spec:
  instanceRef:
    name: my-service-instance-name
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: servicecatalog.kyma.cx/v1alpha1
kind: ServiceBindingUsage
metadata:
  labels:
    Function: my-lambda
    ServiceBinding: my-service-binding
  name: my-service-binding
  namespace: production
spec:
  serviceBindingRef:
    name: my-service-binding
  usedBy:
    kind: function
    name: my-lambda
EOF
```

6. To expose the lambda function outside the cluster create an **Api** custom resource:
```
cat <<EOF | kubectl apply -f -
apiVersion: gateway.kyma.cx/v1alpha2
kind: Api
metadata:
  labels:
    function: my-lambda
  name: my-lambda
  namespace: production
spec:
  authentication: []
  hostname: my-lambda-production.{CLUSTER_DOMAIN}
  service:
    name: my-lambda
    port: 8080
EOF
```

7. To verify that everything was setup correctly you can now call the lambda through https:
  - On a cluster
    ```
    curl https://my-lambda-production.{CLUSTER_DOMAIN}/ -k
    ```
  - On a local deployment:
    ```
    curl https://my-lambda-production.kyma.local/ -k
    ```

A successful response returns a UUID generated by `httpbin.org`:
```json
{
  "uuid": "d44cc373-b26e-4a36-9890-6418d131a285"
}
```