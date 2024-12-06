# Step-CA - too many intermediates for path length constraint
## MKCERT - create certificates
```bash
curl -JLO https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64
chmod +x mkcert-v*-linux-amd64
ln -s mkcert-v*-linux-amd64 mkcert
./mkcert -install
```

## Docker - Pull & prepare Network
```bash
docker compose pull
```

## Preconfigure `step-ca` for our `mkcert` certificate
```bash
mkdir -p ./step-ca ./step-ca/secrets

# copy Root Key & Certificate
cp "$(./mkcert -CAROOT)/rootCA.pem" ./step-ca/
cp "$(./mkcert -CAROOT)/rootCA-key.pem" ./step-ca/
echo "123456" > ./step-ca/secrets/password

# set permissions - 1000 is the UID/GID used by the step-ca image
sudo chown -R 1000:1000 ./step-ca

# seed the step-ca container
docker compose run step-ca \
  step ca init \
  --root "/home/step/rootCA.pem" \
  --key "/home/step/rootCA-key.pem" \
  --deployment-type=standalone \
  --name "mkcert development CA" \
  --provisioner "admin" \
  --dns "localhost,step-ca" \
  --address ":443" \
  --password-file="/home/step/secrets/password"
docker compose run step-ca \
  step ca provisioner add acme --type ACME
sudo chmod +r step-ca/certs/root_ca.crt
```

# Errors
## Traefik
```plaintext
reverse_proxy          | time="2024-12-06T16:18:40Z" level=error msg="Unable to obtain ACME certificate for domains \"whoami.host.localmachine\": cannot get ACME client get directory at 'https://step-ca:443/acme/acme/directory': Get \"https://step-ca:443/acme/acme/directory\": tls: failed to verify certificate: x509: too many intermediates for path length constraint" providerName=stepCaResolver.acme routerName=rWhoamiSubdomain@docker rule="Host(`whoami.host.localmachine`)" ACME CA="https://step-ca:443/acme/acme/directory"
step-ca-bug-step-ca-1  | 2024/12/06 16:18:40 /usr/local/go/src/net/http/server.go:3487: http: TLS handshake error from 172.??.??.??:50812: remote error: tls: bad certificate
```

## step-ca
```bash
docker compose exec step-ca step ca health                          
```
Output
```plaintext
client GET https://localhost/health failed: tls: failed to verify certificate: x509: too many intermediates for path length constraint
```