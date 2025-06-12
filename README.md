Here's a step-by-step guide to generating your private key and Certificate Signing Request (CSR), installing your commercial SSL certificate, configuring Nginx, and verifying the setup on an Ubuntu machine. This guide is applicable to most third-party SSL certificates issued by a Certificate Authority (CA).Important Note: Throughout this guide, remember to replace example_domain.com with your actual domain name.1. SSH into the ServerFirst, you need to connect to your Ubuntu server using SSH. Replace <Lightsail-IP> with the actual IP address of your Amazon Lightsail instance.ssh ubuntu@<Lightsail-IP>
2. Generate a 2048-bit RSA Private KeyNext, you'll generate a 2048-bit RSA private key. It's good practice to store this key in a secure location like /etc/ssl/private.Create the directory (if it doesn't exist):sudo mkdir -p /etc/ssl/private
Generate the private key:sudo openssl genrsa -out /etc/ssl/private/example_domain.key 2048
sudo openssl genrsa: This command generates an RSA private key.-out /etc/ssl/private/example_domain.key: Specifies the output path and filename for your private key.2048: Defines the key size in bits, which is 2048 for strong encryption.3. Generate the Certificate Signing Request (CSR)Finally, you'll generate the CSR using the private key you just created. This CSR will contain information about your domain and organization, which you'll submit to your Certificate Authority (CA) when you purchase your SSL certificate.sudo openssl req -new -key /etc/ssl/private/example_domain.key \
    -out /tmp/example_domain.csr
sudo openssl req -new: This command initiates the process for creating a new CSR.-key /etc/ssl/private/example_domain.key: Specifies the private key file that will be used to sign the CSR.-out /tmp/example_domain.csr: Defines the output path and filename for your CSR. We're placing it in /tmp temporarily.You will be prompted to enter several details. Make sure to provide accurate information, especially for the "Common Name (e.g. server FQDN or YOUR name)". This should be your exact domain name (e.g., example_domain.com or www.example_domain.com).4. Combine Your Certificate FilesAfter obtaining your SSL certificate from your Certificate Authority (CA), you will typically receive your primary certificate (e.g., example_domain.crt) and one or more intermediate and/or root CA certificate files (which might be bundled into a single file like ca-bundle.crt or provided separately). You need to combine these into a single "fullchain" file for Nginx.Important: The exact filenames and the order of concatenation may vary depending on your CA. Always refer to your CA's specific instructions. Generally, the order is: your domain's certificate first, followed by any intermediate certificates (in the correct order), and finally the root certificate.Transfer your downloaded certificate files to your server. You can upload them to your /tmp directory or directly to /etc/ssl/certs/. For the command below, we'll assume they are in /tmp.Combine the certificates (example for common CA bundle format):sudo cat /tmp/example_domain.crt /tmp/example_domain.ca-bundle \
    | sudo tee /etc/ssl/certs/example_domain.fullchain.crt
This command concatenates your domain's certificate (example_domain.crt) and the CA bundle (example_domain.ca-bundle) into a new file named example_domain.fullchain.crt and places it in /etc/ssl/certs/. The tee command is used because direct redirection with > might not work with sudo.If your CA provides separate intermediate certificates (e.g., intermediate1.crt, intermediate2.crt), you would concatenate them in the correct order, typically from leaf to root:sudo cat /tmp/example_domain.crt /tmp/intermediate1.crt /tmp/intermediate2.crt /tmp/root.crt \
    | sudo tee /etc/ssl/certs/example_domain.fullchain.crt
5. Configure Nginx Server BlocksNow, you need to configure your Nginx server to use the new SSL certificate. You'll set up two server blocks: one for HTTPS traffic on port 443 and one for HTTP traffic on port 80 to redirect to HTTPS.Open your Nginx configuration file. This is usually located at /etc/nginx/sites-available/example_domain.com (replace example_domain.com with your actual domain name).sudo nano /etc/nginx/sites-available/example_domain.com
Add or modify the following server blocks:server {
    listen 443 ssl http2;
    server_name example_domain.com www.example_domain.com; # Replace with your actual domain

    ssl_certificate /etc/ssl/certs/example_domain.fullchain.crt; # Path to your combined certificate
    ssl_certificate_key /etc/ssl/private/example_domain.key;     # Path to your private key
    # ssl_trusted_certificate /etc/ssl/certs/example_domain.ca-bundle; # Optional: For OCSP Stapling, point to your CA bundle if separate from fullchain.crt

    # Recommended SSL/TLS settings for security
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:!DES-CBC3-SHA:!RC4:!aNULL:!eNULL:!MD5:!DSS:!SRP:!CAMELLIA';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Optional: OCSP Stapling (improves performance and security)
    # ssl_stapling on;
    # ssl_stapling_verify on;
    # resolver 8.8.8.8 8.8.4.4 valid=300s; # Google Public DNS, adjust if you have a preferred DNS resolver
    # resolver_timeout 5s;

    # Your normal root / proxy_pass etc. for your website content
    # Example:
    # root /var/www/example_domain.com/html;
    # index index.html;

    # location / {
    #     try_files $uri $uri/ =404;
    # }
}

server {
    listen 80;
    server_name example_domain.com www.example_domain.com; # Replace with your actual domain
    return 301 https://$host$request_uri;
}
listen 443 ssl http2;: Configures Nginx to listen on port 443 for HTTPS traffic, enabling HTTP/2 for better performance.ssl_certificate: Points to the fullchain.crt file you created.ssl_certificate_key: Points to your private key.ssl_trusted_certificate: This directive is primarily used for OCSP Stapling and should point to the CA bundle file that contains the intermediate and root certificates. If your fullchain.crt already contains the entire chain and you're not using OCSP stapling, this directive can be omitted.listen 80;: Configures Nginx to listen on port 80 for HTTP traffic.return 301 https://$host$request_uri;: A permanent redirect for all HTTP requests to their HTTPS equivalent.6. Test Nginx ConfigurationBefore reloading Nginx, always test your configuration for syntax errors.sudo nginx -t
You should see output similar to this, indicating that the test is successful:nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
7. Reload NginxIf the configuration test is successful, reload Nginx to apply the changes. This will apply the new configuration without dropping active connections.sudo systemctl reload nginx
8. Verify SSL InstallationFinally, verify that your SSL certificate has been installed correctly and is being served by Nginx.Using openssl s_client (from your local machine or server):openssl s_client -connect example_domain.com:443 -servername example_domain.com
Replace example_domain.com with your actual domain.In the output, look for:Verify return code: 0 (ok): This indicates that the certificate chain is complete and trusted.Details about your certificate, including its issuer (your CA), subject (your domain), and validity dates.Using a web browser:Open your web browser and navigate to https://example_domain.com. You should see a padlock icon in the address bar, indicating a secure connection. Click on the padlock to view certificate details and ensure it's issued for your domain by your Certificate Authority.Use online SSL checkers:For a more in-depth analysis, use online tools like SSL Labs' SSL Server Test (https://www.ssllabs.com/ssltest/) or other generic SSL checkers.
