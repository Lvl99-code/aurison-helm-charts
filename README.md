helm template aurison-backend ./aurison-backend --set aurisonBackendPacs.enabled=true --set mtls.enabled=true --set-file mtls.caCertificate=C:/Users/inventor/Downloads/tmp/ca-certificate.pem --set-file mtls.caPrivateKey=C:/Users/inventor/Downloads/tmp/ca-private-key.pem


Modify chart version

Run this
helm package ./aurison-backend 

 helm repo index .  