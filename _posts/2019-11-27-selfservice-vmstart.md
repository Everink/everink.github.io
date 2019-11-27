---
layout: post
title: "PowerShell Function app behind Azure AD authentication"
categories: [Azure,Azure Functions,PowerShell]
---

Every year, me and a couple of cousins of mine pick a day, and we game all day. Usually it's an old game that everybody can play. Like Command & Conquer: Generals.  
Back in the days we all came together and linked our PC's with a switch, and we were good to go. But these days we're grown up and living miles apart, so we usually link are PC's over the internet.
With an old game like that internet connect doesn't work that well, it needs a LAN. So I build an pfsense server in Azure that supports Layer 2 broadcast messages over the VPN. This all works really well.

To reduce my Azure spending I have an auto shutdown schedule configured. It shuts down my VM at 23:30, and it stays that way until turned on.  
Sometimes my cousins want to game while I have to work, or have other stuff to do, and I dont have time to turn on my Azure VM, and of course, I don't want to give them access to my subscription either. 

There is a solution for my usecase:  
**Azure Functions**

Azure Functions for PowerShell core is now [General Available](https://azure.microsoft.com/en-in/updates/powershell-support-in-azure-functions-is-now-generally-available/){:target="_blank"}, so its fully supported. And it gives me the opportunity to secure it behind Azure authentication.


# Table of Contents

1. [Setup]({{ site.baseurl }}{{ page.url }}#setup)
2. [Creating the Function App]({{ site.baseurl }}{{ page.url }}#create-functionapp)
3. [Creating a Managed Identity]({{ site.baseurl }}{{ page.url }}#managed-identity)
4. [Creating the first function]({{ site.baseurl }}{{ page.url }}#first-function)
5. [Creating the second function]({{ site.baseurl }}{{ page.url }}#second-function)
6. [Add AzureAD authentication]({{ site.baseurl }}{{ page.url }}#azuread-authentication)
7. [Group or user based access]({{ site.baseurl }}{{ page.url }}#group-based)


# 1. Setup <a name="setup"></a>

For this we will need 2 functions
- One function will be the main website, it should also show the status of my VM, it will contain a button that will trigger my second function
- My second function will start my VM. It should also give some status back.

Both functions can use the same Function App. They will both have to be secured with Azure AD authentication, so not everyone can start my VM. 

With this setup done, lets start building

# 2. Creating the Function App <a name="create-functionapp"></a>
We can start by creating a resource group to hold our functions.

I call my resource group: *RG_WEU_SelfServiceFunctions*, as it's located in West Europe.

After that we can go to: **+Create a Resource** -> Search for: **Function App** -> **Create**

Select a subscription. And select your resource group that you just made. 
Give your function app a name, it should be globally unique  
For Runtime stack choose *PowerShell Core*  

![]({{ site.baseurl }}/images/selfservice-function/new-functionapp1.png)

On the next page you can create a new storage account, or choose an existing one.  
For the plan I chose Consumption, depending on your usecase you can also choose another plan.

![]({{ site.baseurl }}/images/selfservice-function/new-functionapp2.png)

As I don't need any monitoring I turn application insights off. This is only for my personal use, so I don't need any monitoring.

![]({{ site.baseurl }}/images/selfservice-function/new-functionapp3.png)

Then on the Review + Create tab. Choose **Create**

# 3. Creating a Managed Identity <a name="managed-identity"></a>

To determine the status I can use the Az PowerShell module, the Az module gets is supported right out of the box in Azure Functions, so I don't have to install it first. But I do need an account to authenticate to my subscription.  
For this we can use a managed identity.

The managed identity is created on the Function App level, so it's for all our functions in our Function App.
Go to the **function** -> **Platform features** -> **Identity**

![]({{ site.baseurl }}/images/selfservice-function/managed-identity1.png)

Change the status to **On** and then click **Save**

![]({{ site.baseurl }}/images/selfservice-function/managed-identity2.png)

Our Function App now has a managed identity, but it doesn't have any rights in our subscription. For this I assign rights on just the resource group, but depending on your function you can also assign rights on the subscription level.

![]({{ site.baseurl }}/images/selfservice-function/managed-identity3.png)

The managed identity has the same name as your function app. Select it, and click **Save**

# 4. Creating the first function <a name="first-function"></a>

Once your Function App is ready, and you have your managed identity, You can create a function with the quickstart by clicking on the  **+** sign next to functions.

![]({{ site.baseurl }}/images/selfservice-function/new-function1.0.png)

The downside of the quickstart is that it will give your function a standard name. If you want to name your function yourself, click on **Functions** and then **+ New function**

![]({{ site.baseurl }}/images/selfservice-function/new-function1.1.png)

As my first function will be a website, the trigger is an HTTP trigger, it will be a simple GET request like any website, so click **HTTP Trigger**  
After that you can give your function a name, and and authorization level. I choose Anonymous, as I will add Azure AD authentication later.

![]({{ site.baseurl }}/images/selfservice-function/new-function1.2.png)

We get thrown into a webeditor to edit our function. As I want to show a webpage, I have to build some HTML code, and insert some status about my VM. As we created a managed identity we can just use Az commandlets, and it will use the managed identity to authenticate.
For the code, it can be found on my [GitHub repo](https://github.com/Everink/SelfService-VMstart/blob/master/web/run.ps1){:target="_blank"}.

There's only one more thing to do. Currently my function supports both the GET and POST method. We can disable the POST method.  
This is done on the **Integrate** section. Deselect **POST**, and click **Save**

![]({{ site.baseurl }}/images/selfservice-function/new-function1.3.png)

To test your function, you can go to the webpage. It's in the form of:
> https://\<functionApp-name\>.azurewebsites.net/api/\<function-name\>

You can also find it in the interface on the following location

![]({{ site.baseurl }}/images/selfservice-function/new-function1.4.png)

# 5. Creating the second function <a name="second-function"></a>

As you might have noticed, in our first function we call the second function with a [POST method](https://github.com/Everink/SelfService-VMstart/blob/master/web/run.ps1#L45){:target="_blank"}. It doesn't really contain any data, but you could build it like that if you would want to (like selecting which VM to start for example).

The procedure for the second function is the same as the first function. There are a few exceptions:

We name it **vmstarter**  
Allowed method is now POST, so we have to **uncheck GET** under Integrate

That's it. It uses the same managed identity as our first function, as they both are using the same function app.

For the PowerShell code, you can use and edit the code on my [GitHub repo](https://github.com/Everink/SelfService-VMstart/blob/master/vmstarter/run.ps1){:target="_blank"}.

Now it's time to test the function and see if the VM will start!

# 6. Add AzureAD authentication <a name="azuread-authentication"></a>

As I don't want my functions exposed to the internet I can add Azure AD authentication. This can be enabled on the Function App level.
Go to your **function app**, and click **Platform features** -> **Authentication / Authorization**

![]({{ site.baseurl }}/images/selfservice-function/authentication1.png)

Turn **On** App Service Authentication  
In the dropdown menu, select **Log in with Azure Active Directory**  
This will deny any unauthenticated request to my functionapp. 
We will have to configure Azure AD, so click on **Azure Active Directory**

![]({{ site.baseurl }}/images/selfservice-function/authentication2.png)

For Management mode, click **Express**, and click **OK**  
This will create an app registration in Azure AD, called *selfservicevm*

![]({{ site.baseurl }}/images/selfservice-function/authentication3.png)

Click **Save** again.

This will enable Azure AD authentication. If you login to the function app again, you will be triggered to login. Because our app wants to read the profile, everyone has to give consent, or you can give consent on behalf of your organization with a global admin account.

![]({{ site.baseurl }}/images/selfservice-function/authentication4.png)

Now all users can login to your function app!

In my case, I can invite my cousins as guest users to my Azure AD, so they can also use this with their own Microsoft accounts.

# 7. Group or user based access <a name="group-based"></a>

If you want to control access to your application based on group membership or to single users, that is also possible.
The selfservice app can be found under Enterprise Application. In the properties section User assignment required is now set to **No**. This means everyone in our tenant has access. This can be set to **Yes**.

![]({{ site.baseurl }}/images/selfservice-function/group-based-access1.png)

After that go to Users & Groups and select the users or groups that are allowed to access the application.

This concludes the guide to create an Azure PowerShell function behind Azure AD authentication


