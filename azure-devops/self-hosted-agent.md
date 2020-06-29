# HOWTO Create a self-hosted Azure DevOps agent

## Requirements

You need to be administrator of the virtual machine (VM) that will be used to be the Azure DevOps agent.

### Azure VM

- Go to Azure Portal:
  - Create the new VM (for example Windows 2019 Datacenter or Ubuntu 18.04)
  - For Windows:
    - Update networking to only allow RDP on Ops location IP
    - Download RDP file

## Linux OS

### Installation steps on Linux

- SSH to the machine:

  ```
  # run updates
  sudo apt-get update
  
  # download the agent file
  wget https://vstsagentpackage.azureedge.net/agent/2.168.2/vsts-agent-linux-x64-2.168.2.tar.gz
  
  # uncompress the file
  tar zxvf ../vsts-agent-linux-x64-2.168.2.tar.gz
  
  # run the configuration
  ./config.sh
  ```

## Windows OS

### Installation steps on Windows

- Connect to the machine (RDP with `mstsc.exe`):
    - Run Windows Updates
    - Enable Windows Subsystem for Linux Feature
    - Download and install [Mozilla](https://www.mozilla.org/)
    - Download and install [Chrome](https://www.google.com/intl/en_us/chrome/)
    - Download and install [Notepad++](https://notepad-plus-plus.org/downloads/)
    - Download and install [git](https://git-scm.com/)
    - Download and install [.NET Core](https://dotnet.microsoft.com/download)
    - Download and install [7-zip](https://www.7-zip.org/download.html)
    - (Optional) Download and install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)
    - (Optional) Download and install [MongoDB CLI](https://www.mongodb.com/download-center/community), and:
      - Add bin directory in the PATH
    - Download and install [Visual Studio Build Tools](https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16)
    - Create `C:\Agent` folder and give all permissions to user `Network service` (used by Azure DevOps agent)
    - Download `chromedriver.exe` from [chromedriver.storage.googleapis.com](http://chromedriver.storage.googleapis.com/index.html) and extract it inside `C:\Agent`
      - Create a system environment variable `ChromeWebDriver` whose value is `C:\Agent\chromedriver_win32`
    - Download [psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) and extract it to `C:\Agent`
    - Follow [instuctions](https://docs.microsoft.com/en-us/windows/wsl/install-on-server)
      - Use `Ubuntu 18.04`
      - Rename to zip file, extract and move to `C:\Agent`
    - Open a command window, go to `C:\Agent\PSTools` and execute `psexec -i -u "nt authority\network service" cmd.exe`
      - In the new window go to `C:\Agent\CanonicalGroupLimited.Ubuntu18.04onWindows_1804.2018.XXX.YYY` and execute `ubuntu1804.exe` (see [Windows Server Installation Guide](https://docs.microsoft.com/en-us/windows/wsl/install-on-server) and [Manually download Windows Subsystem for Linux distro packages](https://docs.microsoft.com/en-us/windows/wsl/install-manual))
      - Execute bash
        - (Optional) Install [MongoDB](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/)

## Configuration steps on Windows

- Go to Azure DevOps, follow the [documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops#check-prerequisites)
  - Create a new Personal Access Token
  - Add pool from "https://dev.azure.com/yourorganization/_settings/agentpools"
  - Once create, click on New Agent
  - Copy agent zip file and extract it in `C:\Agent`
  - Download the agent file, unzip it and inside execute the command `config.cmd` (chose service)
  - Update agent capabilities
- (Optional) Go to external systems (such as MongoDB Atlas and Azure VM networking)
  - Add VM IP in system IP allowed list
