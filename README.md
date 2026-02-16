# SECMON - Modernized

This repository serves as the **Installation Guide and Technical Documentation** for the modernized **SECMON** (Security Monitor) platform. It details the process of adapting the original ("Legacy") project to function on current infrastructure (Kali Linux 2024 / Debian 12 Bookworm), covering the resolution of major dependency incompatibilities and the migration of data collection to the new **NIST NVD API v2.0** standard.

---

## 🚀 Key Features

- **Containerized Architecture:** Fully isolated environment (Apache + Python + SQLite) running on Docker.
- **NIST API v2.0 Integration:** Rewritten data collection engine to support the new JSON format and CVSS v3.1 metrics, replacing the deprecated v1.0 API.
- **Security (HTTPS):** Implementation of SSL/TLS encryption using self-signed certificates.
- **Alerting System:** SMTP integration for email notifications upon detection of critical vulnerabilities.
- **RSS News Feed:** Monitoring of security news sources (Exploit-DB, CSHub) for detecting active malware campaigns.

---

## 🛠️ Installation and Configuration Guide

Follow these steps to deploy a fully functional instance.

### 1. Cloning and Building the Image

Download the repository and build the Docker image. We use --no-cache to ensure the latest system packages are downloaded.

```bash
# Build the local image
sudo docker build . -t secmon:latest --no-cache
```

### 2. Launching the Container

Run the container mapping HTTP (80) and HTTPS (443) ports to the host machine.

```bash
sudo docker run -dit --name secmon_instance -p 80:80 -p 443:443 secmon:latest
```

### 3. Resolving Dependencies (Critical Fix - Debian 12)

Modern systems block the installation of old Python libraries due to OpenSSL risks. For SECMON to function, we force the installation of compatible versions:

```bash
sudo docker exec -it secmon_instance pip3 install --break-system-packages \
requests==2.31.0 urllib3==1.26.15 legacy-cgi
```

> **Note:** Downgrading to `urllib3 v1.26` is essential to avoid the `SSL: UNSAFE_LEGACY_RENEGOTIATION` error.

### 4. Web Server Configuration & Permissions

The default Apache configuration must be replaced to serve the WSGI application correctly.

```bash
# Copy optimized Apache configuration
sudo docker cp secmon_apache.conf secmon_instance:/etc/apache2/sites-available/secmon.conf

# Set permissions (Apache runs as www-data)
sudo docker exec -it secmon_instance chown -R www-data:www-data /var/www/secmon
```

### 5. SSL Configuration and Site Activation

Enable SSL modules and the site, then restart the service. Ensure you have generated `server.crt` and `server.key` locally or via the setup script.

```bash
# Enable site and SSL module
sudo docker exec -it secmon_instance a2ensite secmon.conf
sudo docker exec -it secmon_instance a2enmod ssl

# Restart to apply changes
sudo docker exec -it secmon_instance service apache2 restart
```

### 6. Final Setup (Database & SMTP)

Run the initialization script to create the SQLite database and configure alerts.

```bash
sudo docker exec -it secmon_instance python3 /var/www/secmon/setup.py -tls yes
```

(Follow the on-screen instructions to enter the email address and SMTP server).

---

## 💻 Technical Modifications (Refactoring)

To bring the application up to date, direct interventions in the source code were necessary:

- **NIST API:** The `secmon_lib.py` file was modified to query the endpoint `https://services.nvd.nist.gov/rest/json/cves/2.0`.
- **User-Agent:** Added the `User-Agent: SECMON-Tool` header in HTTP requests to prevent blocking by the NIST firewall (403 Forbidden Error).
- **JSON Parsing:** The data structure was adapted to correctly extract the CVSS score and attack vectors from the new v2.0 format.

---

## 🐛 Troubleshooting (Common Issues)

**Error: "Unable to connect" in browser**

The browser attempts to access port 80, but we forced HTTPS.

*Solution:* Explicitly access using the secure protocol: `https://<Your-IP>` (e.g., `https://192.168.254.130`). Accept the security risk for the self-signed certificate.

**Error: Database is empty / "No new CVEs"**

If no data appears after running `python3 secmon.py`:

- Verify that you have added products (e.g., "Windows 10") in Settings -> Product Management.
- Ensure that `cve_updater.py` has successfully run the initial historical population.

---

## 📜 License and Credits

This project is an educational adaptation based on the original work of the alb-uss team. Modifications were made for the purpose of demonstrating legacy infrastructure modernization.
