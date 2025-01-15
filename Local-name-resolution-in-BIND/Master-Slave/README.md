# README - Konfiguracja DNS Master-Slave na AWS

## Wprowadzenie
Ten projekt przedstawia konfigurację serwerów DNS Master-Slave przy użyciu BIND na dwóch maszynach w AWS.

### Adresy IP:
- **Master**: `107.23.53.86`
- **Slave**: `18.209.84.115`

---

## Kroki konfiguracji

### 1. Konfiguracja Master DNS

#### **1.1 Edycja pliku named.conf.local**
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

#### **1.2 Upewnij się, że pliki stref istnieją**
Utwórz pliki stref, jeśli ich jeszcze nie ma:

Plik: `/etc/bind/zones/mydomain.local.zone`
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

Plik: `/etc/bind/zones/107.23.53.rev`
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

#### **1.3 Restart BIND**
Sprawdź poprawność konfiguracji i zrestartuj usługę:
```bash
sudo named-checkconf
sudo named-checkzone mydomain.local /etc/bind/zones/mydomain.local.zone
sudo named-checkzone 86.53.23.107.in-addr.arpa /etc/bind/zones/107.23.53.rev
sudo systemctl restart bind9
```

---

### 2. Konfiguracja Slave DNS

#### **2.1 Edycja pliku named.conf.local**
Na serwerze Slave edytuj plik `/etc/bind/named.conf.local` i dodaj konfigurację stref jako Slave:

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

#### **2.2 Upewnij się, że katalog na pliki stref istnieje**
Utwórz katalog na pliki stref:
```bash
sudo mkdir -p /var/cache/bind
sudo chown bind:bind /var/cache/bind
```

#### **2.3 Restart BIND**
Sprawdź poprawność konfiguracji i zrestartuj usługę:
```bash
sudo named-checkconf
sudo systemctl restart bind9
```

---

### 3. Testowanie Master-Slave

#### **3.1 Sprawdź logi transferu stref na Slave**
Na Slave upewnij się, że strefy zostały pobrane z Master:
```bash
sudo tail -f /var/log/syslog
```
Oczekiwany wynik:
```text
zone mydomain.local/IN: Transfer started.
zone mydomain.local/IN: Transfer completed.
```

#### **3.2 Test zapytań na Slave**
Wykonaj zapytanie do Slave:
```bash
dig @18.209.84.115 mydomain.local
```
Oczekiwany wynik:
```text
;; ANSWER SECTION:
mydomain.local.  86400  IN  A  107.23.53.86
```

#### **3.3 Test zapasowego działania Slave**
Zatrzymaj Master i sprawdź, czy Slave obsługuje zapytania:
```bash
sudo systemctl stop bind9  # Na Master
```
Następnie wykonaj zapytanie do Slave:
```bash
dig @18.209.84.115 mydomain.local
```

---

## Ważne uwagi
- **Security Groups:** Upewnij się, że port 53 (TCP i UDP) jest otwarty pomiędzy Master a Slave.
- **Monitorowanie:** Regularnie sprawdzaj logi, aby upewnić się, że transfer stref działa poprawnie.
- **Zabezpieczenie:** Skonfiguruj odpowiednie reguły w `allow-transfer`, aby ograniczyć transfer stref tylko do zaufanych serwerów.

---