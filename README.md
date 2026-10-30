# Project 1 — Lift-and-Shift: Migrating a Legacy LAMP App to AWS EC2

Rehosting a company's legacy website onto AWS by rebuilding the full LAMP stack (Linux, Apache, MySQL, PHP) on a single EC2 instance and deploying WordPress on top of it.

## Architecture Diagram <img width="2400" height="1256" alt="project1-architecture-linkedin" src="https://github.com/user-attachments/assets/e1331014-1b82-4537-a157-3a53c6c6e085" />


---

## The business problem

The company's site was running on an aging, on-prem physical server: hard to patch, hard to scale, and a single box that someone had to physically babysit. The goal was the fastest path to the cloud with the least change to the application: a classic **lift-and-shift (rehost)** migration. No re-architecture yet. Get it running on AWS first, then modernize from a stable base.

## Architecture

| Layer | Choice | Notes |
|---|---|---|
| Region / AZ | `us-east-1` / `us-east-1a` | Single AZ |
| Network | Default VPC, public subnet | Public IPv4 assigned to the instance |
| Firewall | Security group | `:80` open to the internet, `:22` restricted to my IP |
| Compute | EC2 `t3.micro`, Ubuntu 24.04 | Hosts every tier |
| Web server | Apache 2 | Listens on port 80 |
| Runtime | PHP 8.3 (`libapache2-mod-php`, `php-mysql`) | Apache hands `.php` requests to the PHP module |
| Database | MySQL 8.0 | Bound to `localhost:3306`, never exposed publicly |
| App | WordPress (latest) | Files in `/var/www/html`, owned by `www-data` |

**Request flow:** Browser → HTTP :80 → Apache → PHP → MySQL (localhost) → rendered page back to the browser.

> ⚠️ **Known limitation, by design:** everything runs on one instance in one AZ: no load balancer, no failover. If the instance or the AZ goes down, the site goes down. That's the trade-off of a pure lift-and-shift, and it's what Project 2 addresses.

---

## Build steps

### 1. Launch and connect to the EC2 instance<img width="889" height="738" alt="EC2 instance in the Running state" src="https://github.com/user-attachments/assets/eef3c9d4-2110-44b5-bacf-4318bb066b14" />

Launched an Ubuntu 24.04 `t3.micro` with a security group allowing HTTP from anywhere and SSH from my IP only.

```bash
ssh -i "your-key.pem" ubuntu@<EC2_PUBLIC_DNS>
```

### 2. Patch the OS
```bash
sudo apt update && sudo apt upgrade -y
```
<img width="974" height="614" alt="Screenshot 2026-10-29 at 8 52 36 PM" src="https://github.com/user-attachments/assets/e62bce8e-bd2d-48de-a897-b8b39b5482e7" />

### 3. Install the LAMP stack
```bash
sudo apt install apache2 -y
sudo apt install php libapache2-mod-php php-mysql -y
sudo apt install mysql-server -y
```

<img width="567" height="74" alt="php installed" src="https://github.com/user-attachments/assets/34f19470-e468-4484-b6aa-c4047fe95cec" />
<img width="504" height="32" alt="mySQL installed" src="https://github.com/user-attachments/assets/2029832f-70a1-4b43-bf1f-1ed44b4feed2" />
<img width="389" height="40" alt="apache installed" src="https://github.com/user-attachments/assets/4d4722e0-9406-4704-8359-1bf0714417bb" />


### 4. Start services and enable them on boot
```bash
sudo systemctl start apache2 mysql
sudo systemctl enable apache2 mysql

# verify
sudo systemctl status apache2
sudo systemctl status mysql
```
<img width="983" height="586" alt="start and enable" src="https://github.com/user-attachments/assets/53b9b66e-f446-42c1-adbb-ad72a8ff6ac9" />

### 5. Harden MySQL
```bash
sudo mysql_secure_installation
```
This sets a root password policy, removes anonymous users and the test database, and disables remote root login.
<img width="540" height="567" alt="mysql_secure_installaton" src="https://github.com/user-attachments/assets/205cc67f-920a-4f92-8e2c-c4943b9dc23d" />


### 6. Create the WordPress database and a least-privilege user
```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY '<STRONG_PASSWORD>';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
WordPress gets its own user scoped to its own database and to `localhost` only. It never uses root.

### 7. Deploy WordPress
```bash
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
sudo mv wordpress/* /var/www/html/
sudo rm /var/www/html/index.html        # remove Apache's default "It works!" page

sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### 8. Point WordPress at the database
```bash
sudo mv /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php
```
```php
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wordpress_user');
define('DB_PASSWORD', '<STRONG_PASSWORD>');
```
<img width="843" height="751" alt="wp-config" src="https://github.com/user-attachments/assets/29f37c67-9e23-4fa1-ac45-2ad532f68160" />



### 9. Restart Apache and finish setup in the browser
```bash
sudo systemctl restart apache2
```
Browse to `http://<EC2_PUBLIC_IP>` → the WordPress setup wizard loads (language, site title, admin account). At that point all of the infrastructure work is done.
<img width="881" height="922" alt="wordpress launch" src="https://github.com/user-attachments/assets/9017d189-f2f0-4846-9234-6acd39cc2d52" />

---

## Result

WordPress running on AWS, served by Apache, backed by a local MySQL database, with services that come back up automatically after a reboot. The legacy server is replaced with infrastructure that can be patched, snapshotted, and resized from the AWS console.

## What I learned

- **How the LAMP pieces actually hand off to each other.** Apache doesn't "run" WordPress. It passes `.php` files to the PHP module, and PHP talks to MySQL over `localhost`. Knowing that chain makes troubleshooting a blank page or a "database connection error" much faster.
- **File ownership matters as much as file location.** WordPress files must be owned by `www-data` (Apache's user), or uploads, plugin installs and updates fail.
- **Least privilege starts at the database.** A dedicated MySQL user scoped to one database and to `localhost` limits the blast radius if the app is ever compromised.

## What I'd do differently next time

- **Separate the database tier.** Move MySQL to **Amazon RDS** (Multi-AZ) so the database gets managed backups and failover and no longer shares a box with the web server.
- **Remove the single point of failure.** Put the web tier behind an **Application Load Balancer** with an **Auto Scaling Group** across at least two AZs.
- **Add HTTPS.** Terminate TLS with an ACM certificate on the ALB and stop serving over plain HTTP.
- **Get secrets out of `wp-config.php`.** Pull DB credentials from **AWS Secrets Manager** or SSM Parameter Store instead of hardcoding them.
- **Drop SSH entirely.** Use **SSM Session Manager** and close port 22.
- **Tighten permissions.** Use `755` for directories and `644` for files, and lock `wp-config.php` down to `640`, rather than a blanket `chmod -R 755`.
- **Make it repeatable.** Codify the whole build in **Terraform** with a user-data/cloud-init script so it can be torn down and rebuilt in minutes. *(In progress: a Terraform rebuild of this project.)*

---

## Tech stack

`AWS EC2` · `VPC` · `Security Groups` · `Ubuntu 24.04` · `Apache` · `PHP 8.3` · `MySQL 8.0` · `WordPress` · `Bash`

## Author

**Somone Letman**, Cloud / DevOps Engineer - GitHub: [@SomoneL](https://github.com/SomoneL)
