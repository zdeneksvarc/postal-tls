## About

**❗ This repository runs with the Postal [version 2.1.2](https://github.com/postalserver/postal/releases). It should be fine for version 2.1.x maybe 2.x**

The Postal delivery platform does not offer encrypted services in the basic installation. That is certificate management, HTTPS interface, mail submission via oportunistic TLS (StartTLS) and implicit TLS. Deploying this repository via Docker Compose will provide these services.

Components of the deployment:

- [Caddy 2](https://hub.docker.com/_/caddy) web server used for automated ACME certificate management and HTTPS reverse proxy. [Let's Encrypt](https://letsencrypt.org) HTTP-01 challenge type is preconfigured.
- [Stunnel](https://hub.docker.com/r/dweomer/stunnel/) for implicit TLS termination on port 465. This is the preferred method of mail submission, see [RFC8314](https://www.rfc-editor.org/rfc/rfc8314)
- [Socat](https://hub.docker.com/r/alpine/socat/) for mirroring ports 587 and 25, which is running StartTLS directly from Postal.
- Restarter is [Docker outside of Docker](https://hub.docker.com/_/docker) service which restarts Postal SMTP and Stunnel once every ten days to reload certificate.

## Pre-requisites

Installed Postal according to the [official instructions](https://docs.postalserver.io/install/prerequisites) with working web interface on port 80 and SMTP on port 25. No HTTPS reverse proxy, no certificates management.

## Installation procedure

1. Edit configuration file `/opt/postal/config/postal.yml` adding [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) of the Postal host (for example `postal.example.com`) instead of `<HOST>`

```
smtp_server:
  port: 25
  tls_enabled: true
  tls_certificate_path: /caddy-data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/<HOST>/<HOST>.crt
  tls_private_key_path: /caddy-data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/<HOST>/<HOST>.key
```

2. Working directory will be `/opt/caddy-tls`, so do `cd /opt`, fetch the install repository `sudo git clone https://github.com/zdeneksvarc/postal-tls.git` and set working directory `cd /opt/postal-tls`
3. Set the FQDN of the Postal host in the files `.env` and `Caddyfile` and also set your e-mail address in the `Caddyfile` instead of `your@email.com` which is used to identify the owner of the certificate.
4. Still in the `/opt/postal-tls` now run `docker compose up https` and wait a while until Caddy gets the certificates, then exit via CTRL-C
5. Create a symlink `sudo ln -s /opt/postal-tls/caddy-data /opt/postal/caddy-data` and set the permissions of directory `sudo find /opt/postal-tls/caddy-data -type d -exec chmod 755 {} +` and files `sudo find /opt/postal-tls/caddy-data -type f -exec chmod 644 {} +`
6. Add the volume with certificates `/opt/postal/caddy-data:/caddy-data` to the Postal smtp service in `/opt/postal/install/docker-compose.yml`.
7. This step is not necessary, but for the sake of clarity we can delete the Caddyfile offered by Postal `sudo rm /opt/postal/config/Caddyfile` and link a working one `sudo ln -s /opt/postal-tls/Caddyfile /opt/postal/config/Caddyfile` and remove .git directory `sudo rm -r /opt/postal-tls/.git` and .gitignore file `sudo rm -r /opt/postal-tls/.gitignore`
8. Stop Postal via `postal stop` and start Postal via `postal start`, which will now start with StartTLS support on port 25.
9. Still in the working directory `/opt/postal-tls` start the postal-tls Docker Compose project via `docker compose up -d` and you're done 🎉

## Services testing

- For StartTLS on port 587 or 25 you can try `openssl s_client -starttls smtp -connect <host>:<port>`
- For implicit TLS on port 465 you can try `openssl s_client -connect <host>:465`
- The Postal SMTP container log can be accessed via `docker compose -p postal logs -f | grep smtp_1`
- You can use [Swaks](https://jetmore.org/john/code/swaks/) to send test emails and [testssl.sh](https://testssl.sh) for TLS checks.

## A few notes

- There is a small glitch because the implicit TLS on port 456 also advertises `250-STARTTLS`. This is an obvious behavior, since the decrypted communication from port 465 goes to port 25. There are no known issues that this should cause in production.
- Follows the recommendations of [RFC8314](https://www.rfc-editor.org/rfc/rfc8314) use implicit TLS on port 465 instead of StartTLS on port 587 due to the possibility of strip SSL attack.
- As you can see, the service on ports 25 and 587 is identical because it is mirroring.
- Postal does not support [ECC](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) certificates, but these are default for Caddy v2. Therefore, an explicit `{ key_type rsa4096 }` directive is used in the Caddyfile to force Caddy to use [RSA](https://simple.wikipedia.org/wiki/RSA_algorithm) certificates.
- Postal internal services also run as Docker Compose project, but are handled by the `/usr/bin/postal` wrapper.
- It's okay to be cautious about the trustworthiness of public docker images. If you consider it better, you can build the image of each of the three services used yourself.
