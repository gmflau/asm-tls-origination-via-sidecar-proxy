# TLS Origination via Istio sidecar proxy



#### 1. Create a fully managed TLS-enabled Redis Enterprise database instance from GCP Marketplace
Create a Redis Enterprise databae via [GCP Marketplace](https://console.cloud.google.com/marketplace/details/redislabs-public/redis-enterprise)    
Collect the following information from Redis Enterprise Cloud Subscription Manager portal (app.redislabs.com):  
* Public endpoint service name  
* Public endpoint service port   
* Default user's password    
* Turn on SSL Client Authentication (Note: DO NOT enforce client authentication)
* Download the proxy CA certificate: redislabs_ca_cert.pem (Note: click 


  
#### 2. Create Service Entry for the external fully managed Redis Enterprise database instance
```
export REDIS_HOST=<REDB's public endpoint service name>
export REDIS_PORT=<REDB's public endpoint service port>
```  
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-tls-redb-${REDIS_PORT}
  namespace: redis
spec:
  hosts:
  - $REDIS_HOST
  location: MESH_EXTERNAL
  resolution: DNS
  ports:
  - number: $REDIS_PORT
    name: redis-port
    protocol: TCP  
EOF
```
   

#### 3. Create a K8 secret to store the proxy CA certificate
```
kubectl create secret generic redb-cert \
--from-file=redislabs_ca_cert.pem
```

   
#### 4. Create Destination Rule for the external fully managed Redis Enterprise database instance
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: external-tls-redb-${REDIS_PORT}
  namespace: redis
spec:
  host: $REDIS_HOST
  trafficPolicy:
    tls:
      mode: SIMPLE
      caCertificates: /etc/redb-cert/redislabs_ca_cert.pem
EOF
```  
  

#### 5. Deploy a redis-cli testing client
```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-client
  namespace: redis
  labels:
    app: redis-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-client
  template:
    metadata:
      labels:
        app: redis-client
      annotations:
        sidecar.istio.io/logLevel: debug
        sidecar.istio.io/inject: "true"
        sidecar.istio.io/userVolumeMount: '[{"name":"redb-cert", "mountPath":"/etc/redb-cert", "readonly":true}]'
        sidecar.istio.io/userVolume: '[{"name":"redb-cert", "secret":{"secretName":"redb-cert"}}]'
    spec:
      containers:
        - image: redis
          name: redis-client
          command: [ "/bin/bash", "-c", "--" ]
          args: [ "while true; do sleep 30; done;" ]
EOF
```
  

#### 6. Verify the TLS Origination connection
Run the following command to locate the redis-client pod name:  
```
kubectl get pods
```
Get a shell to the redis-client container:  
```
kubectl exec -it pod/redis-client-<----------->  -c redis-client  /bin/bash
  
For example,
kubectl exec -it pod/redis-client-6fdbfc948-9wgrc  -c redis-client  /bin/bash
```
Verify the connection using standard TCP to connect to a TLS-enabled Redis Enterprise database instance through Istio sidecar:  
```
redis-cli -h <REPLACE_WITH_REDIS_HOST> -p <REPLACE_WITH_REDIS_PORT> -a <REPLACE_WITH_YOUR_PASSWORD>
set "hello" "Redis Labs"
get "hello"
```
  

