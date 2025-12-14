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

### Part 2b: Kong API Gateway Implementation ensure to chnge proxy type to reflect our implementation type, LoadBalancer type is for clod providers, NodePort is for local implementation
helm install kong/kong -n kong --create-namespace kong --version 2.47.0 --set proxy.type=NodePort 


### 1. Kong Plugin Templates
Copy and customize these templates for Kong implementation:

```yaml


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

## Create 

Deploy Kong to map Kind cluster host port to Kong ingress
helm install idp-kong kong/kong \
  --namespace kong --create-namespace \
  --version 2.47.0 \
  --set proxy.type=NodePort \
  --set proxy.http.nodePort=30080 \
  --set proxy.tls.nodePort=30443
