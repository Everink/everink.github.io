---
layout: post
title: "Azure AD FIDO2 Authentication"
categories: [Azure,Authentication]
---

FIDO2 authentication is the latest method to sign in passwordless on devices and websites.

It was already possible for a few months to login passwordless with Microsoft accounts with a FIDO2 security key, but recently Azure AD FIDO2 authentication is in public preview.

In this blog I will show you how to setup your own tenant and device so you can leverage passwordless signin to Azure AD with FIDO2

## Table of Contents
1. [What is FIDO2]({{ site.baseurl }}{{ page.url }}#what-is-fido2)
2. [Downsides of passwords]({{ site.baseurl }}{{ page.url }}#downsides-passwords)
3. [Benefits of FIDO2 (password-less)]({{ site.baseurl }}{{ page.url }}#benefits-passwordless)
4. [Requirements]({{ site.baseurl }}{{ page.url }}#requirements)
5. [Enable combined registration preview]({{ site.baseurl }}{{ page.url }}#combined-registration-preview)
6. [Enable FIDO2 authentication]({{ site.baseurl }}{{ page.url }}#enable-FIDO2)
7. [User registration and management of security keys]({{ site.baseurl }}{{ page.url }}#userregistration)
8. [Login to websites with a FIDO2 security keys]({{ site.baseurl }}{{ page.url }}#login-websites)
9. [Login to Windows 10 with a FIDO2 security keys]({{ site.baseurl }}{{ page.url }}#login-windows10)

## What is FIDO2 <a name="what-is-fido2"></a>
FIDO stands for: Fast Identity Online

FIDO has been founded by the [FIDO Alliance](https://fidoalliance.org/overview/).
The FIDO Alliance is an open industry association with a focused mission: authentication standards to help reduce the world’s over-reliance on passwords.

FIDO2 is the overarching term for FIDO Alliance’s newest set of specifications. FIDO2 enables users to leverage common devices to easily authenticate to online services in both mobile and desktop environments. The FIDO2 specifications are the World Wide Web Consortium’s (W3C) [Web Authentication (WebAuthn) specification](https://fidoalliance.org/fido2/fido2-web-authentication-webauthn/) and FIDO Alliance’s corresponding [Client-to-Authenticator Protocol (CTAP)](https://fidoalliance.org/specifications/download/).

This makes it possible to login to services as Azure AD, or a random website, with a simple USB key, smartcard or even your phone.

This is based on the device that carries your credentials, combined with something you know or are. Like a PIN or a fingerprint

## Downsides of passwords <a name="downsides-passwords"></a>
Passwords are everywhere, because they are so simple to implement. But that simple implementation can also have it downsides!

**Passwords are often reused**  
Because we need so many passwords, they are often recycled for differnt services. So when 1 password is compromised, multiple services are often at risk because the same password could be used for those aswell.

**Vulnerable to database hacks**  
Passwords are based on a pre-shared key. So the online service has a copy of your password, or a hashed version of your password.
These databases can be compromised, so an attacker can find out your password

**Vulnerable to Brute Force attacks**  
Passwords need to be remembered, so are often easily breached by a brute force or dictionary attack.

**Vulnerable to social engineering attacks**  
This could be a simple as watching over someone shoulder as he/she types the password, or by guessing the password based on data found on social media.

**Vulnerable to keyloggers**  
Regular passwords are vulnerable for keyloggers, this could be a hardware based keylogger between the PC and keyboard, but also a software keylogger. These record all keystrokes, including your password

## Benefits of FIDO2 (password-less) <a name="#benefits-passwordless"></a>
With FIDO2 all downsides of passwords are non-existent. This is because of the following.

**Public / private keys**  
FIDO2 does not use a pre-shared key, but it uses public / private key pairs.

The public key is handed to the identity provider (like Azure AD), and the private key stays on the device, and will never leave it.
It will only be used to sign a challenge.

So even if your public key gets stolen it's no problem, because it's useless. That's what makes it a public key after all.

**Unique keypairs**  
Where passwords are often reused for differnt services, that isn't the case with FIDO2. For every online identity a new key pair is generated. So every identity has its own public and private key.

**Phishing is part of the past**  
Because your identity and keypair is linked to a login domain (like login.microsoft.com), a challenge from a phishing site won't be recognised by your FIDO2 key. A phishing site will try to let your login to login.micr0s0ft.com for example, and your FIDO2 key won't have a keypair for that login domain.

**Social engineering**  
You will not be vulnerable to social engineering attacks, as there is nothing to gues. You don't even know your own private key.
And if you use a FIDO2 key with a fingerprint there is also nothing to see when you login. So no more looking away for your colleagues.

## Requirements <a name="requirements"></a>

To enable FIDO2 authentication there are a few requirements.

- Enable combined registratie (preview)
- Windows 10 1809 or higher met Microsoft Edge
- Compatible FIDO2 security key

For Windows sign in:
- Azure AD joined Windows 10 1809 or higher


## Enable combined registration preview <a name="combined-registration-preview"></a>

The possibility to register a security key is only available in the new registration portal. That's why we have to enable this feature first.

To to this login to <https://portal.azure.com> with a global administrator account.
Go to **User Settings** -\> **Manage user feature preview settings**

![]({{ site.baseurl }}/images/fido2-authentication/cd639ac5645ac878afecac21680daa72.png)

Make sure you select **All** or **Selected** for the feature:

-   Users can use preview features for registering and managing security info – enhanced

If you choose **Selected** you can select a pilot group first.
Then click on **Save**

![]({{ site.baseurl }}/images/fido2-authentication/aa1030f79f088898c33e116b29735bcf.png)

## Enable FIDO2 authentication <a name="enable-FIDO2"></a>

To enable FIDO2 as an authentication method we login to the azure portal with a global administrator account.
Then go to **Azure Active Directory** -\> **Authentication Methods**

Then click on  **FIDO2 Security Key**

![]({{ site.baseurl }}/images/fido2-authentication/24d2b56d2e8a125e5c9abaca4c9bba54.png)

At **ENABLE** click **Yes**

At **TARGET** I selected **All users**. It is possible to select a pilot group or user here.
Then click on **Save**

## User registration and management of security keys <a name="userregistration"></a>

To register a security key as a user, you can go to the following page:
<http://aka.ms/setupsecurityinfo>

Click on **Add method**

![]({{ site.baseurl }}/images/fido2-authentication/a4f5a8bbf18dbb0bca2686ad224d6c99.png)

Then choose the option: **Security Key** and click on **Add**

![]({{ site.baseurl }}/images/fido2-authentication/0e70e5b1f4a936799356ffe31f15ae5e.png)

As my FIDO2 key is a USB device, I pick that.

![]({{ site.baseurl }}/images/fido2-authentication/1178b8cb3d94058b3f35f4f3bd9b497a.png)

After that I have to take action on my security key. This is dependent on how your key is configured and which features it supports
I have a Feitian Biopass with a fingerprint reader which is sufficient for me. Sometimes there's only a single button on it, and you have to enter a PIN as additional verification.

You can also see that the public / private key pair will be made for login.microsoft.com, so the key pair will only work for this site.

![]({{ site.baseurl }}/images/fido2-authentication/3b0b9a97ba42bfdf574b9230010464ca.png)

You can give your security key a descriptive name to identify it more easy.
I call it: FeitianBiopass-Roel

![]({{ site.baseurl }}/images/fido2-authentication/5b789b810b93329ac064c3f94d50d40c.png)

Then the key is ready for use!

![]({{ site.baseurl }}/images/fido2-authentication/8b74541c02ca33b7c5e875476b35d3b9.png)

## Login to websites with a FIDO2 security keys <a name="login-websites"></a>

To login to a website with our key we can go to any Microsoft page where we can login with our Azure AD account. Like <https://myapps.micosoft.com>,
<https://outlook.office.com>, or <https://office.com>

Click on **Sign-in options**

![]({{ site.baseurl }}/images/fido2-authentication/c78f37eca418b15ad003ee9fe9a1b312.png)

And then **Sign in with a security key**

![]({{ site.baseurl }}/images/fido2-authentication/327b0d5275ff3cebcaf8ab0e2ea3cc9b.png)

Then follow the instructions to insert your key, and take action.

![]({{ site.baseurl }}/images/fido2-authentication/f8673b59dee4f167b1b217c6758ea08c.png)

![]({{ site.baseurl }}/images/fido2-authentication/61f900710c7ccf1308b0e64787fc9edd.png)

And with that we signed in to our account, without using our username or password!

## Login to Windows 10 with a FIDO2 security keys <a name="login-windows10"></a>

To login to Windows 10 with a security key, we have to enable this feature first.
You can do this in two ways:

1.  credential provider via Intune
2.  credential provider via provisioning package

I go for option 1, via Intune

The reason for this is that my device has to be Azure AD joined anyway, and it's a small effort to do the Intune configuration compared to the provisioning package.
It's also more scalable with Intune

Go to the Azure portal, and search for Intune([or click here for the direct link](https://portal.azure.com/#blade/Microsoft_Intune_DeviceSettings/ExtensionLandingBlade/overview))

Go to **Device configuration** -\> **Profiles** -\> **Create profile**
And make a profile with the following settings:

**Name:** [profilename]  
**Description:** [beschrijving van het profiel]  
**Platform:** Windows 10 and later  
**Profile type:** Custom

![]({{ site.baseurl }}/images/fido2-authentication/df40f0b78b5565ba9f089ee0b890f0c9.png)

You have to add a new OMA-URI at settings.
Click on **Add**
And fill out the following fields:

**Name:** [Name of the setting]  
**Description:** [Description of the setting]  
**OMA-URI:** ./Device/Vendor/MSFT/PassportForWork/SecurityKey/UseSecurityKeyForSignin  
**Data type:** Integer  
**Value:** 1

![]({{ site.baseurl }}/images/fido2-authentication/1687e9c140e81857cbf2a716db7c4dbd.png)

Go to **Assignments**
At **Assign to** select **All Users & All Devices**, or optionally select a pilot group or user

![]({{ site.baseurl }}/images/fido2-authentication/2573ffee1ed3a7100f24d05303765451.png)

As soon as these settings are pushed to our device it's possible to sign in to windows with our FIDO2 security key
For this, click on **Sign-in Options**, and choose the **USB symbol (FIDO security key)**

![]({{ site.baseurl }}/images/fido2-authentication/1fad81d24b7d2136ea4750a6e97cb66f.jpg)

Next we can authenticate on your security key, and we will be signed into Windows, again, without using our username or password!

![]({{ site.baseurl }}/images/fido2-authentication/fb3236cff94006270af90c6746a863ed.jpg)

This completes our full setup. We can now sign in to our device and to web pages with our security key and enjoy the benefits of passwordless authentication!
