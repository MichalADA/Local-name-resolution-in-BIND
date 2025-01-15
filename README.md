# Local-name-resolution-in-BIND
Project Objective: Create a DNS server that serves a fictitious domain (e.g. mydomain.local). Reverse DNS for an assigned IP address. Practice configuring, debugging and testing BIND.


# README - Konfiguracja DNS Master-Slave na AWS

## Wprowadzenie
Ten projekt przedstawia konfigurację serwerów DNS Master-Slave przy użyciu BIND na dwóch maszynach w AWS.

### Adresy IP:
- **Master**: `107.23.53.86`
- **Slave**: `18.209.84.115`

---

## Kroki konfiguracji

### 1. Instalacja i konfiguracja BIND na Master i Slave

#### **1.1 Aktualizacja systemu**
Zaktualizuj system operacyjny:
```bash
sudo apt update
sudo apt upgrade -y
```

#### **1.2 Instalacja BIND**
Zainstaluj BIND i powiązane narzędzia:
```bash
sudo apt install bind9 bind9utils bind9-doc -y
```

#### **1.3 Sprawdź status BIND**
Upewnij się, że usługa BIND działa:
```bash
sudo systemctl status bind9
```

---

### 2. Konfiguracja Master DNS

#### **2.1 Edycja pliku named.conf.local**
Na serwerze Master edytuj plik `/etc/bind/named.conf.local` i dodaj konfigurację stref:

```bash
sudo nano /etc/bind/named.conf.local
```

Dodaj:
```text
zone "mydomain.local" {
    type master;
    file "/etc/bind/zones/mydomain.local.zone";
    allow-transfer { 18.209.84.115; };  // IP Slave
};

zone "86.53.23.107.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/107.23.53.rev";
    allow-transfer { 18.209.84.115; };  // IP Slave
};
```

#### **2.2 Utwórz katalog na pliki stref**
Utwórz katalog na pliki stref:
```bash
sudo mkdir -p /etc/bind/zones
```

#### **2.3 Plik strefy mydomain.local.zone**
Utwórz plik strefy:
```bash
sudo nano /etc/bind/zones/mydomain.local.zone
```
Dodaj:
```text
$TTL 86400
@   IN  SOA   ns1.mydomain.local. admin.mydomain.local. (
        2025011401 ; Serial
        3600       ; Refresh
        1800       ; Retry
        1209600    ; Expire
        86400 )    ; Minimum TTL
@   IN  NS    ns1.mydomain.local.
@   IN  A     107.23.53.86
ns1 IN  A     107.23.53.86
www IN  A     107.23.53.86
app IN  A     107.23.53.87
```

#### **2.4 Plik strefy 107.23.53.rev**
Utwórz plik strefy odwrotnej:
```bash
sudo nano /etc/bind/zones/107.23.53.rev
```
Dodaj:
```text
$TTL 86400
@   IN  SOA   ns1.mydomain.local. admin.mydomain.local. (
        2025011401 ; Serial
        3600       ; Refresh
        1800       ; Retry
        1209600    ; Expire
        86400 )    ; Minimum TTL
@   IN  NS    ns1.mydomain.local.
86  IN  PTR   ns1.mydomain.local.
86  IN  PTR   www.mydomain.local.
87  IN  PTR   app.mydomain.local.
```

#### **2.5 Sprawdzenie konfiguracji**
Sprawdź poprawność konfiguracji i plików stref:
```bash
sudo named-checkconf
sudo named-checkzone mydomain.local /etc/bind/zones/mydomain.local.zone
sudo named-checkzone 86.53.23.107.in-addr.arpa /etc/bind/zones/107.23.53.rev
```

#### **2.6 Restart BIND**
Uruchom ponownie usługę BIND:
```bash
sudo systemctl restart bind9
```

---

### 3. Testowanie konfiguracji Master

#### **3.1 Test zapytań do Master**
Sprawdź rekordy DNS na Master:
```bash
dig @107.23.53.86 www.mydomain.local
dig -x 107.23.53.86
```

---

### 4. Konfiguracja Slave DNS

#### **4.1 Edycja pliku named.conf.local**
Na serwerze Slave edytuj plik `/etc/bind/named.conf.local`:

```bash
sudo nano /etc/bind/named.conf.local
```

Dodaj:
```text
zone "mydomain.local" {
    type slave;
    masters { 107.23.53.86; };  // IP Master
    file "/var/cache/bind/mydomain.local.zone";
};

zone "86.53.23.107.in-addr.arpa" {
    type slave;
    masters { 107.23.53.86; };  // IP Master
    file "/var/cache/bind/107.23.53.rev";
};
```

#### **4.2 Utwórz katalog na pliki stref**
Utwórz katalog na pliki stref:
```bash
sudo mkdir -p /var/cache/bind
sudo chown bind:bind /var/cache/bind
```

#### **4.3 Sprawdzenie konfiguracji**
Sprawdź poprawność konfiguracji na Slave:
```bash
sudo named-checkconf
```

#### **4.4 Restart BIND**
Uruchom ponownie usługę BIND:
```bash
sudo systemctl restart bind9
```

---

### 5. Testowanie Master-Slave

#### **5.1 Sprawdź logi transferu stref na Slave**
Na Slave sprawdź, czy strefy zostały pobrane z Master:
```bash
sudo tail -f /var/log/syslog
```
Oczekiwany wynik:
```text
zone mydomain.local/IN: Transfer started.
zone mydomain.local/IN: Transfer completed.
```

#### **5.2 Test zapytań na Slave**
Wykonaj zapytanie do Slave:
```bash
dig @18.209.84.115 www.mydomain.local
```

#### **5.3 Test zapasowego działania Slave**
Zatrzymaj Master i sprawdź, czy Slave obsługuje zapytania:
```bash
sudo systemctl stop bind9  # Na Master
dig @18.209.84.115 www.mydomain.local
```

---

### 6. Jak skonfigurować dostęp na zewnątrz?

#### **6.1 Edytuj plik named.conf.options**
Na Master i Slave edytuj plik `/etc/bind/named.conf.options`:
```bash
sudo nano /etc/bind/named.conf.options
```

Dodaj forwardery:
```text
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8;  // Google DNS
        1.1.1.1;  // Cloudflare DNS
    };

    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
};
```

#### **6.2 Restart BIND**
Zrestartuj usługę:
```bash
sudo systemctl restart bind9
```

---

## Ważne uwagi
- **Security Groups:** Upewnij się, że port 53 (TCP i UDP) jest otwarty między Master a Slave.
- **Zabezpieczenie:** Skonfiguruj odpowiednie reguły w `allow-transfer`, aby ograniczyć transfer stref tylko do zaufanych serwerów.

---

## Co dalej?
- Rozszerzenie konfiguracji o DNSSEC dla dodatkowego bezpieczeństwa.
- Wdrożenie Split-Horizon DNS, aby zwracać różne odpowiedzi w zależności od lokalizacji klienta.
- Automatyzacja zarządzania DNS za pomocą Ansible lub Terraform.
