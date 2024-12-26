# SSO-Trino
OAuth 2.0 authentication for Trino

# Trino

Following DOC https://trino.io/docs/current/security/oauth2.html#security-oauth2--page-root

1. Change the svc config for trino coordinator
```
service:
  annotations: 
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-shared-securityrule: "true"
  type: LoadBalancer
  port: 8080
```
2. Get the ip addr for the loadbalancer.
```
k get svc -n trino
```
3. Map the LB endpoint to a domain. It would be an **A record**
![_AF547E40-2F68-4CCD-805B-3C23BFD628EB_](/uploads/38fc7dd87dc7d59549b1723261a9c3f9/_AF547E40-2F68-4CCD-805B-3C23BFD628EB_.png){width=1017 height=114}
4. Open trino on http://<domain>:8080/


### Create a self signed cert for trino
Followed this doc: https://trino.io/docs/current/security/tls.html#secure-trino-directly
```
$ openssl req -newkey rsa:2048 -nodes -keyout private_key.pem -x509 -days 3650 -out public_certificate.pem

```
It will prompt you to put some details. Please mark that common name must be same as your fqdn
![_13CB451A-E2C6-4F5A-83F6-3322FD1AF055_](/uploads/9f8b8f5913c7f3436b3b65d4b96d8570/_13CB451A-E2C6-4F5A-83F6-3322FD1AF055_.png){width=638 height=126}
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:IN
State or Province Name (full name) [Some-State]:Maharashtra
Locality Name (eg, city) []:Mumbai
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dronapay Pvt Ltd
Organizational Unit Name (eg, section) []:Operations
Common Name (e.g. server FQDN or YOUR name) []:trino.uditsharma.online
Email Address []:udit@dronapay.com
```
Now combine the private key and the certificate into one pem file using this  command
```
$ cat private_key.pem public_certificate.pem > clustercoord.pem
```

## Change the trino applications helmchart 

1. Add the clusterprod.pem content as configmap to in trino-values.yaml
```
coordinator:
  additionalVolumeMounts: 
    clustercoord.pem: |
      -----BEGIN PRIVATE KEY-----
      MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQC8eWGu9IrAlmrp
      ...
      -----END PRIVATE KEY-----
      -----BEGIN CERTIFICATE-----
      MIIEMzCCAxugAwIBAgIUX/knWPGBN6x8l+2WruiSkPOnPjEwDQYJKoZIhvcNAQEL
      ...
      -----END CERTIFICATE-----
```

2. This configmap will be automatically mounted onto path **/etc/trino**
3. Make changes in trino-values.yaml file 
```
server:
  coordinatorExtraConfig: |
    http-server.https.enabled=true
    http-server.https.port=8443
    http-server.https.keystore.path=/etc/trino/clustercoord.pem
    http-server.authentication.allow-insecure-over-http=true

#Also add
coordinator:
  additionalExposedPorts: 
    https:
      servicePort: 443
      name: https
      port: 8443
      protocol: TCP

