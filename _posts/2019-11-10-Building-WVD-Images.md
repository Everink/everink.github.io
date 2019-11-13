---
layout: post
title: "Building a WVD image the right way"
categories: [Azure,Windows Virtual Desktop]
---

Over the past few months I've seen multiple articles about how to create a Windows Virtual Desktop (WVD) image.
They usually login to the VM themselves, install some apps and do some modifications, and sysprep the VM. After that you can optionally make a snapshot of your VM, and then convert it to a managed image which you can use for a WVD deployment.

Now, this is all good for a POC, demo or small production environment. But when things start to scale out, it just doesn't cut it anymore.
When building any type of image, you'd want the following things taken care off.

- Consistent
- Automated
- Rollback procedure

You'd want consistency so you don't accidental forget something in your golden image  
Preferably you'd have something automate your creation of your image. Installing multiple pieces of software can be a tedious task, that time can be better spend doing something else. It also reduces the change of human error.  
When you make a change to your golden image that breaks it, you'd better remember what you changed, and how to rollback to the previous state.

All those things can be solved by a single Azure service:  
[**Azure Image Builder**](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/image-builder-overview)

Azure Image Builder (AIB) allows you to take a source image, which can be any of the following:

- RHEL ISO
- Marketplace image
- Managed image
- Shared Image Gallery image version

It then can customize that specific image to your needs in an automated way.  
And as a final step AIB can distribute your image to any or multiple of the following:

- Managed image
- Shared Image Gallery
- VHD in a storage account

You can see the overview in the following picture
![]({{ site.baseurl }}/images/AIB/AIB-overview.png)

Now, AIB is still in preview, so there are a few limitations to the service.
It is limited to the folling locations, but can still distribute outside of these locations.
- East US
- East US 2
- West Central US
- West US
- West US 2

There is also no GUI (yet?) for Azure Image Builder  
When building the image there is no way to check the progress, other then checking the logs manually, or checking if your image is already present at your distribution location.

With that bit of background info, lets start creating our own image with AIB!

## Register the feature

To use Azure Image Builder during the preview, we have to first enable the service. We do that by registering the feature.
This can be done with PowerShell, so we need to install the Az module within a privileged PowerShell windows, and after that we can login to our subscription

{% highlight powershell linenos %}
Install-Module Az -Force
Connect-AzAccount
{% endhighlight %}

When that is done, we can register the AIB feature with the following PowerShell command:

{% highlight powershell linenos %}
Register-AzProviderFeature -ProviderNamespace Microsoft.VirtualMachineImages -FeatureName VirtualMachineTemplatePreview
{% endhighlight %}

To check the status of the feature, run the following command

{% highlight powershell linenos %}
Get-AzProviderFeature -ProviderNamespace Microsoft.VirtualMachineImages -FeatureName VirtualMachineTemplatePreview
{% endhighlight %}

> Note: It can take a while before the registrationstate will change to: **Registered**

After that make sure the following 2 commands also show **Registred**.

{% highlight powershell linenos %}
Get-AzResourceProvider -ProviderNamespace Microsoft.VirtualMachineImages | Select-Object RegistrationState
Get-AzResourceProvider -ProviderNamespace Microsoft.Storage | Select-Object RegistrationState
{% endhighlight %}

If there's something showing **NotRegistered** run the following commands

{% highlight powershell linenos %}
Register-AzResourceProvider -ProviderNamespace Microsoft.VirtualMachineImages
Register-AzResourceProvider -ProviderNamespace Microsoft.Storage
{% endhighlight %}

When everything is done, you are ready for the next step!



## Assign rights to AzureImageBuilder

During the previous step we enabled the feature, one of the things that happend, was that a service principal has been made in our Azure AD.
This service principal (SP) is used to give AIB rights on certian resource (groups). The Application ID of the service principal is always the same:  
*cf32a0cc-373c-47c9-9156-0db11f6a6dfc*  

This SP needs rights on a resourcegroup that will be used for AIB. We first make this resourcegroup with the command:  
{% highlight powershell linenos %}
New-AzResourceGroup -Name "RG_EUS_AzureImageBuilder" -Location 'East US'
{% endhighlight %}

The location needs to be in one of the supported AIB regions. Here I choose East US.
Then we have to assign the **contributor** right to our SP for this resource group. You can do this with the portal, or also with the commandline.  
{% highlight powershell linenos %}
New-AzRoleAssignment -RoleDefinitionName "Contributor" -ApplicationId "cf32a0cc-373c-47c9-9156-0db11f6a6dfc" -ResourceGroupName "RG_EUS_AzureImageBuilder"
{% endhighlight %}

