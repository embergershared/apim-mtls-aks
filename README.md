# Enable mTLS authentication between APIM and AKS Ingress



## Setup TLS

### Create a Self-signed certificate for TLS and its Kubernetes secret

```pwsh

> Note: The clients should "skip" the certificate verification as it is self-signed

```pwsh
# Create a self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt -subj "/CN=whoami-ing.ebdemos.info/O=apim-mtls-aks repo"

# Create the Kubernetes TLS secret
kubectl create secret tls self-tls --key server.key --cert server.crt -n whoami
```

### Create Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-localhost
  namespace: whoami
spec:
  ingressClassName: nginx
  rules:
  - host: whoami-ing.ebdemos.info
    http:
      paths:
      - backend:
          service:
            name: demo-app
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - whoami-ing.ebdemos.info
    secretName: self-tls
```

### Test TLS Ingress

```pwsh
curl -k -v https://whoami-ing.ebdemos.info/
```

> `-k` to skip self-signed certificate verification
>
> `-v` to see connection logs

## Add mTLS authentication, on top of TLS

### Create a CA that will sign clients certificates and be known by the ingress controller

```pwsh
# Step 1: Create a CA (Certification Authority) self-signed certificate
# Create a CA (Certification Authority) self-signed certificate
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj '/CN=The Cert Authority'

# Create the Kubernetes Generic secret for the CA that will be used by the Ingress to authenticate the client
kubectl create secret generic ca-secret --from-file=ca.crt=ca.crt -n whoami
```

### Enable mTLS in the Ingress

```yaml
# Add these 4 annotations to the Ingress
metadata:
  annotations:
      nginx.ingress.kubernetes.io/auth-tls-verify-client: "on" # Turns ON/OFF mTLS verification. When ON getting a HTTP 400 for wrong client cert
      nginx.ingress.kubernetes.io/auth-tls-secret: whoami/ca-secret
      nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
      nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "false" # To send the SSL client cert back to the client
```

## Setup the mTLS authentication on the client

### Create the client certificate

```pwsh
# Create a client certificate signed by the CA

## Create a CSR for the client (APIM backend here)
openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=The mTLS Client'

## Sign the CSR with the CA to create the client certificate
openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
```

### Use the client certificate in the client

```pwsh
# Query from client (curl) without the client certificate
curl -v -k https://whoami-ing.ebdemos.info

# => Get a "400 No required SSL certificate was sent" error
# <head><title>400 No required SSL certificate was sent</title></head>

# Query from client (curl) with the client certificate
curl -k -v https://whoami-ing.ebdemos.info/ --key client.key --cert client.crt
```

## Use APIM

1. Add the client mTLS certificate to APIM:

    APIM can manage Certificates directly in 2 ways:

      1. Reference a certificate in Key Vault

      2. Add a custom certificate directly in API Management:

        - Generate the PFX for the client cert with a password

      ```bash
      openssl pkcs12 -export -out client.pfx -inkey client.key -in client.crt` # Requires to create a password
      ```

        - import the PFX file and enter the password in APIM

2. Set the APIM backend to use the client certificate

3. Disable the SSL verification (as we use a self-signed certificate)


## Articles

[Mutual TLS Over a Kubernetes Nginx Ingress Controller](https://earthly.dev/blog/mutual-tls-kubernetes-nginx-ingress-controller/)
