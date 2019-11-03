---
layout: post
title: "Step by step: Windows Virtual Desktop and FSLogix"
categories: [Windows Virtual Desktop,FSLogix,Azure]
---

21 March 2019 the Windows Virtual Desktop preview went live. My colleague [Jan Bakker](https://www.linkedin.com/in/jan-bakker/) and myself went strait to all available documentation, and build a testenvironment together.
In this blogpost we will show you how to setup a Windows Virtual Desktop (WVD) environment, and what to watch out for. As a bonus we will also show how to install and configure FSLogix.
This tuturial will be a step by step guide to setup a complete environment starting from zero.

**What is Windows Virtual Desktop?**

Windows Virtual Desktop is a new service from Microsoft and enables you to deliver a virtual desktop from the Azure cloud.
This could be a multi-user Windows 10 desktop, but also Windows 7 (with extended support) is possible.
It also gives you the ability to install Office 365 ProPlus on the virtual desktops.
Windows Virtual Desktop is a schalable service to deploy virtual machines, made possible by Azure resources, storage and advanced networking. 
The back-end like RDS brokers, gateways, web access, databases, and diagnostics are hosted and managed by Microsoft.


Requirements
============

To use Windows Virtual Desktop there are some requirements:

-   Active Directory Domain Services

-   An Azure tenant

-   An Azure subscription with credits.

The virtual machines that are deployed have to be domain joined, or be hybrid domain joined. Azure AD join is not (yet?) possible. So Windows Virtual Desktop is dependand upon an Active Directory domain. The VM's will be joined to this domain, and because of AD Connect you can login with your own Azure AD credentials.
There are 3 options for Active Directory Domain Services:

1.  Install / Enable **Azure Active Directory Domain Services**

2.  Install the Active Directory Domain Services role on a server in Azure

3.  Connect your own on-premises network with Active Directory to your Azure tenant with a Site-2-Site VPN
    or ExpressRoute

In this blog we chose option 2: We install a Server 2019 VM and install the ADDS role.

Create azure tenant & subscription
=====================================

In this blog we start from zero. We'll show you how to deploy Windows Virtual desktop from zero. So we start with requesting a 30-days Azure trial subscription. 
You can request your trial here:
[https://azure.microsoft.com/nl-nl/free/](https://azure.microsoft.com/nl-nl/free/)

Follow the steps to create the trial subscription. You'll also need a creditcard. No worries, this is only for verification, no costs will be billed.

Because we will need an internet routable domainname, we will add this right away to our Azure AD tenant. This domainname we will also use for our Active Directory domainname.
This isn't necessary, but it will make sure we have an easier domainname for our Azure AD users.

We'lll use the domainname: *resultaatgroep.eu*

> HINT: You can also use the standard extension: tenantname.onmicrosoft.com
Your own domain is optional.

![]({{ site.baseurl }}/images/WVDandFSLogix/dad42e1331444e5d6baa70989996d0d4.png)

As soon as we added the custom domainname, we have to create a TXT record in DNS to validate the domain, and after that we can start creating users with the suffix: @resultaatgroep.eu

![]({{ site.baseurl }}/images/WVDandFSLogix/b86bbbc391db5623c6f4a8d5842d08be.png)


Installing Active Directory Domain Services
============================================

To deploy Active Directory we first need a server in our subsciption. We'll also need a network to which the server can connect to. However, we can create the network during the VM creation wizard.

To to: “**Virtual Machines**”, and click on “**+ Add**”

![]({{ site.baseurl }}/images/WVDandFSLogix/9df79137418769e20dd4ef56d5698ea1.png)

Then follow the steps to deploy a Windows Server 2019 Datacenter VM.

Create during the wizard also an extra datadisk. We'll need the disk to create the Active Directory Domain Services database on.

![]({{ site.baseurl }}/images/WVDandFSLogix/c6f05f1923d3167eb11af2fcb722eca0.png)

> **NOTE**: we give our VM a *public IP adres*, this is not recommended in a production environment.

If you do this, make sure you apply a Network Security Group that allows only RDP connections from your own WAN address. You could optionally also create a Point-2-Site VPN connection

For more information: <https://docs.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-portal>

![]({{ site.baseurl }}/images/WVDandFSLogix/580b8606ff6a6a24025db8908e995a5d.png)

When your VM is done, go to “Virtual Machines” --> [VM naam] and note the private IP address.
By default the vNet in Azure will give give the Azure DNS servers to servers. As we will install Active Directory Domain Services & DNS on our VM, we can change the DNS server of the vNet already to the IP address of our VM.

Go to **Virtual Networks** and click on your vNet. At the section DNS servers you can fill out your own DNS server, use the private IP address of your VM that will host the DNS server role.
![]({{ site.baseurl }}/images/WVDandFSLogix/vnet-dns.png)

When this is done, go to your VM again and click on **Connect** to download the RDP file, and login to your VM

![]({{ site.baseurl }}/images/WVDandFSLogix/05ca1b2cdb4a19a0b9bc0a9ce288bb86.png)

Next you can install Active Directory Domain Services. In our demo we'll use PowerShell for this.
As a best-practice, make sure all data is on a datadisk, and not on an OS disk. This also goes for the AD database, log directory and sysvol directory. 
Change the values to your own domain, and check if the datadisk (here with F:\\) is visible

 ```
$securestring = ConvertTo-SecureString -AsPlainText -Force -String *********
Install-windowsfeature -name AD-Domain-Services -IncludeManagementTools

$parameterSplat = @{
    SysvolPath = "F:\\SYSVOL"
    LogPath = "F:\\NTDS"
    DatabasePath = "F:\\NTDS"
    DomainNetbiosName = "RESULTAATGROEP"
    DomainName = "resultaatgroep.eu"
    Force = $true
    SafeModeAdministratorPassword = $securestring
}
Install-ADDSForest @parameterSplat
```

Create OU for the Windows Virtual Desktop VM’s
------------------------------------------------

Now Active Directory has been installed and ready for use we can create a couple of OU's that will house our WVD servers, and our users.
We will also do this with PowerShell, but you can also do this with the GUI.

```powershell
New-ADOrganizationalUnit -Name ResultaatGroep -Path "DC=resultaatgroep,DC=eu"
New-ADOrganizationalUnit -Name Groups -Path "OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
New-ADOrganizationalUnit -Name Users -Path "OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
New-ADOrganizationalUnit -Name Servers -Path "OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
New-ADOrganizationalUnit -Name "Windows Virtual Desktops" -Path "OU=Servers,OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
```

Creating testaccounts
---------------------

To login at our Virtuele Desktops, we create a couple of testaccounts. You'll need these before you create the WVD hostpool, as you need to supply users during the wizard.

```powershell
$newADUserSplat = @{
    Path = "OU=Users,OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
    ChangePasswordAtLogon = $false
    UserPrincipalName = "jan@resultaatgroep.eu"
    GivenName = "Jan"
    AccountPassword = (ConvertTo-SecureString -AsPlainText -Force -String "******")
    SamAccountName = "jan"
    DisplayName = "Jan Bakker"
    Name = "Jan Bakker"
    Enabled = $true
    Surname = "Bakker"
}
New-ADUser @newADUserSplat

$newADUserSplat2 = @{
    Path = "OU=Users,OU=ResultaatGroep,DC=resultaatgroep,DC=eu"
    ChangePasswordAtLogon = $false
    UserPrincipalName = "roel@resultaatgroep.eu"
    GivenName = "Roel"
    AccountPassword = (ConvertTo-SecureString -AsPlainText -Force -String "******")
    SamAccountName = "roel"
    DisplayName = "Roel Everink"
    Name = "Roel Everink"
    Enabled = $true
    Surname = "Everink"
}
New-ADUser @newADUserSplat2
```
![]({{ site.baseurl }}/images/WVDandFSLogix/efe3ba3550f1f6d633c26a6d8de05e84.png)

Installing Azure AD Connect
============================

Next we will install Azure AD Connect to synchronize our Active Directory with Azure Active Directory.

You can download the install file here:
<https://www.microsoft.com/en-us/download/details.aspx?id=47594>

To save cost, we will install this on our Domain Controller.

Download the installfile and execute it, wait till the wizard starts.

![]({{ site.baseurl }}/images/WVDandFSLogix/774022dd18b627126d7e4f2e9590e4de.png)

We choose for **“Use express settings”**. You can pick Customize if needed for OU filtering or other settings. For our demo this isn't important.

During the install you will be asked for Azure AD credentials and ADDS credentials

For Azure AD you'll need Global Admin rights, and for ADDS you'll need an Enterprise Administrator.

After the install the synchronize will start automatically. You can check in your Azure AD tenant if the sync was succesfull.

![]({{ site.baseurl }}/images/WVDandFSLogix/2155f10fc34a467c3df7647859931c14.png)

Configure Windows Virtual Desktop
====================================

To use Windows Virtual Desktop (WVD) there are a few steps to be taken to give the back-end service and web client rights to your Azure AD tenant.

The WVD back-end and client app are multi-tenant enviroments which are managed by Microsoft. That means that, by default, the WVD environment has no rights to our Azure tenant. There is also no WVD tenant for us yet.

So first we have to create the WVD tenant, and give rights to our Azure tenant.

Assigning rights to the WVD back-end & client app
--------------------------------------------------------

to go the site: <https://rdweb.wvd.microsoft.com/>

first, select here “**Server App**” and fill in your Tenant ID.

Then wait about 30 seconds, and select: “**Client App**” with the same Tenant ID.

You can find your Tenant ID at the following location:

“**Azure Active Directory**” “**Properties**” “**Directory ID**”

![]({{ site.baseurl }}/images/WVDandFSLogix/51b183af0958f2334daf4c1401614297.png)

![]({{ site.baseurl }}/images/WVDandFSLogix/5ed9096e694d192581cb28db97c88e04.png)

Creating the WVD tenant
--------------------------

Creating the tenant is done on the server back-end, which we just gave rights to our Azure tenant.

We just have to give a user (or service principal) rights to create the actual WVD tenant. Go in the Azure portal to: “**Azure Active Directory**” -->
“**Enterprise Application**”

In the list are 2 applications: “Windows Virtual Desktop” & “Windows Virtual
Desktop Client”

These have been created because we gave them access to our Azure tenant in the previous step.

Click on:“**Windows Virtual Desktop**” --> “**Users and groups**”
and click on: “**Add user**”

At role there is only one option and its preselected: TenantCreator. Select one or multiple users who can create a tenant.
And click on “**Assign**”

![]({{ site.baseurl }}/images/WVDandFSLogix/725c15d4af3e4d71210fe3a24c5bb4c6.png)

Create the tenant with the following commands: The AadTenantId we already have from the previous steps

You can find the subscriptionId in the portal: “**All Services**” --> “**Subscriptions**” --> “**Subscription ID**”

![]({{ site.baseurl }}/images/WVDandFSLogix/39c622efa9dc890fd89619ccb6431951.png)

Because we will login at the back-end, we can also create a service principal which has rights to create host pools in WVD, and to add servers to the hostpools

You can use the service principal because later during the WVD wizard you'll use an account to create the host pools. And MFA enabled accounts aren't supported.
If you use a regular user account that isn't MFA enabled you can skip creating the service principal.

Start PowerShell as administrator (to install modules) and execute the following commands (substitute variables for your environment):
```powershell
Install-Module -Name Microsoft.RDInfra.RDPowerShell, AzureAD
Import-Module -Name Microsoft.RDInfra.RDPowerShell, AzureAD

#Setup variables, tenantName will be the name of our WVD tenant
$tenantName = "ResultaatGroep-WVD"
$AadTenantId = "11111111-2222-3333-4444-555555555555"
$subscriptionId = "11111111-2222-3333-4444-555555555555"

#Login at the WVD back-end (with the account that has TenantCreator rights)
Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com"

#Creating the RDS tenant
New-RdsTenant -Name \$tenantName -AadTenantId \$AadTenantId -AzureSubscriptionId $subscriptionId

#Login to Azure AD with a global admin account to create a service principal
$aadContext = Connect-AzureAD

#Creating theservice principal with credentials
$svcPrincipal = New-AzureADApplication -AvailableToOtherTenants \$true -DisplayName "Windows Virtual Desktop Svc Principal"
$svcPrincipalCreds = New-AzureADApplicationPasswordCredential -ObjectId $svcPrincipal.ObjectId

#Assign rights to the service principal
New-RdsRoleAssignment -RoleDefinitionName "RDS Owner" -ApplicationId $svcPrincipal.AppId -TenantGroupName "Default Tenant Group" -TenantName $tenantName

#Make sure the credentials of the sevice principal are saved, you can see these with:
$svcPrincipal.AppId
$svcPrincipalCreds.Value
```
You'll need the password and AppId in the next step.

>**Note**: The next step is creating the hostpool, if you want to test FSLogix and Office 365 on your VM's right away, then take the nessicary steps from chapter 7 first. There we will create install scripts for Office and the FSLogix agent, and we'll configure it with Azure Blob storage. Offcourse, these steps can also be done afterwards.

Creating the hostpool
---------------------

And now the creation of the Windows Virtual Desktop Hostpool. This is done from the Azure portal.

Choose: **+ Create a resource** and search for Windows Virtual Desktop. Choose for: **Windows Virtual Desktop – Provision a host pool**

![]({{ site.baseurl }}/images/WVDandFSLogix/3a8ce9025f83cf55825e2782e212739c.png)

Then choose for **Create**

![]({{ site.baseurl }}/images/WVDandFSLogix/6f4a8279c4b20b2405910bb2f117a3b5.png)

![]({{ site.baseurl }}/images/WVDandFSLogix/8ba3733c121cc22a90901b979ea7971c.png)

Choose a Poolname, and pick your type. We go for Pooled, but you can also create peronal VM's

Then fill in your testusers that you made earlier. These users will be linked to this pool and can use its resources.

Fill in the Resource Group op. It has to be **empty**.

Then you assign the location.

![]({{ site.baseurl }}/images/WVDandFSLogix/4eba35378b8b543a4f8b8b0eb1dbb593.png)

In this windows we will determine the amount and type of VM. You can choose at will, for our test 1 VM of type D2s_v3 will suffice.

Also fill in the prefix for your VM's.

More information about scaling and pricing can be found [here](https://azure.microsoft.com/en-us/pricing/details/virtual-desktop/).

![]({{ site.baseurl }}/images/WVDandFSLogix/021cf3afac111280beca8aef5216b83d.png)

Here we will determine the OS verion, domain and the vNet

We choose for the standard Gallery source**Windows 10 Enterprise multi-session**.

We also fill in the info to join our VM to the domain: **resultaatgroep.eu**.

>Note: the vNet you select here has to be able to reach your domain controller

![]({{ site.baseurl }}/images/WVDandFSLogix/a69da8f041fa4489a407c8ed85040762.png)

In this step you fill out your tenant group name and tenant name. You created this earlier and has to match.

Also fill out your RDS Owner, this could be a (non MFA enabled) users, but in our case we use the Service Principal that we created earlier.

We also fill out our Azure AD tenant ID.

![]({{ site.baseurl }}/images/WVDandFSLogix/f7fec9917295bacd327a870251d24cd3.png)

In step 5 there's a validation and summary of our steps, check this one final time and choose **Next** to finish the wizard

![]({{ site.baseurl }}/images/WVDandFSLogix/29559421e48c419520c2eb055c153731.png)

You can follow the status of your deployment under **Deployments** while inside your Resource Group.

At the moment of writing this, it is not yet possible to give a group rights to a hostpool, only users can be granted rights to a hostpool

Connecting to Windows Virtual Desktop
=====================================

You can connecto to Windows Virtual Desktop in 2 ways, will we show both.

HTML 5 browser
--------------

The easiest method is the browser. Go to <https://rdweb.wvd.microsoft.com/webclient> and login with your testuser.

![]({{ site.baseurl }}/images/WVDandFSLogix/340018115e59ba74082467c6a1bd8055.png)

![]({{ site.baseurl }}/images/WVDandFSLogix/e1fd395735f5023914334f253d0d2569.png)

Unfortunatly SSO doesn't work yet, that means that you have to login to Azure AD first, and after selecting your desktop or application, you have to login to the server.

With ADFS it is possible to create an SSO experience.

Windows client – Remote Desktop
-------------------------------

You can also download the Windows client. The benefit is that it performs a bit better, and you get start menu integration.

You can download the client [here](https://go.microsoft.com/fwlink/?linkid=2068602)

After the install you can choose **Subscribe.** then login with the testuser.
testgebruiker.

![]({{ site.baseurl }}/images/WVDandFSLogix/28df9e185348c484b3c9df6810eafaf0.png)

**Optional:** Virtual Desktop VM’s & FSLogix
=============================================

We willen je ook graag laten zien hoe je FSLogix kunt installeren en
configureren. Dit is een optionele stap, maar zeker aan te raden om hier mee aan
de slag te gaan. Met FSLogix kun je het profiel, en daarmee ook de
Outlook/Onedrive cache en de zoekindex opslaan op een aparte VHDX die aan je
sessie gekoppeld wordt wanneer de gebruikt inlogt.

De VHDX staat normaalgesproken op een SMB fileshare op een server. Echter
ondersteund FSLogix ook een Azure page block storage! Aangezien we hiervoor geen
fileserver nodig hebben die we moeten onderhouden, en compute kost, is dit
natuurlijk de optie wie we gaan gebruiken!

We raden je aan ook eerst even in [de FSLogix
documentatie](https://docs.fslogix.com/) te duiken als je hier nog niet bekend
mee bent. Hiermee krijg je snel een beeld van hoe FSLogix werkt, en weet je ook
gelijk welke mogelijkheden er nog meer zijn.

*Voor alle volgende stappen geldt: deze instellingen zijn gezet voor onze demo
omgeving. Sommige instellingen zijn wellicht af te raden voor
productieomgevingen.*

Storage account voor de Profile Containers
------------------------------------------

We beginnen met het aanmaken het Azure Storage Account. Hier komen de FSLogix
VHDX bestanden op te staan. Gebruik je liever een SMB share, kan ook. Maak dan
een fileshare aan volgens deze instructies:
<https://docs.fslogix.com/display/20170529/Requirements+-+Profile+Containers>

Ga naar de Azure portal om het storage account aan te maken. Klik op “**Create a
resource**” zoek naar: “**Storage account**” en klik op “**Create**”

Volg vervolgens de wizard om alle velden in te vullen. Wij kiezen voor premium
storage, zodat de gebruikerservaring zo snel mogelijk is.

Verder staan we alleen toegang toe vanaf het subnet waarin onze Virtual Desktops
komen. Dit zorgt er ook voor dat er ook een [Storage
Endpoint](https://docs.microsoft.com/nl-nl/azure/virtual-network/virtual-network-service-endpoints-overview)
aan dit subnet wordt toegevoegd, zodat de latency naar de storage zo laag
mogelijk is. De volledige settings staan hieronder in de screenshot.

![]({{ site.baseurl }}/images/WVDandFSLogix/bdf34c4d1a71d3cc8a546d53f109154a.png)

GPO – FSLogix Agent installeren
-------------------------------

Om de FSLogix agent te installeren gebruiken we een Group Policy nodig die deze
installeert. De software kan gedownload worden vanaf
<https://go.microsoft.com/fwlink/?linkid=2084562>

Uit deze ZIP heb je *FSLogixAppsSetup.exe* nodig, en de *TXT file* met de
licentie. Ook hebben we later het ADMX en ADML bestand nodig om een GPO te maken
voor de agent-instellingen.

Maak een nieuwe GPO onder de OU die je eerder aangemaakt hebt. Kies voor
Computer Configuration -\> Windows Settings -\> Scripts -\> Startup. Kies voor
Show Files en plaats de EXE in de folder.

![]({{ site.baseurl }}/images/WVDandFSLogix/f5ad12294b84029a5e33df7c60690bc3.png)

Klik vervolgens op Add en voeg de EXE toe. Vul in het parameter veld de volgende
gegevens in : */silent ProductKey=MSFT0-YXKIX-NVQI4-I6WIA-O4TXE*

![]({{ site.baseurl }}/images/WVDandFSLogix/7ddeecd1d53aee351fd57af055159b64.png)

ADMX bestanden importeren
-------------------------

Kopieer het bestand “fslogix.admx” bestand (uit de ZIP file) naar de
C:\\Windows\\PolicyDefinitions map op je domaincontroller. Kopieer het
“fslogix.adml” bestand naar de C:\\Windows\\PolicyDefinitions\\en-us\\ folder.

GPO – FSLogix Agent instellingen
--------------------------------

Om FSLogix te configureren maken we ook een Group Policy. Onder Computer
Configuration -\> Policies -\> Adminstrative Templates kun je de FSLogix
instellingen terug vinden.

Configureer ten minste de volgende instellingen:

| **Computer Configuration -\> Policies -\> Administrative Templates -\> FSLogix/Profile Containers**                                |                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| Enabled                                                                                                                            | Enabled                                                             |
| Size in MB’s                                                                                                                       | 25600                                                               |
| Delete local profile when FSLogix Profile should apply                                                                             | Enabled (wees voorzichtig met deze setting in productie omgevingen) |
| **Computer Configuration -\> Policies -\> Administrative Templates -\> FSLogix/Profile Containers/Advanced**                       |                                                                     |
| Locked VHD retry count                                                                                                             | 1                                                                   |
| Locked VHD retry interval                                                                                                          | 0                                                                   |
| **Computer Configuration -\> Policies -\> Administrative Templates -\> FSLogix/Profile Containers/Cloud Cache**                    |                                                                     |
| Cloud Cache Locations \*                                                                                                           | type=azure,connectionString="XXXXXXX"                               |
| **Computer Configuration -\> Policies -\> Administrative Templates -\> FSLogix/Profile Containers/Container and Directory Naming** |                                                                     |
| SID directory name matching string                                                                                                 | %userdomain%-%username%                                             |
| SID directory name pattern string                                                                                                  | %userdomain%-%username%                                             |
| Virtual disk type                                                                                                                  | VHDX                                                                |

\* Vervang bij de waarde de XXXXXX tekens voor je persoonlijke connection-string
van het strorage account dat je eerder gemaakt hebt in stap 1.1

![]({{ site.baseurl }}/images/WVDandFSLogix/ed011d9883f2af177a754cd9046de0c3.png)

Include en Exclude Groepen aanmaken (Optioneel)
-----------------------------------------------

Standaard worden er 4 lokale groepen aangemaakt waardoor FSLogix voor iedereen
wordt aangezet. Om te bepalen voor wie je Profile Container wilt aanzetten en
voor wie niet, is het nodig om 2 AD Groepen aan te maken. Deze koppelen we met
de groepen die FSLogix lokaal op de server(s) heeft aangemaakt. Zo kunnen we
bijvoorbeeld FSLogix uitschakelen voor domain admins en aanzetten voor
specifieke gebruikers.

Maak 2 groepen in AD: **FSLogix AD Profile Exclude List** & **FSLogix AD Profile
Include List.** Maak de groep domain admins lid van de Exclude groep en voeg in
ieder geval je testaccounts toe aan de include groep.

Maak vervolgens een nieuwe policy en koppel de AD groepen aan de lokale groepen.
De namen van de lokale groepen zijn: **FSLogix Profile Include List** &
**FSLogix Profile Exclude List.** Vink aan dat de groep eerst leeggemaakt moet
worden.

![Afbeelding met schermafbeelding Automatisch gegenereerde beschrijving](media/5b31d32fb7950bdd2196214b1bf20c83.png)

Office Deployment Tool
----------------------

Als je een geldige Office 365 licentie hebt, kun je ook gelijk testen met Office
365 ProPlus. Hiervoor maken we een GPO die dit installeert tijdens het
opstarten. We gebruiken de website <https://config.office.com/> om een XML te
maken voor de installatie. Let er op dat je “Shared Computer Activation”
aanvinkt, anders werkt de software niet op een multi-user omgeving. In mijn
configuratie heb ik een selectie gedaan van een aantal producten. De overige
applicaties zijn uitgesloten. Ook heb ik er voor gekozen dat de bestanden worden
gedownload vanaf het Office Content Delivery Network. Aangezien we onze VM’s
toch in Azure bouwen, is dit de meest makkelijke manier. Je kunt onderstaande
configuratie gebruiken, maar je kan ook je eigen “Configuration.xml” maken.

``` XML 
<Configuration ID="a02f45f4-6a0e-4215-85af-e2458dabbcf4" DeploymentConfigurationID="00000000-0000-0000-0000-000000000000">
    <Add OfficeClientEdition="64" Channel="Insiders" ForceUpgrade="TRUE">
    <Product ID="O365ProPlusRetail">
        <Language ID="MatchOS" />
        <ExcludeApp ID="Access" />
        <ExcludeApp ID="Groove" />
        <ExcludeApp ID="Lync" />
        <ExcludeApp ID="OneNote" />
        <ExcludeApp ID="Publisher" />
    </Product>
    </Add>
    <Property Name="SharedComputerLicensing" Value="1" />
    <Property Name="PinIconsToTaskbar" Value="TRUE" />
    <Property Name="SCLCacheOverride" Value="0" />
    <Updates Enabled="TRUE" />
    <RemoveMSI />
    <AppSettings>
        <Setup Name="Company" Value="ResultaatGroep" />
    </AppSettings>
    <Display Level="Full" AcceptEULA="TRUE" />
</Configuration>
```

Om Office te installeren heb je de Office Deployment Tool nodig. Deze kun je
hier downloaden:
<https://www.microsoft.com/en-us/download/details.aspx?id=49117>

Pak de bestanden uit. De “**setup.exe**” en de eerder gemaakte
“**Configuration.xml”** heb je nodig in de volgende stap.

GPO – Office 365 ProPlus installatie
------------------------------------

Maak een nieuwe GPO. Kies voor Computer Configuration -\> Windows Settings -\>
Scripts -\> Startup en plaats zoals eerder met de FSLogix agent nu de setup.exe
en de Configuration.xml in de folder.

Klik vervolgens op Add en kies de **setup.exe** file. In het Script parameter
veld vul je in: */Configure Configuration.xml*

Koppel de GPO aan de OU die je eerder gemaakt hebt voor je Windows Virtual
Deskop VM’s.

![]({{ site.baseurl }}/images/WVDandFSLogix/870078889b390e4849488efe47aa8739.png)

Office 365 ProPlus settings voor Outlook (Optioneel)
----------------------------------------------------

*Deze stap is optioneel. Je kunt ook handmatig de cache configureren aan de
client kant.*

Om de cache te configureren van Outlook maken we een Group Policy op basis van
de Office 365 ProPlus ADMX bestanden. Deze ADMX bestanden kun je downloaden via
deze link: <https://www.microsoft.com/en-us/download/details.aspx?id=49030>

Kopieer ten minste de ADMX en ADML van Outlook naar
C:\\Windows\\PolicyDefinitions op de domaincontroller.

Let op, deze keer gebruiken we **gebruikersinstellingen** om de cache af te
dwingen.

Maak een nieuwe GPO en configureer als volgt:

| **User Configuration -\> Policies -\> Administrative Templates -\> Microsoft Outlook 2016/Account Settings/Exchange/Cached Exchange Mode** |                  |
|--------------------------------------------------------------------------------------------------------------------------------------------|------------------|
| Cached Exchange Mode Sync Settings                                                                                                         | Enabled          |
| Select Cached Exchange Mode sync settings for profiles                                                                                     | Zelf te bepalen. |
| Use Cached Exchange Mode for new and existing Outlook profiles                                                                             | Enabled          |

Server(s) herstarten
--------------------

Als alle Group Policies zijn aangemaakt is het nodig om de server(s) te
herstarten. Bij het opstarten zullen de FSLogix Agent en Office 365 ProPlus
geïnstalleerd worden. Dit duurt uiteraard een paar minuten.

Hierna kun je inloggen en Outlook en Onedrive instellen voor gebruik. Gebruik
hiervoor een reeds bestaand Office 365 / Exchange account voor nodig.

![]({{ site.baseurl }}/images/WVDandFSLogix/8737b7f535709504b383f1d598ab2794.png)

VHDX locatie
------------

Controleer nu in de Azure portal of je de VHDX op de juiste manier terug ziet.
Als het goed, ziet het er als volgt uit:

![]({{ site.baseurl }}/images/WVDandFSLogix/d6feba40c92a80cf26c9ff284fa69f21.png)

![]({{ site.baseurl }}/images/WVDandFSLogix/f049c5973e5d11668637004455a60cf1.png)

Troubleshoot Profile containers
-------------------------------

Samen met de FSLogix software wordt ook een tool geïnstalleerd waarmee je wat
meer logging kunt uitlezen betreffende Profile Containers. De staat op de
volgende locatie: **"C:\\Program Files\\FSLogix\\Apps\\frxtray.exe"**

*TIP: kopieer deze tool naar C:\\ProgramData\\Microsoft\\Windows\\Start
Menu\\Programs\\StartUp. Hierdoor start deze op voor elke gebruiker.*

![]({{ site.baseurl }}/images/WVDandFSLogix/ccc9341744562675486835b5e350f706.png)

Meer informatie
===============

We hebben onderstaande bronnen gebruikt voor het maken van deze blog. Hier kun
je informatie nog eens rustig nalezen. Check ook deze uitgebreide blog van
Pieter Wigleven :
<https://techcommunity.microsoft.com/t5/Windows-IT-Pro-Blog/Getting-started-with-Windows-Virtual-Desktop/ba-p/391054>

[FSLogix Documentatie](https://docs.fslogix.com/)

[Windows Virtual Desktop
Documentatie](https://docs.microsoft.com/nl-nl/azure/virtual-desktop/overview)

Voor vragen en/of opmerkingen kun je ons een berichtje sturen via LinkedIn.

| [Roel Everink](https://www.linkedin.com/in/roeleverink/) | [Jan Bakker](https://www.linkedin.com/in/jan-bakker/) |
|----------------------------------------------------------|-------------------------------------------------------|
| ![]({{ site.baseurl }}/images/WVDandFSLogix/image35.jpg)             | ![]({{ site.baseurl }}/images/WVDandFSLogix/image36.jpg)            |
