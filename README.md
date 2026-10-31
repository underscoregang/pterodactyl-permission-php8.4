# Fix Pterodactyl Extension Installation Permissions (PHP 8.4)

## Error

> The panel cannot write to /var/www/pterodactyl. Give the www-data user write access to it, then try again.

This means the extension installer cannot modify the panel directory using its current filesystem permissions. The following steps assume a standard Linux installation at `/var/www/pterodactyl` with the PHP-FPM pool running as `www-data`.

## 1. Confirm the PHP-FPM user

```bash
sudo grep -E '^\s*(user|group)\s*=' /etc/php/8.4/fpm/pool.d/www.conf
ls -ld /var/www/pterodactyl
```

If the PHP-FPM pool uses a different user, substitute that account for `www-data` in the steps below.

## 2. Back up the panel

Before allowing an extension to change panel files, back up your panel directory and database. Use your normal backup procedure and keep the backup outside the web root.

## 3. Grant ownership to the panel service account

For a standard single-user panel installation:

```bash
sudo chown -R www-data:www-data /var/www/pterodactyl
sudo find /var/www/pterodactyl -type d -exec chmod 755 {} +
sudo find /var/www/pterodactyl -type f -exec chmod 644 {} +
sudo chmod -R ug+rwX /var/www/pterodactyl/storage /var/www/pterodactyl/bootstrap/cache
```

These commands grant the owner write access without making the panel world-writable. If your installation uses a separate deployment owner or custom ACLs, adjust permissions to match that setup rather than blindly changing all ownership.

**Do not use `chmod -R 777`**: it gives all local users write access and creates unnecessary security risk.

## 4. Verify write access

```bash
sudo -u www-data test -w /var/www/pterodactyl && echo 'Panel directory is writable' || echo 'Panel directory is NOT writable'
sudo -u www-data touch /var/www/pterodactyl/.write-test
sudo rm /var/www/pterodactyl/.write-test
```

If the test fails, inspect directory ownership, ACLs, and any container or read-only mount restrictions.

## 5. Clear Laravel caches

```bash
cd /var/www/pterodactyl
sudo -u www-data php artisan optimize:clear
```

## 6. Restart PHP 8.4-FPM and web services

```bash
sudo systemctl restart php8.4-fpm
sudo systemctl restart nginx
sudo systemctl restart pteroq
```

If your server uses Apache instead of Nginx, restart Apache instead. `pteroq` is the default Pterodactyl queue worker service name; skip or adjust it if your installation differs.

## 7. Retry the extension installation

Refresh the Pterodactyl panel and try the extension installer again. Only install extensions from sources you trust, because write access to the panel lets an extension modify application code.

## Troubleshooting

Check the directory permissions and PHP-FPM pool account:

```bash
namei -l /var/www/pterodactyl
ls -ld /var/www/pterodactyl /var/www/pterodactyl/storage /var/www/pterodactyl/bootstrap/cache
sudo grep -E '^\s*(user|group)\s*=' /etc/php/8.4/fpm/pool.d/www.conf
```

Inspect recent PHP-FPM and web server errors:

```bash
sudo journalctl -u php8.4-fpm -n 50 --no-pager
sudo tail -n 50 /var/log/nginx/error.log
```

If permissions are correct but installation still fails, consult the extension installer's documentation: some extension managers require their own CLI installer, specific dependencies, or additional permissions.

## Notes

- This guide is for **PHP 8.4-FPM**.
- Do not change permissions inside Minecraft server containers to fix panel write errors; they are separate environments.
- Reassess broad write access after installation if your deployment uses a stricter permissions model.
