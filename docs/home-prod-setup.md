# Розгортання на домашній прод (Continuous Deployment)

Цей документ описує автоматичну доставку зібраної версії сайту (`dist/`) на домашній сервер (продакшн) після злиття змін в основну гілку `main`.

---

## 1. Схема роботи

1. Розробник вливає PR в гілку `main`.
2. GitHub Actions збирає сайт (`npm run build`), перевіряє лінтери, типи та запускає аудит Lighthouse CI.
3. Якщо всі перевірки успішні, паралельно запускаються:
   - Деплой на **GitHub Pages** (публічне дзеркало).
   - Деплой на **домашній прод** (через SSH + rsync).

---

## 2. Необхідні GitHub Secrets

Для безпечного підключення до домашнього сервера в налаштуваннях репозиторію на GitHub (**Settings → Secrets and variables → Actions**) додаються такі секрети:

| Назва секрету     | Опис                                    | Приклад значення                          |
| ----------------- | --------------------------------------- | ----------------------------------------- |
| `PROD_HOST`       | Біла IP-адреса або доменне ім'я сервера | `home.knu-ce.com` або `93.175.x.x`        |
| `PROD_USER`       | Користувач Linux на сервері             | `deploy` або `ubuntu`                     |
| `PROD_SSH_KEY`    | Приватний SSH-ключ (ed25519)            | `-----BEGIN OPENSSH PRIVATE KEY----- ...` |
| `PROD_PORT`       | SSH-порт (за замовчуванням `22`)        | `22` (або прокинутий порт роутера)        |
| `PROD_TARGET_DIR` | Шлях до директорії сайту на сервері     | `/var/www/ce-knu-site`                    |

> Якщо секрет `PROD_HOST` не вказано, крок деплою на домашній прод автоматично пропускається, не ламаючи основний пайплайн.

---

## 3. Інструкція з налаштування сервера для Ігоря

### Крок 1. Генерація безпечного SSH-ключа для GitHub Actions

На сервері під користувачем, який буде приймати файли:

```bash
# Генерація окремого ed25519 ключа
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_deploy -N ""

# Додавання публічного ключа до списку дозволених
cat ~/.ssh/github_deploy.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh

# Вивід приватного ключа для додавання в GitHub Secrets (PROD_SSH_KEY)
cat ~/.ssh/github_deploy
```

### Крок 2. Підготовка цільової директорії для сайту

```bash
sudo mkdir -p /var/www/ce-knu-site
sudo chown -R $USER:$USER /var/www/ce-knu-site
```

### Крок 3. Налаштування веб-сервера (Apache / Nginx / Docker)

Якщо сайт віддається через Apache (як у `compose.yaml`):

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/ce-knu-site
    <Directory /var/www/ce-knu-site>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

---

## 4. Альтернативний варіант (якщо сервер за домашнім NAT без білої IP)

Якщо провайдер не надає публічної білої IP-адреси і неможливо прокинути 22 порт, використовується **GitHub Actions Self-Hosted Runner**:

1. У репозиторії перейти в: **Settings → Actions → Runners → New self-hosted runner**.
2. Обрати **Linux / x64** та виконати запропоновані команди на сервері.
3. Після цього джоба в GitHub Actions запускається напряму на домашньому сервері без потреби у зовнішніх SSH-портах.