![]({{ site.baseurl }}/images/AIB/AIB-RBAC.png)

## Deploying AzureImageBuilder ARM template

When all permissions are in place it's time to start with the deployment.
This is done with an [ARM template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authoring-templates).  
There is currently no portal experience for AIB.

The demo ARM template from Microsoft can be found [on GitHub](https://github.com/danielsollondon/azvmimagebuilder/blob/master/quickquickstarts/0_Creating_a_Custom_Windows_Managed_Image/helloImageTemplateWin.json)

If you look at the template you can see that on the properties part there are 3 sections

1. Source
2. Customize
3. Distribute

For source the best way to go in my opinion is to pick a marketplace image. These are tested by Microsoft, and are regularly updated to a new version. Of course you have to choose a new version every month or so.

In the Customize section you can customize your VM with a PowerShell script. This can be an external (publicly available) script, in a storage account or on GitHub. Or it can be an inline script if it's only a few lines.

In the Distribute section you can define how and where to distribute your images. The most easy is to just deploy a managed image in the same resourcegroup as your AIB service.

If you want to use the Microsoft quickstart template, you can download it and edit the following parts:
- Replace \<subscriptionID\> for your subscriptionID
- Replace \<rgName\> for your AIB resourcegroupname
- Replace \<region\> for a region OR replace it with a function that used the ResourceGroupFunction. That means replacing it for: [resourceGroup().location], including the square brackets
- Replace \<imageName\> for a custom managed image name
- Replace \<runOutputName\> for aibWindows (or something else you make up)

If however, you want to make use of another ARM template, I have one in [my GitHub account](https://raw.githubusercontent.com/Everink/AzureImageBuilder/master/AzureImageBuilder.json) as well, that uses parameters with some default values setup.

It will use the customize script that's also in [my GitHub account](https://raw.githubusercontent.com/Everink/AzureImageBuilder/master/AzureImageBuilder.ps1). It will install Visual Studio Code, Teams, Notepad++ and FSLogix.

To deploy this template use the following PowerShell commands

{% highlight powershell linenos %}
$TemplateUri = "https://raw.githubusercontent.com/Everink/AzureImageBuilder/master/Templates/AzureImageBuilder-ManagedImage.json"
New-AzResourceGroupDeployment -ResourceGroupName RG_EUS_AzureImageBuilder -TemplateUri $TemplateUri -OutVariable Output -Verbose
{% endhighlight %}

This will create an ImageTemplate package, which is linked to a new ResourceGroup IT_\<AIB-resourcegroupname\>_\<AIB-imagetemplatename\>\<random GUID\>. In that resourcegroup it will setup all prerequisites, like downloading powershell scripts or files, and checking if AIB has all necessary rights on other resources (like Shared Image Gallery if you want to distribute to that)  
It does not yet start building our image. It can be seen in the portal by selecting **Show hidden items**

![]({{ site.baseurl }}/images/AIB/AIB-imagetemplate.png)

In our previous PowerShell commandlet we used the output variable "Output", to capture the name of our AIB imagetemplate. We can see it with

{% highlight powershell linenos %}
$Output.Outputs["imageTemplateName"].Value
{% endhighlight %}

We will use the imagetemplate name to start building our golden image. This can be done by *invoking* or *executing* the resource.

{% highlight powershell linenos %}
$ImageTemplateName = $Output.Outputs["imageTemplateName"].Value
Invoke-AzResourceAction -ResourceGroupName RG_EUS_AzureImageBuilder -ResourceType Microsoft.VirtualMachineImages/imageTemplates -ResourceName $ImageTemplateName -Action Run
{% endhighlight %}

This will start building the image.

To check the build status run the following command

{% highlight powershell linenos %}
(Get-AzResource -ResourceGroupName RG_EUS_AzureImageBuilder -ResourceType Microsoft.VirtualMachineImages/imageTemplates -Name $ImageTemplateName).Properties.lastRunStatus
{% endhighlight %}

When the build is complete there will be a managed image in our resource group, and we can start deploying VM's from it!

![]({{ site.baseurl }}/images/AIB/AIB-newImage.png)
![]({{ site.baseurl }}/images/AIB/AIB-createVMfromImage.png)

When we login to the VM, we can see that the C:\temp folder has been populated with all the installers, and applications like team have also been installed to our image, just like we wanted.

![]({{ site.baseurl }}/images/AIB/AIB-InsideVM.png)

You can now create different build templates for your different Windows Virtual Desktop hostpools, or create a default server VM image for your IaaS workloads.

If you have any questions, feel free to leave a comment below, or find me on LinkedIn.



