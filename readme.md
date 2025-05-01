# Nginx with Let's Encrypt SSL using Certbot and Docker Compose

This setup uses Docker Compose to automate the process of obtaining and renewing Let's Encrypt SSL certificates using Certbot's webroot plugin with a temporary Nginx server.

## Prerequisites

*   Docker: [Install Docker](https://docs.docker.com/get-docker/)
*   Docker Compose: [Install Docker Compose](https://docs.docker.com/compose/install/)
*   A registered domain name pointing to the public IP address of the host where you will run this setup.
*   Port 80 must be open and accessible from the internet.

## Setup

1.  **Clone the repository or create the files:**
    Make sure you have the necessary files in your project directory.

2.  **Create Environment File:**
    Create a `.env` file in the same directory as `certbot-compose.yml` and define your domain:
    ```env
    CERTBOT_DOMAIN=yourdomain.com
    ```
    Replace `yourdomain.com` with your actual domain name.

3.  **Create Required Directories:**
    Create the directories needed for volumes:
    ```bash
    mkdir ssl static
    ```
    *   `ssl`: This directory will be used by Certbot to store the obtained certificates, account information, and renewal configurations. It maps to `/etc/letsencrypt` inside the Certbot container.
    *   `static`: This directory is used by Nginx to serve the ACME challenge files required by Certbot for domain validation. It maps to `/usr/share/nginx/html/` inside both containers.

4.  **Review Configuration:**

    *   **./certbot-ssl/certbot-compose.yml**: Defines the `certbot` and `nginx-certbot` services.
        ```yaml
        services:
          certbot:
            image: certbot/certbot
            container_name: certbotcontainer
            environment:
              - CERTBOT_DOMAIN=${CERTBOT_DOMAIN}
            volumes:
              - ./ssl:/etc/letsencrypt
              - ./static:/usr/share/nginx/html/
            command: certonly --webroot --webroot-path=/usr/share/nginx/html -d ${CERTBOT_DOMAIN} --agree-tos --email sunilkumarkumawat371@gmail.com --no-eff-email --keep-until-expiring --force-renewal --keep
            depends_on:
              - nginx-certbot


          nginx-certbot:
            image: nginx:alpine
            container_name: nginx-certbot
            ports:
              - "80:80"
              - "443:443" # Port 443 is included but not used by the default nginx.conf for challenge
            volumes:
              - ./static:/usr/share/nginx/html/
              - ./nginx.conf:/etc/nginx/conf.d/default.conf
        ```
    *   **./certbot-ssl/nginx.conf**: A minimal Nginx configuration specifically for handling the Let's Encrypt ACME challenge requests on port 80.
        ```nginx
        server {
            listen 80 default_server;

            location ~ /.well-known/acme-challenge/ {
                alias /usr/share/nginx/html/;
                try_files $uri =404;
            }
        }
        ```

5.  **Run Certbot:**
    Start the services. Docker Compose will start the Nginx container first, and then the Certbot container will run its command to obtain the certificate.
    ```bash
    CERTBOT_DOMAIN=nginx.myeasydevops.xyz docker-compose -f ./certbot-ssl/certbot-compose.yml run certbot
    ```
    If successful, Certbot will place the certificates in the `./certbot-ssl/ssl/live/yourdomain.com/` directory (relative to your host machine). The Certbot container will then stop, but the Nginx container might keep running depending on the Certbot command's exit behavior. You can stop it manually with `Ctrl+C` or `docker compose -f ./certbot-ssl/certbot-compose.yml down`.

## Using the Certificates

Once you have the certificates in `./certbot-ssl/ssl`, you can configure your main web server (whether it's another Nginx container or a host-installed Nginx) to use them for HTTPS. Remember to also set up automatic renewal (Certbot usually handles this via a cron job or systemd timer, but in a container setup, you might need a separate process or cron job within a running container, or periodically run `docker compose run --rm certbot renew`).

The `./certbot-ssl/ssl` directory contains:
*   `accounts`: Your Let's Encrypt account information.
*   `archive`: All previous versions of your certificates.
*   `live`: Symlinks to the current valid certificates (`cert.pem`, `chain.pem`, `fullchain.pem`, `privkey.pem`). **Always point your webserver configuration to the files in this directory.**
*   `renewal`: Configuration files for certificate renewal.