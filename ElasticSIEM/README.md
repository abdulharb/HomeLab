# Disclaimer

While i use a SIEM (that is not Elastic) at work and all that jazz, I've never need to do any of this manual setup. I'm trying to learn this and documenting how **I** did it. It might not be right or the best way, but this is me learning how you do stuff and learning Elastic along the way!


# Installing Elastic

I'll be doing this install on Ubuntu Server 24.04.
#### Some housekeeping 
I've add my ssh key when installing Ubuntu. So I don't want to type password for `sudo` commands.
```
echo "$USER ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee "/etc/sudoers.d/dont-prompt-$USER-for-sudo-password"
```

Update and random shenanigans (install qemu agent b/c I'm using Proxmox)
```
sudo apt update && sudo apt upgrade -y && sudo apt install qemu-guest-agent -y && sudo reboot
```
If you are not running Proxmox
```
sudo apt update && sudo apt upgrade -y && sudo reboot
```
Your Server will reboot!

#### Here starts the Meat

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```
```
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```
```
sudo apt update && sudo apt install elasticsearch
```

you'll get something like this:
```
Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : <PASSWORD_HERE>

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

### NOTE TO SELF: Might be something to automate. A password that you know and does not change?

Reset the password of the elastic built-in superuser with
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

```

###### Make sure to note your default elastic password!!! 
##### Let's start Elastic and enable it as a service
```
sudo systemctl daemon-reload && sudo systemctl enable elasticsearch.service && sudo systemctl start elasticsearch.service
```

#### Elastic are you running son?

Need to run it with `sudo` b/c cert file can only be accessed by root. NOTE: make sure to get the password from above 
```
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:<PASSWORD_HERE> https://localhost:9200
```
You should see something like this:
```
{
  "name" : "elk",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "pI6jMnUjQiOSsoQ-iEiarg",
  "version" : {
    "number" : "8.14.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "93a57a1a76f556d8aee6a90d1a95b06187501310",
    "build_date" : "2024-06-10T23:35:17.114581191Z",
    "build_snapshot" : false,
    "lucene_version" : "9.10.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
#### Some good things to know
Get Elastic's Status
```
sudo systemctl status elasticsearch.service
```
To start, stop or restart Elastic Search 
```
sudo systemctl [start|stop|restart] elasticsearch.service
```

# Kibana

All "one" command
```
sudo apt install kibana && kibanaToken=$(sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana) && sudo /usr/share/kibana/bin/kibana-setup --enrollment-token $kibanaToken && sudo systemctl daemon-reload && sudo systemctl enable kibana.service && sudo systemctl start kibana.service
```
# Logstash

I'm doing a bit of future proofing with this step, I don't think you'll need Logstash if you are only doing agent 
```
sudo apt install logstash && sudo systemctl enable logstash && sudo systemctl start logstash && systemctl --no-pager status logstash | grep Active
```

We are now done with **E**lastic Search, **L**ogStash and **K**ibana or our **ELK** stack! 

Now because I'm using Ubuntu server and I want to make my ELK stack accessible over the network I'm going to use Nginx as a reverse proxy for Kibana as Kibana is only accessible from the localhost. If you are using a Linux flavor that has a GUI (something like Ubuntu Desktop) and are okay with accessing it from that host only, you don't need to do the #Nginx section.
# Nginx

Install Nginx!
```
sudo apt install nginx apache2-utils -y
```

A good to know in VIM **_In short: `Esc`+`gg`+`dG` is what you need to do for deleting all the lines of the file in Vim._** (Thanks [Linux Handbook](https://linuxhandbook.com/delete-all-lines-vim/)!)

#### HTTP

```
sudo vim /etc/nginx/sites-available/default
```
```
server {
  listen 80;
  server_name <DNS_SERVER_NAME_HERE>; # Or you can remove this line if you don't have a DNS Entry for this Server

error_log   /var/log/nginx/kibana.error.log;
access_log  /var/log/nginx/kibana.access.log;

location / {
    rewrite ^/(.*) /$1 break;
    proxy_ignore_client_abort on;
    proxy_pass http://localhost:5601;
    proxy_set_header  X-Real-IP  $remote_addr;
    proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header  Host $http_host;

  }
}
```

Restart Nginx and you should be go to!
```
sudo systemctl restart nginx
```
Go to the server name or IP address and you should be G2G 
`http://<Server_IP>` or `http://<DNS_Name>`


Now for a bit more security:
#### HTTPS: With a self-signed certificate:
```
sudo mkdir -p /etc/nginx/ssl/
```
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/kibana.key -out /etc/nginx/ssl/kibana.crt
```
```
server {
    listen 80;
    server_name elk.local.al-harb.dev; # Replace elk.local.al-harb.dev with your local DNS name

    return 301 https://$server_name$request_uri;

    error_log  /var/log/nginx/kibana.error.log;
    access_log /var/log/nginx/kibana.access.log;
}

server {
    listen 443 ssl;
    server_name elk.local.al-harb.dev; # Replace elk.local.al-harb.dev with your local DNS name

    ssl_certificate /etc/nginx/ssl/kibana.crt;
    ssl_certificate_key /etc/nginx/ssl/kibana.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    error_log  /var/log/nginx/kibana.error.log;
    access_log /var/log/nginx/kibana.access.log;

    location / {
        rewrite ^/(.*) /$1 break;
        proxy_ignore_client_abort on;
        proxy_pass http://localhost:5601;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
    }
}
```

Now you should be able to access your ELK Server by the DNS/IP  `https://elk.local.al-harb.dev` in my case or `https://Server_IP`. Keep in mind that you'll get a "The certificate is not trusted because it is self signed" or "Your Connection is not private". You can safely ignore these warnings.

#### HTTPS with valid public Cert

So here is the thing, for this I did a bit of shenanigans. I did not want to expose the ELK Stack that i made on the internet so I fired up an Azure Server allowed HTTPS HTTP and SSH, pointed the DNS A record to the IP address of the server and used `certbot` to get the cert for `elk.local.al-harb.dev`. I'm not going to show you any of that but the [certbot website](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal) is great!

```
sudo su
```

Make sure to edit `elk.local.al-harb.dev` with your own domain. 
```
mkdir -p /etc/nginx/ssl/elk.local.al-harb.dev/
```
```
echo "<PUT THE WHOLE PUBLIC KEY CERT HERE>" > /etc/nginx/ssl/elk.local.al-harb.dev/fullchain.pem
```
```
echo "<PUT THE WHOLE PRIVATE KEY CERT HERE>" > /etc/nginx/ssl/elk.local.al-harb.dev/privkey.pem
```
Lets get out of the root user now!
```
exit
```

```
sudo vim /etc/nginx/sites-available/default
```

**_In short: `Esc`+`gg`+`dG` is what you need to do for deleting all the lines of the file in Vim._**

Now past this file!
```
server {
    listen 80;
    server_name elk.local.al-harb.dev; # Replace with actual DNS Name

    return 301 https://$server_name$request_uri;

    error_log  /var/log/nginx/kibana.error.log;
    access_log /var/log/nginx/kibana.access.log;
}

server {
    listen 443 ssl;
    server_name elk.local.al-harb.dev; # Replace with actual DNS Name

    ssl_certificate /etc/nginx/ssl/elk.local.al-harb.dev/fullchain.pem; # Replace with the localation of your Public key
    ssl_certificate_key /etc/nginx/ssl/elk.local.al-harb.dev/privkey.pem;# Replace with the localation of your Private key

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    error_log  /var/log/nginx/kibana.error.log;
    access_log /var/log/nginx/kibana.access.log;

    location / {
        rewrite ^/(.*) /$1 break;
        proxy_ignore_client_abort on;
        proxy_pass http://localhost:5601;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
    }
}
```

###### Test the config
```
sudo nginx -t
```
```
You should get something like this:
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

###### Restart nginx so that it takes the new Configs! 
```
sudo systemctl restart nginx
```


Now you should be able to access your ELK Server by the DNS record  `elk.local.al-harb.dev` in my case.

# Fleet Server Setup

We now need to setup a Fleet Server, and I'm going to set it up on the same server that I have ELK running on.

1. Go to `https://elk.al-harb.dev/app/fleet/agents` or `https://[URL]/app/fleet/agents`
	1. Use the username `elastic` and the password that you saved above!
		1. Oh you didn't safe the password? run `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic` and it should give you a new password.
2. Click `Add Fleet Server`!
	1. Give you fleet a name! `ELK Server` in my case
	2. Give it a Host URLs! `https://elk.local.al-harb.dev:8220` in my case. Note to self, make sure to add the 8220, if you do not it will go to 443 and I got Kibana on 443... not sure what happens if both live on 443, my expectations is that it won't work but hey man you do you!
	3. It gives you a script that you run places... again I ran it on the server that runs ELK!
Done? Profit???

# Continue enrolling Elastic Agent...

1. Make an Agent Policy. It will walk you though this after you add the Fleet server. (or go to `[URL]/app/fleet/policies?create`)
	1. I called mine `Windows Domain Joined` 
	2. It will give you a script to run. I want to run it via GPO
![[Pasted image 20240630165308.png]]

# Install via GPO

### PowerShell Script Setup
I made this script and call it `ELK-Install.ps1`. I put this script in `C:\Deployment Script`
```
$ProgressPreference = 'SilentlyContinue'

# Log file path
$logFile = "$env:TEMP\elastic-agent-installation.log"

# Function to write log messages
function Write-Log {
    param (
        [string]$message
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "$timestamp - $message"
    Write-Output $logMessage
    Add-Content -Path $logFile -Value $logMessage
}

# Download URL for the installer ##REPLACE
$installerUrl = "https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.14.1-windows-x86_64.zip"

# Enrollment token (replace with your actual token) ##REPLACE
$enrollmentToken = "XzRSOWE1QUI0Zm5rNVFlenV4bGM6QU1LdmRBTWlSQk9EcXREa2NqYlNhZw=="

# Fleet Server URL ##REPLACE
$fleetServer = "https://elk.local.al-harb.dev:8220"

# Temporary directory for extraction
$tempDir = "$env:TEMP\elastic-agent"

# Log start of script
Write-Log "Script started."

# Check if Elastic Agent service exists
$isInstalled = (Get-Service -Name "Elastic Agent" -ErrorAction SilentlyContinue) -ne $null

# Create temporary directory
New-Item -ItemType Directory -Path $tempDir -ErrorAction SilentlyContinue | Out-Null
Write-Log "Temporary directory created: $tempDir"

if (!$isInstalled) {
    Write-Log "Elastic Agent is not installed. Proceeding with installation."

    # Download installer
    try {
        Invoke-WebRequest -Uri $installerUrl -OutFile "$tempDir\elastic-agent.zip"
        Write-Log "Downloaded installer from $installerUrl to $tempDir\elastic-agent.zip"
    } catch {
        Write-Log "Error downloading installer: $_"
        throw $_
    }

    # Extract installer (modify arguments if needed)
    try {
        Expand-Archive -Path "$tempDir\elastic-agent.zip" -DestinationPath $tempDir
        Write-Log "Extracted installer to $tempDir"
    } catch {
        Write-Log "Error extracting installer: $_"
        throw $_
    }

    # Install agent with URL and token
    try {
        Start-Process -FilePath "$tempDir\elastic-agent*\elastic-agent.exe" -ArgumentList "install --non-interactive --insecure --url=$fleetServer --enrollment-token=$enrollmentToken" -Wait -NoNewWindow
        Write-Log "Elastic Agent installation started."
    } catch {
        Write-Log "Error starting Elastic Agent installation: $_"
        throw $_
    }
} else {
    # Write message indicating agent is already installed
    Write-Log "Elastic Agent is already installed."
}

# Cleanup temporary directory
try {
    Remove-Item -Path $tempDir -Recurse -Force
    Write-Log "Temporary directory removed: $tempDir"
} catch {
    Write-Log "Error removing temporary directory: $_"
}

# Log end of script
Write-Log "Script completed."
```
P.S. I'm using --insecure b/c "Remote Elastic Agents enrolling into a Fleet Server with self-signed certificates must specify the `--insecure` flag" Elastic Docs

###### Make sure to edit the variables below with the values that you get from the Elastic script :
1. `installerUrl`
2. `enrollmentToken`
3. `fleetServer`

##### Share the folder!
Right click on the the folder -->  `Give access to` --> `specific people...` --> Share it with `Everyone` to make things easier!
Make sure to save the file path `\\DC01\Deployment Script` in my case

### GPO TIME!!

1. Open `Group Policy Management`
	1. Windows Key + R --> type `gpmc.msc`
2. Go to `Forest: [DOMAIN]` --> `Domains` --> `[Domain]` --> `[Group Policy Objects]`
	1. You should be here:![[Pasted image 20240630170647.png]]
		p.s. I got those other GPO's from Black Hills Info Sec's [GitHub](https://github.com/blackhillsinfosec/EventLogging)! I might need to do a start a Lab from scratch write up... but later™... 
3. Right click on the folder  `Group Policy Objects` and click `New`
	1. Give it a name, I called mine `Install ELK Agent`
4. Right click on the GPO that you just made and click `Edit`. (This is in the folder `Group Policy Objects`)
5. Go to `Computer Configuration` --> `Preferences` --> `Control Panel Setting` --> `Scheduled Tasks` --> `New` --> `Scheduled Task (at least windows 7)`
6. `General` Tab
	1. Give it a name
	2. Run it with the system account
		1. Click `Change User or Group`
		2. type `system`
		3. click `OK`
		4. You should see the user as `NT AUTHORITY\System` now.
	3. check `Run wheather user is logged on or not`
	4. check `Run with highest privileges`
7. `Triggers` Tab
	1. New
	2. For "`Begin the Task`" chose `At startup`. 
		1. Note, for me it's easy to reboot all my servers at once, so I picked what's easiest for me. If something else makes more sense in your case please do that!
	3. Done!
8. `Actions` Tab:
	1. New
	2. Keep on `Start a program`
	3. Program/script: `powershell.exe`
	4. Add Arguments `-executionpolicy bypass -File "PAHT"`  Give it the file share that we made
		1. `-executionpolicy bypass -File "\\DC01\Deployment Script\ELK-Install.ps1"` in my case
	5. Click `OK`
9. `Settings` Tab
	1. Check `Allow Task to be run on demand` (in case you need to test things out or run it manually)
10. Click `Apply` and `OK`!
11. Close `Group Policy Management Editor`
12. Now time to Link the GPO
	1. You should still have `Group Policy Management` open, if you do not do step 1 and come back here.
	2. Right click on your domain (`Forest: [DOMAIN]` --> `Domains` --> `[Domain]`<-- This one )
	3. Click `Link an Existing GPO...` 
	4. Click the GPO you made, `Install ELK Agent` in my case
13. To make things easy for me, I'm going to add everyone to the GPO 
	![[Pasted image 20240630190924.png]]

Now I'm going to force a GPO Update with `gpupdate /force` (GPO by default update every 90 to 120 minutes so you could wait and things should get pushed out, I'm just an impatient little girl) and reboot all my servers/workstations BUT the DC and we should see agents start to show up in our elastic Dashboard. 
If you go to `[URL]/app/fleet/agents` you should get see all the agents you have installed! (5 Windows Servers/Workstations in my case + a Fleet Server)

After about 5 min I give the DC a reboot.

You can look at `C:\Windows\Temp\elastic-agent-installation.log` to see if you run into any issues with the install.

# Elastic Agent Setup

Now time to do some fun stuff!!

Go to `Agent policies` here `https://[URL]/app/fleet/policies`  and click on the policy you want to edit `Windows Domain Joined` in my case. Click `Add integration` and add `Elastic Defend`.

Give it a name, I'm going to use `Defend integration` (imaginative I know...) and I'm going to leave everything the same. (make sure that `Agent policy` is set to your agent policy `Windows Domain Joined` in my case)

Now b/c i want to be extra... and get way to much information... I'm going to collect Windows PowerShell and Sysmon (p.s. I had to install Sysmon and enable some extra logging for PowerShell, the Elastic Agent _should_ be getting most of this info for you. But again I like to be extra.)


The  Windows `Application`, `Security` and `System` should be automatically collected. you can verify this by going to `[URL]/app/fleet/policies` then finding your policy `Windows Domain Joined` in my case, going to `system` in the integrations and you should see `Collect events from the Windows event log` as enabled.


Now Sysmon and PowerShell!

So let's go to `[URL]/app/integrations/browse` and search for Windows. You'll see an Integration called `Windows`...

Let's add that!

For now I'm going to make sure `PowerShell`, `PowerShell Operational` and `Sysmon Operational` are enabled. I'm going to disable `Forwarded` as non of my systems are WEC (Windows Event Collectors).

(an extra things that I'm doing is enabling `AppLocker/EXE and DLL`, `AppLocker/MSI and Script`, `Packaged app-Deployment` and `Packaged app-Execution` for shits and giggles... tbh I'm not sure what they do but it's going to be fun figuring that stuff out!!)

Make sure I'm adding this integration to my existing `Windows Domain Joined` policy!

And now BOOM we should be done!

If you want to give it an extra bit of ✨spice✨ I'm going to start a 30 day trial of Elastic Plati­num (?) to get some advanced features (you can go [here](https://www.elastic.co/subscriptions) and see what you would be getting, tbh not if it's worth it but I'm going to do it anyway)

Going back to Fleet management `[URL]/app/fleet/policies` I'm going to make sure everything is enabled when it comes to the premium features. Click on the policy that you are using, (`Windows Domain Joined` in my case) then `Defend integration` then scroll down to enable everything! 

`Ransomware protections`? SURE!
`Memory threat protections`? Why the hell not!
`Malicious behavior protections`? okay!
`Attack surface reduction`? Yes plz!

And one last thing that I'm going to do is enable `Register as antivirus`. Save and we are done!!

Now I'm going to give things a couple of minutes to make sure everything syncs and that I start getting data. 

Now I'm going to login to one of the servers and run a command in PowerShell to make sure everything is coming to my ELK stack.

Open PowerShell and type something `Echo "Hello Mom"`. Go to `[URL]/app/discover` and just search `Hello Mom` and you should see results!

kk and with that, we are done here! You should have a fully working ELK stack with Elastic Agents collecting Windows Event logs, Sysmon and PowerShell logs. 


Good luck everyone (me include)!

## Acknowledgments
https://simplifyingtechcode.wordpress.com/2024/03/15/how-to-install-and-configure-elasticsearch-kibana-logstash-8-12-version-on-ubuntu-linux-2024/

https://medium.com/@musmanqureshi109/installation-steps-of-elk-stack-with-nginx-server-2a1961812f72

https://v2cloud.com/tutorials/how-to-configure-scripts-with-gpos-in-windows-server
