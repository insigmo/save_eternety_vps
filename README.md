# Спасаем заблокированный «вечный» VPS через IPv6

**Схема:** `Клиент → VPS в РФ (IPv4) → зарубежный VPS (IPv6) → интернет`

IPv4 зарубежного VPS заблокирован, а напрямую на IPv6 клиент часто не подключится (не все операторы его дают). Поэтому ставим посредника — VPS в РФ, который ходит к зарубежному по IPv6.

Почему IPv6, а не новый IPv4: на vdsina.ru IPv4 стоит ~7 ₽/день, IPv6 — копейки.

**Понадобится:**
- зарубежный VPS (заблокированный) с 3x-ui;
- VPS в РФ с 3x-ui (арендовать, если нет);
- IPv6 на **обоих** серверах.

---

## 1. Подключаем IPv6 на обоих VPS

Делалось в vdsina. Поэтому буду приводить в пример его, у вас может быть свой хостер, не думаю что сильно будет отличаться. 
В [панели vdsina](https://cp.vdsina.ru): **Мои серверы → сервер → вкладка IP → «Заказать IPv6»**. Появится подсеть, например `2a10:9200:0:332::/64`. Берём адрес из неё, например `2a10:9200:0:332::1`. Шлюз — `fe80::1`.

> 💡 Проще всего на **новом** сервере: сначала закажите IPv6, потом переустановите ОС — IPv6 настроится сам. На рабочем сервере настраиваем руками (ниже).

> ⚠️ На заблокированный VPS по SSH из РФ не зайти. Для этого шага используйте **VNC-консоль** в панели хостера или любой рабочий VPN.

Ниже `2a10:9200:0:332::1` замените на свой адрес. Строки IPv4 не трогайте.

<details>
<summary><b>Ubuntu 22.04 / 24.04</b></summary>

Файл `/etc/netplan/01-netcfg.yaml` (на 24.04 может называться `50-cloud-init.yaml`), секция `ens3`:

```yaml
ens3:
  addresses:
    - "ВАШ_IPv4/24"
    - "2a10:9200:0:332::1/64"
  nameservers:
    addresses:
      - "8.8.8.8"
      - "2001:4860:4860::8888"
  routes:
    - to: default
      via: ВАШ_ШЛЮЗ_IPv4
      on-link: true
    - to: ::/0
      via: fe80::1
      on-link: true
```

```bash
netplan apply && reboot
```
</details>

<details>
<summary><b>Ubuntu 18.04 / 20.04</b></summary>

Файл `/etc/netplan/01-netcfg.yaml`, секция `ens3`:

```yaml
ens3:
  addresses:
    - ВАШ_IPv4/24
    - 2a10:9200:0:332::1/64
  gateway4: ВАШ_ШЛЮЗ_IPv4
  gateway6: fe80::1
```

```bash
netplan apply && reboot
```
</details>

<details>
<summary><b>Debian 9 - 13</b></summary>

Добавьте в `/etc/network/interfaces`:

```
iface lo inet6 loopback
iface ens3 inet6 static
        address 2a10:9200:0:332::1
        netmask 64
        gateway fe80::1
```

```bash
/etc/init.d/networking restart   # или reboot
```
</details>

<details>
<summary><b>CentOS 7 / 8, AlmaLinux, Rocky</b></summary>

В конец `/etc/sysconfig/network-scripts/ifcfg-eth0` (CentOS 7) или `ifcfg-ens3` (CentOS 8+):

```
IPV6ADDR="2a10:9200:0:332::1"
IPV6_DEFAULTGW="fe80::1"
IPV6INIT=yes
```

```bash
reboot
```
</details>

**Проверка (на обоих серверах):**

```bash
ping -6 -c 3 google.com
```

Источник: [vdsina.ru — Выдаёте ли IPv6?](https://vdsina.ru/qa/q/vydaete-li-ipv6)

---

## 2. Как попасть на зарубежный VPS (SSH и панель)

VPS в РФ используем как **jump-host**: заходим на него по IPv4, а дальше он сам ходит на зарубежный по IPv6. На Windows 10/11 `ssh` уже есть в терминале.

**SSH на зарубежный VPS:**

```bash
ssh -J root@IPv4_РФ root@IPv6_ЗАРУБЕЖ
```

**Веб-панель зарубежного VPS** (проброс порта, `2053` — порт панели, замените на свой):

```bash
ssh -J root@IPv4_РФ -L 2053:localhost:2053 root@IPv6_ЗАРУБЕЖ
```

Не закрывая окно, откройте в браузере `https://localhost:2053/ваш_путь_панели/`. Предупреждение о сертификате — нормально, жмите «Продолжить».

---

## 3. Выпускаем сертификат на IP (на зарубежном VPS)

Скрипт выпускает бесплатный сертификат Let's Encrypt сразу на IPv4 и IPv6 сервера и (по желанию) ставит автопродление.

Запускать **на заблокированном зарубежном VPS**. Нужны свободный 80-й порт и установленный `acme.sh`:

```bash
curl https://get.acme.sh | sh -s email=ваша@почта.ru
```

<details>
<summary><b>📄 Скрипт update_certs.sh (развернуть и скопировать)</b></summary>

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

DEBUG_ENABLED=0
DEBUG_LEVEL=""
LOG_FILE=""
DOMAIN=""
DELETE_AUTO_UPDATE=0
CRON_RUN=0

RENEW_BEFORE_DAYS=3
ACME_SERVER=letsencrypt
CERT_PROFILE=shortlived
HTTP_PORT=80

CERT_OUTPUT_DIR=/etc/ssl/acme-public-ip-certs
CERT_FILE=${CERT_OUTPUT_DIR}/pub_ip.crt
KEY_FILE=${CERT_OUTPUT_DIR}/pub_ip.key
SCRIPT_INSTALL_PATH=/usr/local/sbin/update_certs.sh
CRON_FILE=/etc/cron.d/update-certs
CRON_LOG_FILE=/var/log/update-certs-cron.log
STATE_DIR=/var/lib/update-certs
LOCK_FILE=/run/lock/update-certs.lock

SUCCESS_STATE=${STATE_DIR}/success-pub_ip.timestamp
SUCCESS_ADDR_STATE=${STATE_DIR}/success-pub_ip.addresses
ATTEMPT_STATE=${STATE_DIR}/attempt-pub_ip.timestamp
ATTEMPT_ADDR_STATE=${STATE_DIR}/attempt-pub_ip.addresses

INTERVAL_SECONDS=$((5 * 24 * 60 * 60))
RETRY_SECONDS=$((24 * 60 * 60))

CURRENT_TMP=""
failures=0
BUNDLE_SUCCESS=0
BUNDLE_SKIPPED=0

usage() {
  cat <<'USAGE'
Usage: update_certs.sh [OPTIONS]

Options:
  --domain DOMAIN      Add a DNS name to the certificate after validating its DNS addresses
  --debug [N]          Enable acme.sh debug output (optional level 0-3)
  --log FILE           Write acme.sh logs to the specified file
  --force              Accepted for compatibility; force is always enabled
  --delete-auto-update Remove the cron job and files used for automatic renewal
  -h, --help           Show this help

Certificate files without --domain:
  /etc/ssl/acme-public-ip-certs/pub_ip.crt
  /etc/ssl/acme-public-ip-certs/pub_ip.key

With --domain example.com, files are named example.com.crt and example.com.key.
The .crt file contains the full certificate chain. Automatic renewal is checked hourly
and runs at most once per 5 days for an unchanged set of IPs and domain.
USAGE
}

cleanup() {
  if [[ -n "$CURRENT_TMP" ]]; then
    rm -f -- "$CURRENT_TMP"
  fi
}
trap cleanup EXIT

while (($#)); do
  case "$1" in
    --debug)
      DEBUG_ENABLED=1
      DEBUG_LEVEL=""
      if (($# > 1)) && [[ "$2" =~ ^[0-3]$ ]]; then
        DEBUG_LEVEL="$2"
        shift
      fi
      shift
      ;;
    --log)
      if (($# < 2)) || [[ -z "$2" || "$2" == --* ]]; then
        echo "[ERROR] --log requires a file path" >&2
        exit 2
      fi
      LOG_FILE="$2"
      shift 2
      ;;
    --force)
      # Force mode is always enabled.
      shift
      ;;
    --domain)
      if (($# < 2)) || [[ -z "$2" || "$2" == --* ]]; then
        echo "[ERROR] --domain requires a DNS name" >&2
        exit 2
      fi
      DOMAIN="$2"
      shift 2
      ;;
    --delete-auto-update)
      DELETE_AUTO_UPDATE=1
      shift
      ;;
    --cron-run)
      # Internal option used only by the installed cron job.
      CRON_RUN=1
      shift
      ;;
    -h|--help)
      usage
      exit 0
      ;;
    *)
      echo "[ERROR] Unknown argument: $1" >&2
      usage >&2
      exit 2
      ;;
  esac
done

if (( EUID != 0 )); then
  echo "[ERROR] Run as root: sudo $0 [options]" >&2
  exit 1
fi

# Acquire the same lock for normal runs and deletion so cleanup cannot race a cron run.
install -d -m 755 /run/lock
exec 9>"$LOCK_FILE"
if command -v flock >/dev/null 2>&1; then
  if ! flock -n 9; then
    echo "[INFO] Another update process is running; skipping."
    exit 0
  fi
fi

# Disable automatic renewal without removing the installed certificates or key.
if (( DELETE_AUTO_UPDATE )); then
  echo "[INFO] Removing automatic certificate renewal..."
  rm -f -- "$CRON_FILE" "$SCRIPT_INSTALL_PATH" "$CRON_LOG_FILE"
  rm -rf -- "$STATE_DIR"
  echo "[OK] Cron configuration removed: $CRON_FILE"
  echo "[OK] Installed script removed: $SCRIPT_INSTALL_PATH"
  echo "[OK] Automatic renewal state and log removed"
  echo "[INFO] Existing certificate and private key were preserved."
  echo "[OK] Automatic renewal disabled"
  exit 0
fi

install -d -o root -g root -m 700 "$STATE_DIR"

write_timestamp() {
  local file="$1"
  local timestamp="$2"
  local tmp="${file}.tmp.$$"

  printf '%s\n' "$timestamp" > "$tmp"
  chmod 600 "$tmp"
  mv -f -- "$tmp" "$file"
}

write_text() {
  local file="$1"
  local value="$2"
  local tmp="${file}.tmp.$$"

  printf '%s\n' "$value" > "$tmp"
  chmod 600 "$tmp"
  mv -f -- "$tmp" "$file"
}

read_timestamp() {
  local file="$1"
  local value=""

  [[ -r "$file" ]] || return 1
  value="$(<"$file")"
  [[ "$value" =~ ^[0-9]+$ ]] || return 1
  printf '%s\n' "$value"
}

install_missing_dependencies() {
  local -a missing=()

  command -v curl >/dev/null 2>&1 || missing+=(curl)
  command -v socat >/dev/null 2>&1 || missing+=(socat)

  if ((${#missing[@]} == 0)); then
    echo "[OK] curl and socat are installed"
    return 0
  fi

  echo "[INFO] Installing dependencies: ${missing[*]}"

  if command -v apt-get >/dev/null 2>&1; then
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y "${missing[@]}"
  elif command -v dnf >/dev/null 2>&1; then
    dnf install -y "${missing[@]}"
  elif command -v yum >/dev/null 2>&1; then
    yum install -y "${missing[@]}"
  elif command -v apk >/dev/null 2>&1; then
    apk add --no-cache "${missing[@]}"
  else
    echo "[ERROR] Cannot install dependencies automatically: ${missing[*]}" >&2
    exit 1
  fi

  command -v curl >/dev/null 2>&1 || {
    echo "[ERROR] curl is still unavailable" >&2
    exit 1
  }
  command -v socat >/dev/null 2>&1 || {
    echo "[ERROR] socat is still unavailable" >&2
    exit 1
  }

  echo "[OK] Dependencies installed"
}

find_acme_sh() {
  local candidate
  local candidates=(
    "$(command -v acme.sh 2>/dev/null || true)"
    /root/.acme.sh/acme.sh
    /home/*/.acme.sh/acme.sh
    /usr/local/bin/acme.sh
    /usr/bin/acme.sh
  )

  for candidate in "${candidates[@]}"; do
    [[ -n "$candidate" && -x "$candidate" && -f "$candidate" ]] || continue
    printf '%s\n' "$candidate"
    return 0
  done
  return 1
}

valid_ipv4() {
  local ip="$1"
  local part
  local -a octets

  [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] || return 1
  IFS='.' read -r -a octets <<< "$ip"
  for part in "${octets[@]}"; do
    (( 10#$part <= 255 )) || return 1
  done
  return 0
}

valid_ipv6() {
  local ip="$1"
  [[ "$ip" == *:* && "$ip" =~ ^[0-9A-Fa-f:]+$ ]]
}

valid_domain() {
  local domain="$1"

  # Require a DNS hostname with at least two labels; each label is 1-63 chars.
  # Supports ASCII names and IDNA/Punycode (xn--...) names, but not raw Unicode.
  [[ -n "$domain" && ${#domain} -le 251 && "$domain" == *.* ]] || return 1
  valid_ipv4 "$domain" && return 1
  [[ "$domain" == *:* ]] && return 1
  [[ "$domain" =~ ^([A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?\.)+[A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?$ ]]
}

resolve_domain_ips() {
  local domain="$1"

  # Resolve both record families through the system resolver. This avoids relying
  # on ICMP: ping can be blocked even when HTTP/ACME validation is reachable.
  # Some NSS/glibc configurations return IPv4 addresses from the IPv6 lookup as
  # IPv4-mapped IPv6 values (for example, ::ffff:212.118.38.99). Normalize those
  # back to ordinary IPv4 before comparing them with the server's detected IPs.
  {
    getent ahostsv4 "$domain" 2>/dev/null | awk '{print $1}' || true
    getent ahostsv6 "$domain" 2>/dev/null | awk '{
      ip = tolower($1)
      if (ip ~ /^::ffff:[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/) {
        sub(/^::ffff:/, "", ip)
      }
      print ip
    }' || true
  } | LC_ALL=C sort -u
}

fetch_public_ips() {
  local family="$1"
  local url ip found=0
  local -A seen=()
  local -a urls

  if [[ "$family" == 4 ]]; then
    urls=(https://api.ipify.org https://ipv4.icanhazip.com https://v4.ident.me)
  else
    urls=(https://api64.ipify.org https://ipv6.icanhazip.com https://v6.ident.me)
  fi

  for url in "${urls[@]}"; do
    ip="$(
      curl --noproxy '*' "-$family" -fsS \
        --connect-timeout 5 --max-time 12 "$url" 2>/dev/null |
        tr -d '[:space:]' || true
    )"

    if [[ "$family" == 4 ]] && valid_ipv4 "$ip"; then
      if [[ -z "${seen[$ip]+x}" ]]; then
        printf '%s\n' "$ip"
        seen["$ip"]=1
        found=1
      fi
    elif [[ "$family" == 6 ]] && valid_ipv6 "$ip"; then
      if [[ -z "${seen[$ip]+x}" ]]; then
        printf '%s\n' "$ip"
        seen["$ip"]=1
        found=1
      fi
    fi
  done

  if (( ! found )); then
    echo "[WARN] Could not detect a public IPv${family} address" >&2
  fi
  return 0
}

# Run a command quietly and show a simple progress indicator.
run_quietly() {
  local description="$1"
  shift

  local rc=0
  local spinner='|/-\'
  local index=0
  local frame
  local pid

  CURRENT_TMP="$(mktemp)"
  "$@" >"$CURRENT_TMP" 2>&1 &
  pid=$!

  if [[ -t 1 ]]; then
    while kill -0 "$pid" 2>/dev/null; do
      frame="${spinner:index:1}"
      printf '\r[....] %s %s' "$description" "$frame"
      index=$(((index + 1) % ${#spinner}))
      sleep 0.25
    done
  else
    echo "[RUN] $description"
  fi

  if wait "$pid"; then
    rc=0
  else
    rc=$?
  fi

  if [[ -t 1 ]]; then
    printf '\r\033[K'
  fi

  if (( rc != 0 )); then
    echo "[ERROR] ${description} failed (exit code: ${rc})" >&2
    if (( DEBUG_ENABLED )) && [[ -z "$LOG_FILE" ]]; then
      echo "[DEBUG] acme.sh output:" >&2
      sed -n '1,120p' "$CURRENT_TMP" >&2
    fi
    if [[ -n "$LOG_FILE" ]]; then
      echo "[INFO] Details: $LOG_FILE" >&2
    else
      echo "[INFO] Use --debug 2 --log /var/log/update-certs.log for diagnostics." >&2
    fi

    rm -f -- "$CURRENT_TMP"
    CURRENT_TMP=""
    return "$rc"
  fi

  rm -f -- "$CURRENT_TMP"
  CURRENT_TMP=""
  return 0
}

ensure_cron_available() {
  if ! command -v cron >/dev/null 2>&1 && ! command -v crond >/dev/null 2>&1; then
    echo "[INFO] Cron daemon not found; installing it."

    if command -v apt-get >/dev/null 2>&1; then
      export DEBIAN_FRONTEND=noninteractive
      apt-get update || return 1
      apt-get install -y cron || return 1
    elif command -v dnf >/dev/null 2>&1; then
      dnf install -y cronie || return 1
    elif command -v yum >/dev/null 2>&1; then
      yum install -y cronie || return 1
    else
      echo "[ERROR] Cannot install a cron daemon automatically." >&2
      return 1
    fi
  fi

  if command -v systemctl >/dev/null 2>&1 && [[ -d /run/systemd/system ]]; then
    if command -v cron >/dev/null 2>&1; then
      systemctl enable --now cron || return 1
    elif command -v crond >/dev/null 2>&1; then
      systemctl enable --now crond || return 1
    fi
  elif command -v service >/dev/null 2>&1; then
    if command -v cron >/dev/null 2>&1; then
      service cron start >/dev/null 2>&1 || true
    elif command -v crond >/dev/null 2>&1; then
      service crond start >/dev/null 2>&1 || true
    fi
  fi
}

setup_auto_update() {
  local source_script
  local domain_arg=""
  source_script="$(readlink -f -- "${BASH_SOURCE[0]}")"

  # DOMAIN is validated to contain only hostname characters before this is called.
  if [[ -n "$DOMAIN" ]]; then
    domain_arg=" --domain ${DOMAIN}"
  fi

  if [[ ! -f "$source_script" ]]; then
    echo "[ERROR] Cannot locate the current script." >&2
    return 1
  fi

  ensure_cron_available || return 1
  install -d -o root -g root -m 755 /usr/local/sbin || return 1
  install -d -o root -g root -m 755 /etc/cron.d || return 1

  if [[ "$source_script" != "$SCRIPT_INSTALL_PATH" ]]; then
    install -o root -g root -m 750 "$source_script" "$SCRIPT_INSTALL_PATH" || return 1
  else
    chmod 750 "$SCRIPT_INSTALL_PATH" || return 1
  fi

  touch "$CRON_LOG_FILE" || return 1
  chown root:root "$CRON_LOG_FILE" || return 1
  chmod 600 "$CRON_LOG_FILE" || return 1

  cat > "$CRON_FILE" <<EOF_CRON
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
17 * * * * root ${SCRIPT_INSTALL_PATH} --cron-run${domain_arg} >> ${CRON_LOG_FILE} 2>&1
EOF_CRON

  chown root:root "$CRON_FILE" || return 1
  chmod 644 "$CRON_FILE" || return 1

  echo "[OK] Automatic certificate renewal enabled."
  echo "[INFO] Script: $SCRIPT_INSTALL_PATH"
  echo "[INFO] Cron: $CRON_FILE"
  echo "[INFO] Checked hourly; the combined certificate is renewed every 5 days."
  echo "[INFO] Cron log: $CRON_LOG_FILE"
}

refresh_installed_script() {
  local source_script
  source_script="$(readlink -f -- "${BASH_SOURCE[0]}")"

  [[ -f "$CRON_FILE" ]] || return 0
  [[ -f "$source_script" ]] || return 1

  if [[ "$source_script" != "$SCRIPT_INSTALL_PATH" ]]; then
    install -o root -g root -m 750 "$source_script" "$SCRIPT_INSTALL_PATH" || return 1
    echo "[INFO] Installed automatic-update script refreshed."
  fi
}

prompt_auto_update() {
  local answer

  if [[ -f "$CRON_FILE" ]]; then
    # Refresh both the installed script and the cron argument so --domain
    # remains consistent with the certificate successfully issued this run.
    if ! setup_auto_update; then
      echo "[ERROR] Could not refresh automatic-renewal configuration." >&2
      return 1
    fi
    echo "[INFO] Automatic certificate renewal is already configured and has been updated."
    echo "[INFO] Use --delete-auto-update to disable it."
    return 0
  fi

  if [[ ! -t 0 ]]; then
    echo "[INFO] Non-interactive session; automatic renewal was not enabled."
    echo "[INFO] Run the script interactively to configure it."
    return 0
  fi

  while true; do
    echo
    read -r -p \
      "Would you like to enable automatic certificate renewal every 5 days? [Y/n] " \
      answer || answer=""

    case "$answer" in
      ""|[Yy]|[Yy][Ee][Ss])
        if setup_auto_update; then
          return 0
        fi
        echo "[ERROR] Certificate was installed, but automatic renewal setup failed." >&2
        return 1
        ;;
      [Nn]|[Nn][Oo])
        echo "[INFO] Automatic renewal was not enabled."
        return 0
        ;;
      *)
        echo "Please answer Y or N."
        ;;
    esac
  done
}

if [[ -n "$DOMAIN" ]]; then
  # Normalize DNS input before validating or using it in file paths.
  DOMAIN="${DOMAIN%.}"
  DOMAIN="${DOMAIN,,}"
  if ! valid_domain "$DOMAIN"; then
    echo "[ERROR] Invalid domain name: $DOMAIN" >&2
    echo "[ERROR] Use a valid DNS hostname, e.g. example.com (ASCII/Punycode)." >&2
    exit 2
  fi
fi

install_missing_dependencies

ACME_BIN="$(find_acme_sh || true)"
if [[ -z "$ACME_BIN" ]]; then
  echo "[ERROR] acme.sh was not found. Install it first." >&2
  exit 1
fi

echo "[INFO] Using acme.sh: $ACME_BIN"
echo "[INFO] Force renewal: enabled"

install -d -o root -g root -m 750 "$CERT_OUTPUT_DIR"

if [[ -n "$LOG_FILE" ]]; then
  mkdir -p "$(dirname -- "$LOG_FILE")"
  touch "$LOG_FILE"
  chmod 600 "$LOG_FILE"
fi

ACME_OPTIONAL_ARGS=()
if (( DEBUG_ENABLED )); then
  ACME_OPTIONAL_ARGS+=(--debug)
  if [[ -n "$DEBUG_LEVEL" ]]; then
    ACME_OPTIONAL_ARGS+=("$DEBUG_LEVEL")
  fi
fi
if [[ -n "$LOG_FILE" ]]; then
  ACME_OPTIONAL_ARGS+=(--log "$LOG_FILE")
fi

mapfile -t IPV4S < <(fetch_public_ips 4)
mapfile -t IPV6S < <(fetch_public_ips 6)

if ((${#IPV4S[@]} == 0 || ${#IPV6S[@]} == 0)); then
  echo "[ERROR] A combined certificate requires at least one public IPv4 and one public IPv6 address." >&2
  exit 1
fi

ALL_IPS=("${IPV4S[@]}" "${IPV6S[@]}")
mapfile -t ALL_IPS < <(printf '%s\n' "${ALL_IPS[@]}" | LC_ALL=C sort -u)

printf '[INFO] Detected public IPv4 address(es): %s\n' "${IPV4S[*]}"
printf '[INFO] Detected public IPv6 address(es): %s\n' "${IPV6S[*]}"
if ((${#ALL_IPS[@]} > 2)); then
  echo "[WARN] More than two distinct public IPs were detected; all will be included as certificate SANs."
fi

# A supplied DNS name must resolve exclusively to one or more of this host's
# publicly detected IP addresses. This is stricter than merely finding one match,
# because ACME could otherwise validate against an unexpected A/AAAA destination.
if [[ -n "$DOMAIN" ]]; then
  mapfile -t DOMAIN_IPS < <(resolve_domain_ips "$DOMAIN")

  if ((${#DOMAIN_IPS[@]} == 0)); then
    echo "[ERROR] Domain '$DOMAIN' does not resolve to any IPv4 or IPv6 addresses." >&2
    exit 1
  fi

  declare -A PUBLIC_IP_SET=()
  for ip in "${ALL_IPS[@]}"; do
    PUBLIC_IP_SET["$ip"]=1
  done

  DOMAIN_MATCHED=0
  for resolved_ip in "${DOMAIN_IPS[@]}"; do
    if [[ -z "${PUBLIC_IP_SET[$resolved_ip]+x}" ]]; then
      echo "[ERROR] Domain '$DOMAIN' resolves to an IP not detected on this server: $resolved_ip" >&2
      echo "[ERROR] All A/AAAA records must point to this server's public IPs before requesting a certificate." >&2
      exit 1
    fi
    DOMAIN_MATCHED=1
  done

  if (( ! DOMAIN_MATCHED )); then
    echo "[ERROR] Domain '$DOMAIN' does not resolve to this server's public IPs." >&2
    exit 1
  fi

  CERT_FILE="${CERT_OUTPUT_DIR}/${DOMAIN}.crt"
  KEY_FILE="${CERT_OUTPUT_DIR}/${DOMAIN}.key"
  printf '[OK] Domain validated: %s -> %s\n' "$DOMAIN" "${DOMAIN_IPS[*]}"
else
  CERT_FILE="${CERT_OUTPUT_DIR}/pub_ip.crt"
  KEY_FILE="${CERT_OUTPUT_DIR}/pub_ip.key"
fi

echo "[INFO] Certificate output: $CERT_FILE"
echo "[INFO] Private key output: $KEY_FILE"

CURRENT_ADDRESS_SET="$(printf '%s\n' "${ALL_IPS[@]}" | LC_ALL=C sort -u | paste -sd, -)"
if [[ -n "$DOMAIN" ]]; then
  CURRENT_ADDRESS_SET="${CURRENT_ADDRESS_SET}|domain=${DOMAIN}"
fi

# Skip periodic cron runs until 120 hours have passed for the same IP set.
# If addresses change, retry immediately so the new IPs are covered.
if (( CRON_RUN )); then
  now="$(date +%s)"

  last_success=""
  last_success_addresses=""
  last_attempt=""
  last_attempt_addresses=""
  last_success="$(read_timestamp "$SUCCESS_STATE" || true)"
  [[ -r "$SUCCESS_ADDR_STATE" ]] && last_success_addresses="$(<"$SUCCESS_ADDR_STATE")"
  last_attempt="$(read_timestamp "$ATTEMPT_STATE" || true)"
  [[ -r "$ATTEMPT_ADDR_STATE" ]] && last_attempt_addresses="$(<"$ATTEMPT_ADDR_STATE")"

  if [[ -n "$last_success" && "$last_success_addresses" == "$CURRENT_ADDRESS_SET" ]] &&
     (( now - last_success < INTERVAL_SECONDS )); then
    BUNDLE_SKIPPED=1
    echo "[INFO] Combined certificate is current; next renewal is due after the five-day interval."
  elif [[ -n "$last_attempt" && "$last_attempt_addresses" == "$CURRENT_ADDRESS_SET" ]] &&
       (( now - last_attempt < RETRY_SECONDS )); then
    BUNDLE_SKIPPED=1
    echo "[INFO] A recent attempt for this IP set failed; retrying after the 24-hour retry interval."
  else
    write_timestamp "$ATTEMPT_STATE" "$now"
    write_text "$ATTEMPT_ADDR_STATE" "$CURRENT_ADDRESS_SET"
  fi
fi

issue_combined_certificate() {
  local primary_ip="${IPV4S[0]}"
  local ip
  local now
  local -a issue_args install_args

  echo
  echo "[INFO] Requesting one certificate covering all detected IPv4/IPv6 addresses."

  issue_args=(
    --issue
    --standalone
    --httpport "$HTTP_PORT"
    --server "$ACME_SERVER"
    --certificate-profile "$CERT_PROFILE"
    --days "$RENEW_BEFORE_DAYS"
    --keylength ec-256
    --force
    --listen-v4
    --listen-v6
  )

  # Keep an IP as the primary identifier used by acme.sh's install-cert lookup.
  # The requested DNS name is an additional SAN when --domain was provided.
  for ip in "${ALL_IPS[@]}"; do
    issue_args+=(-d "$ip")
  done
  if [[ -n "$DOMAIN" ]]; then
    issue_args+=(-d "$DOMAIN")
  fi
  issue_args+=("${ACME_OPTIONAL_ARGS[@]}")

  if ! run_quietly "Issuing combined IP certificate" "$ACME_BIN" "${issue_args[@]}"; then
    return 1
  fi

  echo "[OK] Combined certificate issued."

  # The first -d entry is the primary domain in acme.sh's certificate store.
  # Save the full chain directly as pub_ip.crt, and the matching private key as pub_ip.key.
  install_args=(
    --install-cert
    -d "$primary_ip"
    --ecc
    --key-file "$KEY_FILE"
    --fullchain-file "$CERT_FILE"
  )
  install_args+=("${ACME_OPTIONAL_ARGS[@]}")

  if ! run_quietly "Installing combined certificate and private key" "$ACME_BIN" "${install_args[@]}"; then
    return 1
  fi

  if [[ ! -s "$CERT_FILE" || ! -s "$KEY_FILE" ]]; then
    echo "[ERROR] Certificate or private key file is missing or empty." >&2
    return 1
  fi

  chown root:root "$CERT_FILE" "$KEY_FILE"
  chmod 644 "$CERT_FILE"
  chmod 600 "$KEY_FILE"

  now="$(date +%s)"
  write_timestamp "$SUCCESS_STATE" "$now"
  write_text "$SUCCESS_ADDR_STATE" "$CURRENT_ADDRESS_SET"
  rm -f -- "$ATTEMPT_STATE" "$ATTEMPT_ADDR_STATE"

  BUNDLE_SUCCESS=1
  echo "[OK] Combined certificate and private key installed."
  return 0
}

if (( ! BUNDLE_SKIPPED )); then
  if ! issue_combined_certificate; then
    failures=$((failures + 1))
  fi
fi

echo
if (( BUNDLE_SUCCESS )); then
  echo "========================================"
  echo " Certificate and key locations"
  echo "========================================"
  printf 'IPv4 SAN(s):  %s\n' "${IPV4S[*]}"
  printf 'IPv6 SAN(s):  %s\n' "${IPV6S[*]}"
  printf 'Certificate:  %s\n' "$CERT_FILE"
  printf 'Private key:  %s\n' "$KEY_FILE"
elif (( BUNDLE_SKIPPED )); then
  echo "[INFO] No renewal was needed for the current IP set."
  if [[ -s "$CERT_FILE" && -s "$KEY_FILE" ]]; then
    printf '[INFO] Certificate: %s\n' "$CERT_FILE"
    printf '[INFO] Private key: %s\n' "$KEY_FILE"
  fi
fi

echo
if [[ -n "$LOG_FILE" ]]; then
  echo "[INFO] acme.sh log: $LOG_FILE"
fi

echo "[INFO] Certificate directory: $CERT_OUTPUT_DIR"

if (( failures > 0 )); then
  echo "[WARN] Certificate operation failed." >&2
  echo "[INFO] Automatic renewal setup was not offered because issuance or installation failed."
  exit 1
fi

# Cron must never prompt for automatic-update setup.
if (( CRON_RUN )); then
  if (( BUNDLE_SUCCESS )); then
    echo "[OK] Scheduled combined certificate renewal completed successfully."
  fi
  exit 0
fi

if ! prompt_auto_update; then
  exit 1
fi

echo
if (( BUNDLE_SUCCESS )); then
  echo "[OK] Combined certificate operation completed successfully."
else
  echo "[OK] No certificate renewal was required."
fi
```
</details>

Сохраните скрипт и запустите:

```bash
nano update_certs.sh          # вставить скрипт, Ctrl+O, Enter, Ctrl+X
chmod +x update_certs.sh
sudo ./update_certs.sh
```

На вопрос про автопродление ответьте **Y**. Сертификат появится в `/etc/ssl/acme-public-ip-certs/` (`pub_ip.crt` и `pub_ip.key`).

---

## 4. Настраиваем 3x-ui на зарубежном VPS

<details>
<summary><b>🐳 Только если 3x-ui запущен через Docker Compose</b></summary>

Добавьте папку с сертификатами в `volumes` в `compose.yml`:

```yaml
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    hostname: vpn
    volumes:
      - $PWD/db/:/etc/x-ui/
      - /etc/ssl/acme-public-ip-certs/:/root/cert/   # ← эта строка
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      XUI_ENABLE_FAIL2BAN: "true"
    tty: true
    network_mode: host
    restart: unless-stopped
```

Пересоздайте контейнер и зайдите в него:

```bash
docker compose up -d --force-recreate && docker exec -it 3x-ui bash
```

Дальше блок команд ниже выполняйте **внутри контейнера**.
</details>

Скопируйте и выполните целиком. Блок включает панель и подписку на всех адресах (IPv4 + IPv6), прописывает сертификаты и выводит ссылку на панель по IPv6:

```bash
DB=/etc/x-ui/x-ui.db
if [ -f /root/cert/pub_ip.crt ]; then C=/root/cert; else C=/etc/ssl/acme-public-ip-certs; fi
command -v sqlite3 >/dev/null || apk add --no-cache sqlite 2>/dev/null || (apt-get update && apt-get install -y sqlite3)
sqlite3 "$DB" "DELETE FROM settings WHERE key IN ('webListen','subListen','subEnable','webCertFile','webKeyFile','subCertFile','subKeyFile');
INSERT INTO settings(key,value) VALUES ('webListen',''),('subListen',''),('subEnable','true'),
('webCertFile','$C/pub_ip.crt'),('webKeyFile','$C/pub_ip.key'),('subCertFile','$C/pub_ip.crt'),('subKeyFile','$C/pub_ip.key');"
get() { sqlite3 "$DB" "SELECT value FROM settings WHERE key='$1' LIMIT 1"; }
IP6=$(ip -6 -o addr show scope global | awk '{print $4}' | cut -d/ -f1 | head -n1)
P=$(get webPort); B=$(get webBasePath); SP=$(get subPort); SPATH=$(get subPath)
echo; echo "Панель:   https://[$IP6]:${P:-2053}${B:-/}"
echo "Подписка: https://[$IP6]:${SP:-2096}${SPATH:-/sub/}<id_подписки>"
```

Перезапустите панель:
- **Docker:** `exit`, затем `docker restart 3x-ui`
- **Без Docker:** `x-ui restart`

Запомните выведенный IPv6-адрес — он нужен дальше.

---

## 5. Зарубежный VPS: инбаунд и подписка

1. В панели: **Inbounds → Add Inbound** (например, VLESS). Создайте клиента.
2. Скопируйте ссылку на **подписку** клиента, например:
   `https://example.com:2096/pubsub/mua8g4l122y8wbb0`
3. Замените в ней адрес на IPv6 из шага 4 (в квадратных скобках!):
   `https://[2a10:9200:0:332::1]:2096/pubsub/mua8g4l122y8wbb0`

---

## 6. VPS в РФ: инбаунд, аутбаунд, маршрут

**Инбаунд для клиентов.** **Inbounds → Add Inbound**: VLESS, транспорт **gRPC**. Создайте клиента.

**Аутбаунд на зарубежный VPS.** **Outbounds → Subscriptions → Add** → вставьте ссылку из шага 5 → **Add** → **Refresh all**. В колонке **Status** должна появиться зелёная галочка ✅.

<details>
<summary><b>❌ Галочки нет — что проверить</b></summary>

Выполните на **VPS в РФ** (подставьте свои данные):

```bash
ping -6 -c 3 google.com                       # есть ли IPv6 на РФ-сервере
ping -6 -c 3 IPv6_ЗАРУБЕЖ                     # виден ли зарубежный VPS
ssh root@IPv6_ЗАРУБЕЖ                         # пускает ли по SSH
curl -6 -v "https://[IPv6_ЗАРУБЕЖ]:2096/pubsub/ВАШ_ID"   # отдаётся ли подписка
```

И на **зарубежном VPS**:

```bash
ss -tlnp | grep -E '2096|2053'                # слушает ли панель/подписка
```

Скопируйте вывод команд и спросите у ИИ, что не так.
</details>

**Связываем вход и выход.** **Routing → Routing rules → + Routing rule**. Заполните **только два поля**, остальное не трогайте:

| Поле             | Значение                                           |
|------------------|----------------------------------------------------|
| **Inbound tags** | ваш gRPC-инбаунд (например, `in-33343-tcp (grpc)`) |
| **Outbound tag** | аутбаунд из подписки (например, `sub1-...`)        |

**Save Changes.**

---

## 7. Проверка

1. Скопируйте конфиг клиента **с VPS в РФ** и добавьте в VPN-клиент.
2. Подключитесь и откройте [2ip.io](https://2ip.io).
3. Там должен быть **зарубежный** IP, а не российский. Готово 🎉
