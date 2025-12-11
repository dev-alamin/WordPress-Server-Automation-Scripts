# 🚀 WordPress Server Automation Scripts (LEMP Stack)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Automated installation of a complete **LEMP stack** (Linux, Nginx, MariaDB, PHP) and **WordPress site provisioning**, optimized for performance, repeatability, and fast deployments on **Ubuntu**.

This repository contains two essential bash scripts:

1.  `install_stack`: Installs and configures Nginx, PHP 8.3, MariaDB, WP-CLI, firewall, and timezone setup.
2.  `create_wp`: Creates a fully configured WordPress site with database creation, Nginx virtual host, WP installation, permissions, and optional SSL.

---

## 1. `install_stack` – LEMP Stack Installer

This script provisions a modern, secure, and WordPress-ready server on Ubuntu.

### ✨ Features

* Installs **Nginx**.
* Installs **PHP 8.3** (using Ondrej PPA) + essential extensions.
* Installs **MariaDB 10.x** with hardened defaults (Unix-socket authentication for root).
* Installs **WP-CLI**.
* Configures **UFW firewall** (OpenSSH + Nginx Full profile).
* Configures server timezone.
* Includes error-handling for critical steps.

### ⚙️ Installation Details

| Component | Status | Details |
| :--- | :--- | :--- |
| **Nginx** | Enabled + Running | |
| **PHP 8.3 FPM** | Installed + Running | Extensions: `mysql`, `xml`, `gd`, `curl`, `mbstring`, `zip`, `intl` |
| **MariaDB** | Secured | Unix-socket authentication for root |
| **WP-CLI** | Installed | |
| **UFW Firewall** | Configured | Allows SSH and Nginx traffic |

### 🧭 Usage

1.  Make the script executable:
    ```bash
    chmod +x install_stack
    ```
2.  Run the script with `sudo`:
    ```bash
    sudo ./install_stack
    ```
    *The script will prompt you for your server timezone (e.g., `Asia/Dhaka`).*

---

## 2. `create_wp` – WordPress Site Creator

This script utilizes WP-CLI to fully automate a new WordPress installation, including database creation and Nginx configuration.

### ⚠️ Requirements

* You **must** run `install_stack` first.
* The domain's **A record** must point to your server's IP address.

### 🧭 Usage

1.  Make the script executable:
    ```bash
    chmod +x create_wp
    ```
2.  Run the script with the required arguments:
    ```bash
    sudo ./create_wp <domain> <title> <admin_user> <admin_email> <db_name> <db_user> [ssl_choice]
    ```

### 📋 Argument Details

| Argument | Description | Example |
| :--- | :--- | :--- |
| `<domain>` | Domain name (without http/https) | `example.com` |
| `<title>` | WordPress site title | `"My Blog"` |
| `<admin_user>` | WP admin username | `admin` |
| `<admin_email>` | WP admin email | `admin@example.com` |
| `<db_name>` | Database name | `wpdb1` |
| `<db_user>` | Database username | `wpuser1` |
| `[ssl_choice]` | Optional: `y` or `n` (for Certbot SSL) | `y` |

### Example Run

```bash
sudo ./create_wp play.example.com "Play Site" admin admin@example.com playdb playuser y
```

### 🛠️ What the Script Does

* Creates database and DB user with a generated 16-character random password.
* Downloads the latest WordPress core.
* Generates `wp-config.php`.
* Creates Nginx virtual host configuration.
* Enables the site and reloads Nginx.
* Installs WordPress using `wp-cli core install`.
* Fixes file permissions for security.
* Optionally installs SSL using **Certbot** (Let's Encrypt).

### 🔑 Output

The script will print the following critical information. **Copy and store it safely.**

* Site URL
* Admin login URL
* Database credentials
* Admin password (auto-generated)

### 📁 Directory Locations

| Component | Path |
| :--- | :--- |
| **Website files** | `/var/www/<domain>` |
| **Nginx config** | `/etc/nginx/sites-available/<domain>.conf` |
| **PHP socket** | `/run/php/php8.3-fpm.sock` |

### 🔒 SSL (Optional)

If you pass `y` as the last argument, the following command will be run to automatically install and configure SSL:

```bash
certbot --nginx -d domain.com -d [www.domain.com](https://www.domain.com)
```

## 💡 Troubleshooting

| Problem | Command to Run | Resolution |
| :--- | :--- | :--- |
| **Nginx test fails** | `sudo nginx -t` | Fix the configuration, then run `sudo systemctl reload nginx` |
| **WP-CLI command not found** | `wp --info` | If missing, run: `sudo ln -s /usr/local/bin/wp /usr/bin/wp` |
| **Database connection issues** | `sudo mysql` | Ensure you can log in to the MariaDB console |
| **Permission issues** | `sudo chown -R www-data:www-data /var/www/<domain>` | Reset file ownership to the Nginx/PHP user |

---

## 📝 Notes & Best Practices

* Always run scripts using `sudo`.
* DNS **A record** must be active before attempting to enable SSL.
* If you modify the PHP version in one script, ensure you also update the other to maintain compatibility.
