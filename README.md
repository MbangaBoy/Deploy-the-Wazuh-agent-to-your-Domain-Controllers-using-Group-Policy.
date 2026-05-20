# Deploy-the-Wazuh-agent-to-your-Domain-Controllers-using-Group-Policy.
Step-by-step guide to deploy the Wazuh agent to your Domain Controllers using Group Policy.

🚀 Deploying Wazuh Agent via GPO – Step-by-Step
Overview
You can push the Wazuh agent silently to all Domain Controllers by creating a Group Policy Object that installs the agent at the next system reboot. This method uses Software Installation (MSI assignment) and optionally a startup script to apply a custom configuration.

📋 Prerequisites
A network share accessible by Domain Computers (at least Read permission)

The Wazuh agent MSI installer (download from wazuh.com – Windows 64‑bit recommended)

Your Wazuh manager IP or hostname

Administrative access to create and link GPOs

📦 Step 1: Prepare the Installation Files
Create a folder on your file server (e.g. E:\WazuhDeploy) and structure it like this:

text
E:\WazuhDeploy
├── WazuhAgent.msi          # the agent installer
├── ossec.conf              # pre-configured agent configuration (optional)
└── InstallWazuh.bat        # startup script (alternative method)
If you use a startup script, you can pass the manager address directly:

batch
@echo off
msiexec /i "\\YourServer\WazuhDeploy\WazuhAgent.msi" /qn WAZUH_MANAGER="192.168.1.100" WAZUH_REGISTRATION_SERVER="192.168.1.100"
(The /qn switch makes it fully silent.)

🔗 Step 2: Create the Network Share
Right-click the folder WazuhDeploy → Properties → Sharing tab → Advanced Sharing.

Check Share this folder, name it WazuhDeploy$ (the $ hides it from casual browsing).

Permissions → add Domain Computers with Read access (or Authenticated Users if preferred).

Note the UNC path: \\YourServer\WazuhDeploy$.

Make sure the MSI and any scripts are also readable.

🛠️ Step 3: Create the Group Policy Object
Open Group Policy Management Console (gpmc.msc).

Right-click Group Policy Objects → New.

Name it, e.g., Deploy Wazuh Agent – DCs.

💻 Step 4: Assign the MSI (Computer Configuration)
This method installs the agent automatically at boot, before the user logs on.

Right-click your new GPO → Edit.

Go to:
Computer Configuration → Policies → Software Settings → Software installation.

Right-click the right pane → New → Package.

In the file dialog, type the UNC path to the MSI:
\\YourServer\WazuhDeploy$\WazuhAgent.msi
(Do not browse by drive letter – always use UNC for network installs.)

Select Assigned as the deployment method.

Click OK.

The package appears. The agent will be installed at the next reboot.

⚠️ Important: This method installs the agent with default settings (it tries to connect to wazuh-manager). To point it to your real manager, you must overwrite ossec.conf after the installation.

🔧 Step 5: Deliver a Custom ossec.conf (Post‑Install)
Use a Startup Script (executed after the MSI installation) to copy a pre‑configured ossec.conf into the agent's directory.

Option A – Copy the file via script

In the same GPO, go to:
Computer Configuration → Policies → Windows Settings → Scripts (Startup/Shutdown).

Double-click Startup → Add → Browse.

Copy this small batch file into the opened Startup folder (or any network location, but local is safer):

Copy-WazuhConfig.bat

batch
xcopy /Y "\\YourServer\WazuhDeploy$\ossec.conf" "C:\Program Files (x86)\ossec-agent\ossec.conf*"
net stop WazuhSvc
net start WazuhSvc
(Adjust the installation path if you used a custom one.)

Click OK twice.

Option B – Use PowerShell to modify the config

Instead of copying a whole file, you can set the manager address with a one-liner in a PowerShell startup script:

powershell
(Get-Content "C:\Program Files (x86)\ossec-agent\ossec.conf") -replace '<address>wazuh-manager</address>', '<address>192.168.1.100</address>' | Set-Content "C:\Program Files (x86)\ossec-agent\ossec.conf"
Restart-Service WazuhSvc
🔁 Step 6: Link the GPO to the Domain Controllers OU
In Group Policy Management, locate the Domain Controllers organizational unit.

Right-click the OU → Link an Existing GPO.

Choose Deploy Wazuh Agent – DCs → OK.

This ensures only your Domain Controllers receive the installation.

🧪 Step 7: Test and Verify
On a Domain Controller, run:

cmd
gpupdate /force
Reboot the server (or schedule a maintenance window).
If you used Assigned Software, the installation starts at boot (“Please wait for the software installation…”).

Once logged in, check:

The Wazuh service is running (services.msc)

The agent is connected to your manager (check in the Wazuh dashboard)

The ossec.conf contains the correct manager address

⚙️ Alternative: Startup Script Only (No MSI Assignment)
If you prefer a single script that does everything (install + configure), skip the Software Installation step and only use a startup script. Example InstallWazuh.bat:

batch
if exist "C:\Program Files (x86)\ossec-agent\ossec.conf" goto :EOF
msiexec /i "\\YourServer\WazuhDeploy$\WazuhAgent.msi" /qn WAZUH_MANAGER="192.168.1.100" WAZUH_REGISTRATION_SERVER="192.168.1.100"
Place it in the GPO’s Startup Scripts. This method avoids the need for a separate config copy, because you pass the manager during install.

🧰 Tips for Success
Reboot required – The MSI installation triggers only at system startup, so plan a maintenance window.

Agent version upgrades – To update later, prepare a new MSI and either replace the package in the GPO (with “Upgrade” option) or use a dedicated startup script that uninstalls the old version first.

Troubleshooting – Check the Windows Application Event Log for MsiInstaller entries. Script errors can be found in %SystemRoot%\debug\mylog.txt if you redirect output in your batch files.
