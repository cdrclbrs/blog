---
layout: post
title: "How to use Group Manaed Service Accounts (gMSA)"
excerpt: "gMSA deployement and usage"
excerpt_separator: "<!--read-->"
categories:
  - tips
tags:
  - active Directory
  - AD
  - Security
last_modified_at: 2021-05-03T03:22:48-05:00
---

Table of Contents
=================

- [Table of Contents](#table-of-contents)
- [Presentation](#presentation)
- [Create a gMSA](#create-a-gmsa)


# Presentation


- The same gMSA can be used on several servers

- gMSAs are stored in the "Managed Service Account" container in the Active Directory

- A gMSA can only be used on Windows Server 2012 and later

- Requires the use of Microsoft Key Distribution Service (kdssvc.dll) for automatic password management and account creation

- a gMSA is similar to a security group in which we will associate computer objects that will be allowed to use this secure service account

- The password is managed by the Active Directory, it is very very complex and nobody knows it

With an MSA or gMSA account, the password management is automatic by the Active Directory itself, unlike the use of a classic user account, which can be used for a service but for which you must manage the password renewal yourself. Very often, as it is time-consuming, the passwords of these accounts are not renewed by the administrators. That's why it's useful to rely on the gMSA solution, which also offers increased security for credentials.

An MSA account can be associated to only one server, unlike gMSA, which is restrictive when you need to use a service account on a service that is redundant between several servers.

In terms of compatibility, gMSA accounts work with different types of applications and features, including:

- Windows services
- Scheduled tasks
- IIS servers (Application Pool), SQL Server, ADFS, etc.

# Create a gMSA

To be able to create gMSA accounts on Active Directory infrastructure, the Key Distribution Service must be running and a root key must be generated. To create a key from the domain controller, we will use PowerShell and the Add-KdsRootKey cmdlet.

It is possible to delay the activation of the generated key by using the -EffectiveTime parameter followed by a date. If we use the -EffectiveImmediately parameter the key will be usable 10 hours after its creation (default behavior) in order to ensure that it is replicated between the different DCs.

Here is the command to execute:
```
Add-KdsRootKey -EffectiveImmediately
```
As part of a lab, if you want to be able to use the KDS key now without having to wait 10 hours, it is possible to cheat by using this command:
```
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
```
The created key is identifiable with a Guid.

![kds](https://blog.lbrs.io/images/gmsa1.png)

The key can be displayed simply by running the command below:

Get-KdsRootKey

This returns a result in this form:
```
AttributeOfWrongFormat:
KeyValue : {174, 194, 208, 151...}
EffectiveTime : 16/04/2020 18:43:58
CreationTime : 16/04/2020 18:43:58
IsFormatValid : True
DomainController : CN=SRV-ADDS-01,OU=Domain Controllers,DC=IT-CONNECT,DC=LOCAL
ServerConfiguration : Microsoft.KeyDistributionService.Cmdlets.KdsServerConfiguration
KeyId: 907a7443-1e9e-4026-b06b-03ae9adf9ce6
VersionNumber : 1
```
If you want to check the validity of a root key, the Test-KdsRootKey cmdlet can be used. You just have to specify the Guid of the key to check. For example:

Test-KdsRootKey -KeyId 907a7443-1e9e-4026-b06b-03ae9adf9ce6

If the key is valid, the value true will be returned in the console.

The Kds keys are visible in the "Active Directory Sites and Services" console, by enabling the "Show Services Node" option in the "View" menu.

Then browse this way: Services > Group Key Distribution Service > Master Root Keys

Now that this prerequisite is met, we can move on to the next step.

Translated with www.DeepL.com/Translator (free version)