```
4. Open the url on external IP of svc
```
k get svc -n trino
```
![_08D6BE6A-0B3B-47C3-B5A4-D834BA0B78E3_](/uploads/154b0337bf5bb8b848a5651975658c8b/_08D6BE6A-0B3B-47C3-B5A4-D834BA0B78E3_.png){width=744 height=86}
![image](/uploads/2e057774f3f978beca3311d40301bc65/image.png)

5. Now open trino using domain.
**https://trino.uditsharma.online**
![image](/uploads/ed2d76bfd0566f00ea0ce7bee479c696/image.png)

## Create azure ad application for TRINO
Followed doc: https://trino.io/docs/current/security/oauth2.html#security-oauth2--page-root

- Open azure, EntraID > App registrations > New Registerations
![_D12EE7FB-B3B4-4A41-AAC0-AAF7F2C83AFF_](/uploads/cc9797fe3c4070855cb3bf3353cc445f/_D12EE7FB-B3B4-4A41-AAC0-AAF7F2C83AFF_.png){width=1440 height=661}
- Tap register

- Open trino-sso applications > Authentication > Add `Front-channel logout URL` to https://<domain>/ui/logout/logout.html
- Leave other properties as it is and save
![_00694999-83AE-4571-8851-45B422B8DB4D_](/uploads/48b122e4a57fc0543dc7f603b4c82557/_00694999-83AE-4571-8851-45B422B8DB4D_.png){width=1440 height=631}

- Go to trino-sso applications > Certificates & secrets > Add New Client Secret > 
- Save the Secret Value generated as it will be used in setting up trino coordinator OIDC Config at http-server.authentication.oauth2.client-secret

- Now Open trino applications > Overview > Copy the Application (client) ID. This will be used at http-server.authentication.oauth2.client-id 
- Change the trino-values.yaml file
```yaml
  workerExtraConfig: |
    # Shared Secret
    internal-communication.shared-secret=meZ7KoBrCAJsJLoZHMES94l9nt71UDxN8I6SFiGKXhGV/kUWa60Ngj7sarps/Fvxd1olBMGJGiimAVY5yFbdDKa1+xFD9nRfb9XVDbL4uPv9AOOnET+o+RnT2VWfQSuuxjQhJZ8fnxErD0wxcftf5x0so5+JEhl8p7xPk33UuBQdbZRuvv+ORmDZDcvplvVmJaerDPztRenFVnO7ohxegRAQjh71LUbjhAXmTtoyGrAO7jyp7o/24BOB/HY4pgpDaiPG+zS+TYPY+nKU2Wta8bdIBWGlOrtf/xYS55/NAPcExWyrnBcGYm7jE1tDbalmebyKhzbemOlOdrUng3qCC36XxNl1PXHSOR8ua50j7y9GwlNKKgc1WcrLRkr7VL5Zk8clZH85EGj8rfpeWqtqAtDPWCOWoa35sf46zfB7omtchOshtTlol4FYBorIXftYSf4G6rvZ7e6S9+ixp4FX1wmMW/yajOesYpCeuCkD56ERxOXvlqKldTwi8mvcYKRZo68WTrErqp6oZR5MGSAwoQCH69bNoCijE8Qmhe60A2eKaVr0IGqq+luqdHs5UFwyTdHU7qaNnPeOFokpDQCBG1YTdvLY9RrZqzI8FhQivw65Jkwwly+BZ0C3WV20862LRmV7GPlSqhgtqD4KVSwM7Ph5ba3eYSY6nbi+qa8a5wo=

  coordinatorExtraConfig: |
    http-server.https.enabled=true
    http-server.https.port=8443
    http-server.https.keystore.path=/etc/trino/clustercoord.pem
    http-server.authentication.allow-insecure-over-http=true

    # Shared Secret
    internal-communication.shared-secret=meZ7KoBrCAJsJLoZHMES94l9nt71UDxN8I6SFiGKXhGV/kUWa60Ngj7sarps/Fvxd1olBMGJGiimAVY5yFbdDKa1+xFD9nRfb9XVDbL4uPv9AOOnET+o+RnT2VWfQSuuxjQhJZ8fnxErD0wxcftf5x0so5+JEhl8p7xPk33UuBQdbZRuvv+ORmDZDcvplvVmJaerDPztRenFVnO7ohxegRAQjh71LUbjhAXmTtoyGrAO7jyp7o/24BOB/HY4pgpDaiPG+zS+TYPY+nKU2Wta8bdIBWGlOrtf/xYS55/NAPcExWyrnBcGYm7jE1tDbalmebyKhzbemOlOdrUng3qCC36XxNl1PXHSOR8ua50j7y9GwlNKKgc1WcrLRkr7VL5Zk8clZH85EGj8rfpeWqtqAtDPWCOWoa35sf46zfB7omtchOshtTlol4FYBorIXftYSf4G6rvZ7e6S9+ixp4FX1wmMW/yajOesYpCeuCkD56ERxOXvlqKldTwi8mvcYKRZo68WTrErqp6oZR5MGSAwoQCH69bNoCijE8Qmhe60A2eKaVr0IGqq+luqdHs5UFwyTdHU7qaNnPeOFokpDQCBG1YTdvLY9RrZqzI8FhQivw65Jkwwly+BZ0C3WV20862LRmV7GPlSqhgtqD4KVSwM7Ph5ba3eYSY6nbi+qa8a5wo=
    
    # OIDC Config
    http-server.authentication.type=oauth2
    http-server.authentication.oauth2.issuer=https://login.microsoftonline.com/d1327173-77da-4002-9636-b1aac62ef787/v2.0
    http-server.authentication.oauth2.client-id=78d9369e-xxxxxxxxxxxxxxxxxx
    http-server.authentication.oauth2.client-secret=GlO8Qxxxxxxxxxxxxxxxxxxxxx
    web-ui.authentication.type=oauth2
```

### Deploy Trino

- Use following commands to deploy the helm-chart
```
cd trino
helm upgrade -i  . --values trino-values.yaml -n trino --create-namespace
```


