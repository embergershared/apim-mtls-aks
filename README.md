# Enable mTLS authentication between APIM and AKS Ingress



## Steps

### Create Self-signed certificate

```pwsh
# Create a self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt -subj "/CN=whoami-ing.ebdemos.info/O=apim-mtls-aks repo"

# Create the Kubernetes TLS secret
kubectl create secret **tls** self-tls --key server.key --cert server.crt -n whoami
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

### Add a CA in the loop

```pwsh
# Step 1: Create a CA (Certification Authority) self-signed certificate
# Create a CA (Certification Authority) self-signed certificate
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj '/CN=The Cert Authority'

# Create the Kubernetes Generic secret for the CA that will be used by the Ingress to authenticate the client
kubectl create secret **generic** ca-secret --from-file=ca.crt=ca.crt -n whoami
```

### Enable mTLS in the Ingress

```yaml
# Add these 4 annotations to the Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/auth-tls-secret: whoami/ca-secret
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
```

### Setup the mTLS authentication on the client

```pwsh
# Create a client certificate signed by the CA

## Create a CSR for the client (APIM backend here)
openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=The mTLS Client'

## Sign the CSR with the CA to create the certificate
openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
```

```pwsh
# Query from client (curl) without the client certificate
curl -v -k https://whoami-ing.ebdemos.info

# => Get a "400 No required SSL certificate was sent" error
# <head><title>400 No required SSL certificate was sent</title></head>

# Query from client (curl) with the client certificate
curl -k -v https://whoami-ing.ebdemos.info/ --key client.key --cert client.crt
```

### Use APIM

APIM can manage directly:

- a Key Vault certificate:
  - Create a secret in the Key Vault
  - Create a certificate in APIM
  - Import the certificate from the Key Vault

- a custom certificate:
  - Generate the PFX for the client cert with a password
`openssl pkcs12 -export -out client.pfx -inkey client.key -in client.crt` # apim
  - import the PFX file and enter the password in APIM


Set the APIM backend to use the client certificate



### Use the wildcard TLS certificate

```pwsh
$CertFolder = "D:\Dropbox\0.CertsTLS\ebdemos.info\extended_wildcard_ebdemos_info"
$CaCertPath = "$CertFolder\DigiCertCA.crt"
$TlsCertPath = "$CertFolder\extended_wildcard_ebdemos_info.crt"
$TlsPrivKeyPath = "$CertFolder\extended_wildcard_ebdemos_info.privkey"

# Create the Kubernetes secret for the wildcard certificate
kubectl create secret tls wildcard-ebdemos-info --cert="$TlsCertPath" --key="$TlsPrivKeyPath"
```

Change the Ingress to use the wildcard certificate

```yaml
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  tls:
  - hosts:
      - whoami-ing.ebdemos.info
    secretName: wildcard-ebdemos-info
    # secretName: self-tls
```





## Articles

[Mutual TLS Over a Kubernetes Nginx Ingress Controller](https://earthly.dev/blog/mutual-tls-kubernetes-nginx-ingress-controller/)
