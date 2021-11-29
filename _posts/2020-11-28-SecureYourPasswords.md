---
layout: post
sidebar_link: true
title: "Secure your passwords with KeepassXC"
excerpt: "KeyPass and Yubikey operations"
excerpt_separator: "<!--more-->"
categories:
  - security
tags:
  - Security
  - Passwords
last_modified_at: 2020-11-28T23:23:48-05:00
---

## Keypass or Bitwarden?
First, I'll explain why I chose Keepass over Bitwarden. Both KeePass and Bitwarden use military grade **AES256** technology.
I'll write about keePassXC, which is a fork of Keepass.
Both solutions are *free* (*Bitwarden offers paid plans*) and OpenSource, which means that the community has the right to look at and improve the source code, which is very important.
The difference is that Bitwarden stores your passwords in the cloud, where Keepass only allows you to store them locally or on a "Synchronized" repository. This is where it's very interesting because you can access your password database from anywhere freely (via a trusted CSP ( like protondrive) or a private cloud like NextCloud etc...

Of course both solution do provide Browser Support with specific add-ins. However, i prefer avoiding browser autofill that represent a risk.

The other feature i like with KeePassXC is the ability to protect the integrity and the confidentiality of the wallet with interesting methods:
- The usage if a KeyFile
- The usage of a Challeng-Response With a HardWare Key (Like Yubikey)

I find these two methods very interesting because they allow you to protect yourself from keylogger malware. 
It doesn't matter if you have your passwords on an "ultra protected" safe with a complex 42 characters password:  If your master password is dumped by a KeyLogger process, you are screwed :)

## Setuping your Keypass Database

#### Find a keyfile
![[mysecurityfile.jpg]]

#### What is a key file ?
A key file is a file containing random bytes that can be added to your master key for additional security. 
If the file changes, it is as if you forgot your password and you will lose access to your database: So, for instance, a static and never-changing holiday picture is okay, your personal notes file is not. 

#### How secure is a key file and how can I sync it to other devices?
A key file is only as secure as you keep it. It is basically a password that you've written down. As a general rule, you should never use a key file without an actual password, because it is harder to keep your key file secret than a memorized password that only you know. However, a key file can be very strong additional protection if kept separately from the database file, such as on an external thumb drive. If you sync your database via a cloud provider (Dropbox, Google Drive, Nextcloud, …).

#### Configuration of the Hardware security Key
To use a YubiKey for securing your KeePassXC database, you have to configure one of your YubiKey slots for **HMAC-SHA1** Challenge Response mode
Download the tool at: yubico.com/pt

![[YubikeyPersonalizationTool.png]]

![[YubikeyPT.png]]

Now you need to create your Database:
Add additionnal protection with your **Key File** and **YubiKey**
![[DbSetupKeepass.png]]

Now while starting, if you do not provide your KeyFile, but your password and your YubiKey, you're still blocked:
![[KeePassStartupError.png]]

While all the infos are filled, just touch the YubiKey (this ensures that there is a physical interaction)
![[KeePassStartupTouch.png]]

Now you can use your wallet of passwords:
Everytime, you write to the database, a "touch" prompt ensures that it is expected access and requires your validation.

Now you're also proteced against the keyloggers and shoulder surfing attacks!