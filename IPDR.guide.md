# Bida IPDR Operation Guide 
*The system's goal is to validate and import raw data into data warehouse and show appropriate monitoring of the progress.*  

The system contains a **Console Application** for operation and a **Web Application** for monitoring.  

> The application binaries are available in `/home/bida/ipdr` path.

### The Console Application:

**Data Ingestion Process**
*It provides a mechanism to validate the data files and prepare them for the next pipeline.*
The process executes as systemd service:
```ini
[Unit]  
Description=BiDA IPDR File Process service  
  
[Service]  
Type=notify  
# will set the Current Working Directory (CWD). Worker service will have issues without this setting  
WorkingDirectory=/home/bida/ipdr/app/  
# systemd will run this executable to start the service  
ExecStart=/home/bida/ipdr/app/Bida.Ipdr.App FileProcess  
# to query logs using journalctl, set a logical name here SyslogIdentifier=fileProcess  
  
  
# Use your username to keep things simple.  
# If you pick a different user, make sure dotnet and all permissions are set correctly to run the app  
# To update permissions, use 'chown yourusername -R /srv/Worker' to take ownership of the folder and files,  
#       Use 'chmod +x /srv/Worker/Worker' to allow execution of the executable file  
User=bida   
# ensure the service restarts after crashing  
Restart=always  
# amount of time to wait before restarting the service  
RestartSec=60  
  
  
# This environment variable is necessary when dotnet isn't loaded for the specified user.  
# To figure out this value, run 'env | grep DOTNET_ROOT' when dotnet has been loaded into your shell.  
Environment=DOTNET_ROOT=/opt/rh/rh-dotnet31/root/usr/lib64/dotnet  
  
[Install]  
WantedBy=multi-user.target
```
The process's execution scheduling is it starts every day at 8:00 AM and stopped at 1:00 PM.
```bash
00 08 * * * /bin/systemctl start fileProcess.service
00 01 * * * /bin/systemctl stop fileProcess.service
 ```

**MemSQL Pipelines process**
*It provides a mechanism to imports files into the data warehouse.*
It starts one time one lives alive during the lifetime of the OS.
The process execution is described as below:
```bash
/home/bida/ipdr/app/Bida.Ipdr.App Pipeline
```
**Data Analysis process**
*It analysis the data that already uploaded into the data warehouse by the previous pipeline and then shrinks the storage to make it ready for the next step.*
The process execution scheduling starts every day at 2:00 AM and usually takes up to 6 hours.
```bash
00 02 * * * cd /home/bida/ipdr/app/ && ./Bida.Ipdr.App ProcedureCall
```
**File Clean Up process**
*It cleanups the processed file from storage to make room for the new files to process.*
The process execution scheduling starts every day at 12:00 AM.
```bash
00 12 * * * cd /home/bida/ipdr/app/ && ./Bida.Ipdr.App FolderRemove
```   
   
**App Deployment**
Go to the project root folder
```bash
/home/hamed/source/BiDA/BiDA.IPDR/src/Bida.Ipdr.App
```   
Run *Publish* command
```bash
dotnet publish Bida.Ipdr.App.csproj --configuration Release --framework netcoreapp3.1 --output bin/publish
```   
Go to the *Release* (bin/publish) folder and copy the new published file into the project's folder of the server.
```bash
-rw-rw-r--  1 hamed hamed 122K Jun  8 06:20 Bida.Ipdr.App.deps.json
-rwxr-xr-x  1 hamed hamed  89K Jun  8 06:20 Bida.Ipdr.App
-rw-rw-r--  1 hamed hamed  25K Jun  8 06:20 Bida.Ipdr.App.dll
-rw-rw-r--  1 hamed hamed  18K Jun  8 06:20 Bida.Ipdr.App.pdb
-rw-rw-r--  1 hamed hamed  146 Jun  8 06:19 Bida.Ipdr.App.runtimeconfig.json
-rw-rw-r--  1 hamed hamed  49K Jun  8 06:19 Bida.Ipdr.Data.dll
-rw-rw-r--  1 hamed hamed  23K Jun  8 06:19 Bida.Ipdr.Data.pdb
-rw-rw-r--  1 hamed hamed  24K Jun  8 06:19 Bida.Ipdr.Common.dll
-rw-rw-r--  1 hamed hamed  16K Jun  8 06:19 Bida.Ipdr.Common.pdb
-rw-rw-r--  1 hamed hamed  938 May 21 00:56 appsettings.json
```  
*Consider stopping the `fileProcess` service before deployment.*

### The Web Application:
*It provides monitoring (and details log) system for all described pipelines above.*

![IPDR Monitoring](https://github.com/SorenZ/Bida.IPDR/blob/main/Bida-IPDR.png?raw=true)

The process executes as systemd service:
```ini
[Unit]  
Description=BIDA IPDR web site  
  
[Service]  
Type=notify  
# will set the Current Working Directory (CWD)  
WorkingDirectory=/home/bida/ipdr/web/  
# systemd will run this executable to start the service  
ExecStart=/home/bida/ipdr/web/Bida.Ipdr.Web --urls "http://*:8080"  
# to query logs using journalctl, set a logical name here SyslogIdentifier=ipdrSite  
  
# Use your username to keep things simple, for production scenario's I recommend a dedicated user/group.  
# If you pick a different user, make sure dotnet and all permissions are set correctly to run the app.  
# To update permissions, use 'chown yourusername -R /srv/AspNetSite' to take ownership of the folder and files,  
#       Use 'chmod +x /srv/AspNetSite/AspNetSite' to allow execution of the executable file.  
User=bida  
  
# ensure the service restarts after crashing  
Restart=always  
# amount of time to wait before restarting the service RestartSec=60  
  
# copied from dotnet documentation at  
# https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-3.1#code-try-7  
KillSignal=SIGINT  
Environment=ASPNETCORE_ENVIRONMENT=Production  
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false  
  
# This environment variable is necessary when dotnet isn't loaded for the specified user.  
# To figure out this value, run 'env | grep DOTNET_ROOT' when dotnet has been loaded into your shell.  
Environment=DOTNET_ROOT=/opt/rh/rh-dotnet31/root/usr/lib64/dotnet  
  
[Install]  
WantedBy=multi-user.target
```

> All described services and processes are guarded against OS reset.  
> All detail logs are preserve at `/home/bida/ipdr/app/logs`   
> Written with [StackEdit](https://stackedit.io/).  

