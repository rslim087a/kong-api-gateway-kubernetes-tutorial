# Kong API Gateway Kubernetes Tutorial
This repository contains all the code needed to follow along with our **[YouTube Tutorial](https://youtu.be/rTcj7znJVZc)**.

## Part 1: Basic Setup (Without Kong)

### 1. Apply the starter manifests:
```bash
kubectl apply -f starter-manifests.yaml
```

### 2. Test the API (Basic Setup):
```bash
# Add grades
curl -X POST http://localhost:3000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Harry", "subject": "Defense Against Dark Arts", "score": 95}'

curl -X POST http://localhost:3000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Ron", "subject": "Charms", "score": 82}'

curl -X POST http://localhost:3000/grades \
  -H "Content-Type: application/json" \
  -d '{"name": "Hermione", "subject": "Potions", "score": 98}'

# Get all grades
curl http://localhost:3000/grades
```

## Part 2: Kong API Gateway Implementation

### 1. Kong Plugin Templates
Copy and customize these templates for Kong implementation:

```yaml
# kong-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: <your-ingress-name>
 namespace: <your-namespace>
 annotations:
   konghq.com/strip-path: "false"
   konghq.com/plugins: <auth-plugin-name>, <ratelimit-plugin-name>    # Plugin names referenced in KongPlugin resources
   kubernetes.io/ingress.class: kong
   konghq.com/methods: "GET, POST"  
spec:
 ingressClassName: kong
 rules:
   - http:
       paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: <your-service-name>   # Name of your K8s Service
               port:
                 number: <service-port>    # Port exposed by your Service

# kong-plugins.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
 name: <ratelimit-plugin-name>     # Must match name in Ingress annotation
 namespace: <your-namespace>
 annotations:
  kubernetes.io/ingress.class: kong
config:
 minute: <requests-per-minute>     # Number of requests allowed per minute
 limit_by: consumer
 policy: local
plugin: rate-limiting
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
 name: <auth-plugin-name>          # Must match name in Ingress annotation
 namespace: <your-namespace>
 annotations:
  kubernetes.io/ingress.class: kong
config:
 key_names:
   - apikey                        # Header name for the API key
 hide_credentials: true
plugin: key-auth

# kong-consumer.yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
 name: <consumer-name>             # Unique name for this consumer
 namespace: <your-namespace>
 annotations:
  kubernetes.io/ingress.class: kong
username: <username>                # Username for this consumer
custom_id: <custom-id>              # Unique ID for this consumer
credentials:
 - <secret-name>                   # Name of the Secret containing the API key

# kong-secret.yaml
apiVersion: v1
kind: Secret
metadata:
 name: <secret-name>               # Must match name referenced in KongConsumer
 namespace: <your-namespace>
 labels:
   konghq.com/credential: key-auth
type: Opaque
stringData:
 kongCredType: key-auth
 key: <your-api-key>               # Your chosen API key value
```

### 2. Test API with Kong Security:
```bash
# Replace <node-port> with Kong proxy port and your-secret-key with the key from kong-secret.yaml
# Add grades
curl -X POST http://localhost:<port>/grades \
  -H "apikey: your-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"name": "Harry", "subject": "Defense Against Dark Arts", "score": 95}'

# Get all grades
curl http://localhost:<port>/grades -H "apikey: your-secret-key"
```

## Become a Cloud and DevOps Engineer

Learn every tool that matters: [https://rslim087a.github.io/rayanslim](https://rslim087a.github.io/rayanslim)
