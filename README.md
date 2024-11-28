
# **Setup Nginx with HTTPS (Cloudflare) and SELinux on CentOS 9**

This guide provides step-by-step instructions for setting up Nginx with HTTPS using Cloudflare Origin CA and configuring SELinux on CentOS 9.

---

## **1. Install Nginx**

1. **Update the system:**
   ```bash
   sudo dnf update -y
   ```

2. **Install Nginx:**
   ```bash
   sudo dnf install nginx -y
   ```

3. **Enable and start Nginx:**
   ```bash
   sudo systemctl enable nginx
   sudo systemctl start nginx
   ```

4. **Open HTTP and HTTPS ports:**
   ```bash
   sudo firewall-cmd --permanent --add-service=http
   sudo firewall-cmd --permanent --add-service=https
   sudo firewall-cmd --reload
   ```

---

## **2. Generate Cloudflare Origin Certificate**

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com/).
2. Navigate to **SSL/TLS > Origin Server**.
3. Click **Create Certificate** and choose:
   - **Key Type**: RSA (default).
   - **Hosts**: Add your domain (e.g., `example.com` and `*.example.com`).
   - **Certificate Validity**: 15 years (recommended).
4. Copy the **certificate** and **private key** provided.

---

## **3. Install the Origin Certificate**

1. Create a directory for SSL certificates:
   ```bash
   sudo mkdir -p /etc/nginx/ssl
   ```

2. Save the certificate:
   ```bash
   sudo nano /etc/nginx/ssl/cloudflare-origin.pem
   ```
   Paste the certificate content and save.

3. Save the private key:
   ```bash
   sudo nano /etc/nginx/ssl/cloudflare-origin.key
   ```
   Paste the private key content and save.

4. Set secure permissions for the files:
   ```bash
   sudo chmod 600 /etc/nginx/ssl/cloudflare-origin.*
   sudo chown root:root /etc/nginx/ssl/cloudflare-origin.*
   ```

---

## **4. Configure Nginx for HTTPS**

1. Edit the Nginx configuration:
   ```bash
   sudo nano /etc/nginx/conf.d/yourdomain.conf
   ```

2. Add the following configuration:
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com www.yourdomain.com;

       # Redirect HTTP to HTTPS
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl http2;
       server_name yourdomain.com www.yourdomain.com;

       # SSL Configuration
       ssl_certificate /etc/nginx/ssl/cloudflare-origin.pem;
       ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

       # SSL Settings
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_ciphers HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers on;

       # Backend Proxy
       location / {
           proxy_pass http://127.0.0.1:8555;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto https;
       }
   }
   ```

3. Test the Nginx configuration:
   ```bash
   sudo nginx -t
   ```

4. Reload Nginx:
   ```bash
   sudo systemctl reload nginx
   ```

---

## **5. Configure Cloudflare**

1. In Cloudflare, navigate to **SSL/TLS > Overview**.
2. Set **SSL/TLS Encryption Mode** to **Full (Strict)**.
3. Go to **DNS** and ensure your domain's A record points to your server's public IP with **Proxy Status** set to **Proxied** (orange cloud).

---

## **6. Enable Nginx SELinux Policy**

1. Check if SELinux is enforcing:
   ```bash
   sestatus
   ```
   Output should indicate:
   ```
   Current mode: enforcing
   ```

2. Enable network connections for Nginx:
   ```bash
   sudo setsebool -P httpd_can_network_connect 1
   ```

3. If further denials occur, capture and allow them:
   ```bash
   sudo ausearch -c 'nginx' --raw | audit2allow -m nginx_local > nginx_local.te
   ```

4. Compile and apply the custom SELinux policy:
   ```bash
   sudo checkmodule -M -m -o nginx_local.mod nginx_local.te
   sudo semodule_package -o nginx_local.pp -m nginx_local.mod
   sudo semodule -i nginx_local.pp
   ```

---

## **7. Verify the Setup**

1. Test the backend connection:
   ```bash
   curl http://127.0.0.1:8555
   ```

2. Test the site via Nginx:
   ```bash
   curl -I https://yourdomain.com
   ```

3. Monitor Nginx logs for errors:
   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```

---

## **8. Optional: Restrict Access to Cloudflare IPs**

1. Add Cloudflare IP ranges to your Nginx configuration:
   ```nginx
   set_real_ip_from 173.245.48.0/20;
   set_real_ip_from 103.21.244.0/22;
   set_real_ip_from 103.22.200.0/22;
   set_real_ip_from 103.31.4.0/22;
   set_real_ip_from 141.101.64.0/18;
   set_real_ip_from 108.162.192.0/18;
   set_real_ip_from 190.93.240.0/20;
   set_real_ip_from 188.114.96.0/20;
   set_real_ip_from 197.234.240.0/22;
   set_real_ip_from 198.41.128.0/17;
   set_real_ip_from 162.158.0.0/15;
   set_real_ip_from 104.16.0.0/13;
   set_real_ip_from 172.64.0.0/13;
   set_real_ip_from 131.0.72.0/22;
   real_ip_header CF-Connecting-IP;
   ```

2. Reload Nginx:
   ```bash
   sudo systemctl reload nginx
   ```

---

## **9. Final Notes**
- Use an online SSL checker like [SSL Labs](https://www.ssllabs.com/) to verify your HTTPS configuration.
- Regularly monitor logs for security or connectivity issues:
  ```bash
  sudo tail -f /var/log/nginx/error.log
  ```
