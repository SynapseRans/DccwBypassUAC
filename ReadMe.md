<h1>DccwBypassUAC</h1>
<p align="justify">This exploit abuses the functionality of "dccw.exe" by means of a derivative Leo's Davidson "Bypass UAC" method so as to obtain an administrator shell without prompting for consent. It supports "x86" and "x64" architectures. Moreover, it has been successfully tested on Windows 8.1 9600, Windows 10 14393, Windows 10 15031 and Windows 10 15062.

If you want to see how execute the script, take a look at the <a href="https://github.com/L3cr0f/DccwBypassUAC#3-usage">usage</a> section.</p>

In the following days more updates will be uploaded, even a Metasploit version.
<br>
<h2>1. Development of a New Bypass UAC</h2>
<h3>1.1. Vulnerability Search</h3>
<p align="justify">To develop a new bypass UAC, first we have to find a vulnerability on the system and, to be more precise, a vulnerability in an auto-elevate process. To get a list of such processes we used the "Sysinternals" tool "Strings". After that, we could see some auto-elevate processes like "sysprep.exe", "cliconfig.exe", "inetmgr.exe", "consent.exe" or "CompMgmtLauncher.exe" that had (some of them still have) vulnerabilities that allow the execution of a "bypass UAC". So we start to study how other auto-elevate processes worked with the "Sysinternals" application "Process Monitor" ("ProcMon"), but focusing on the process "dccw.exe".</p>

<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/AutoElevate_Processes.png">

<p align="justify">However, before starting with "ProcMon", first we check the manifest of such applications with another "Sysinternals" application called "Sigcheck", and, of course, in our case "dccw.exe" is an auto-elevate process.</p>

<p align="center">

<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/autoElevation_confirmed.png">

</p>
<p align="justify">Then, we could start the execution flow of "dccw.exe" with "ProcMon" to see in something strange occurs, something we checked immediately. At some point, if we have executed "dccw.exe" as a 64 bits process in a 64 bits Windows machine it looks for the directory "C:\Windows\System32\dccw.exe.Local\" to load a specific DLL called "GdiPlus.dll", the same as it were executed in a 32 bits Windows machine, whereas if we execute it as a 32 bits in the same machine, the process will look for the directory "C:\Windows\SysWOW64\dccw.exe.Local\". Then, due to the fact that it does not exist (fig …), the process always looks for a folder in the path "C:\Windows\WinSxS\" to get the desired DLL, this folder has a name with the following structure:</p>
[architecture]_microsoft.windows.gdiplus_[sequencial_code]_[Windows_version]_none_[sequencial_number]<br>
<br>

<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/dccw_dotLocal_notFound.png">


<p align="justify">If we take a look into "WinSxS" we could see more than one folder that matches with this structure, this means that "dccw.exe" can load the desired DLL from any of these folders. The only thing we are sure is that if the application is invoked as a 32 bits process, the folder name will start with the string "x86", while if we execute it as a 64 bits process, its name will start with the string "amd64".</p>

<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/gdiplus_folders.png">

<p align="justify">This situation can be abused to perform a DLL hijacking and then execute code with high integrity without prompting for consent.</p>

<h3>1.2. Vulnerability Verification</h3>
<p align="justify">Once we have found some error during the execution of an auto-elevate process we need to verify whether it can be abused or not. To do this we just will create the folder "dccw.exe.Local" in the desired path and into that folder we are going to create the folders located in "WinSxS" that could be invoked by the process, but without the DLL stored in that folders.</p>
<p align="justify">Now, if we execute "dccw.exe" we will see that it has found the folder "dccw.exe.Local" and one of the "WinSxS" folders, but not the desired DLL, something that throws an error. This is what we expected, due to that situation can be exploited by an attacker by means of a "DLL hijacking" and then, executing malicious code with high integrity.</p>

<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/dccw_vuln_checking.png">

<h3>1.3. Exploit Development</h3>
<p align="justify">At this point, we already know that we can perform a bypass UAC on Windows 10 abusing "dccw.exe", but how?</p>

<h4>1.3.1. Method</h4>
<p align="justify">Well, we can adapt the method developed by <a href="https://www.pretentiousname.com/misc/win7_uac_whitelist2.html" target="_blank">Leo Davidson</a> to exploit the discovered vulnerability. The problem of that method is the process injection that is performed to invoke the "IFileOperation" COM object, which can be detected by some antivirus software, so a better approach to use it is that one called "Masquerade PEB" developed by <a href="https://www.fuzzysecurity.com/tutorials/27.html" target="_blank">FuzzySecurity</a> and used by Cn33liz in its own bypass UAC.</p>
<p align="justify">Also, we have to modify the way "IFileOperation" is invoked in newer Windows 10 versions, since Leo Davidson method triggers UAC from build 15002. So the way we have to invoke such operation is the same as the original, but without the operation flags "FOF_SILENT", "FOFX_SHOWELEVATIONPROMPT" and "FOF_NOERRORUI".</p>

<h4>1.3.2.Initial Checks</h4>
<p align="justify">Before executing the exploit, it is important to check some aspects to not execute it unsuccessfully and therefore trigger some alarms. The first thing we check is the Windows build version, since some versions do not support the exploit (those with a build version lower than 7000). After that, we verify that we do not have administrator rights yet, if it is not the case, there is no reason to execute the script. Then, we check the UAC settings so as to confirm that it is not set to "Always notify", since if it were set to that value, our exploit would be useless. Finally, we have to verify whether the exploit will be executed in a 32 bits or in a 64 bits Windows machine. This final checking is performed manually.</p>

<h4>1.3.3. Interoperability</h4>
<p align="justify">When an exploit is developed is important that can work in as many systems as possible, this includes 32 bits Windows systems. To achieve this, we need to compile our exploit for such systems, since we can also execute it in 64 bits systems.</p>
<p align="justify">When our 32 bits exploit is executed in a 64 bits Windows machine, the way "dccw.exe" operates is a bit different due to the invocation of WOW64 (Windows subsystem that allows 64 bits machines to run 32 bits applications). This means the folder "dccw.exe.Local" will be looked for in "C:\Windows\SysWOW64\" directory, instead of "C:\Windows\System32\", but also the targeted "GdiPlus.dll" will be a 32 bits DLL, which implies that it will be looked for in a folder that matches with that name pattern "C:\Windows\WinSxS\x86_microsoft.windows.gdiplus_*". However, if it is executed in a 32 bits Windows system, the exploit will work as expected.</p>
<p align="justify">Finally, it is important to remark that we need to consider all the paths that matches with the pattern "C:\Windows\WinSxS\x86_microsoft.windows.gdiplus_*" when the DLL hijack is performed in order to assure a 100% of effectiveness.</p>

<h4>1.3.4. Malicious DLL</h4>
<p align="justify">To execute a process with high integrity we need to develop a DLL that invokes it via DLL hijacking. However, it is not simple as it looks, because, if we only do that, neither "dccw.exe" nor our code will be executed. This is because "dccw.exe" depends on some functions of "GdiPlus.dll", so we need to implement such functions or redirect the execution to the legit DLL.</p>
<p align="justify">The best option is forwarding the execution to the legit DLL, because in this way the size of our DLL will be lower. To do so, we use the program "ExportsToC++" to port all exports of "GdiPlus.dll" to C++ language, this will be implemented in our DLL.</p>
<p align="justify">Now, it seems that the problem has been fixed, but if we forward the execution to a specific "GdiPlus.dll" in C:\Windows\WinSxS\", the DLL will work only in specific systems, due to the name of the internal folders of "WinSxS" changes every Windows build. To overcome this problem, we came up with an elegant solution, forwarding the execution to "C:\Windows\System32\GdiPlus.dll", due to the fact that the path is the same in all Windows versions.</p>
<p align="justify">The last thing we have to do is stopping the execution of "dccw.exe" after executing our malicious code so as to avoid the window opening of that process.</p>
<p align="justify">Now, once we have developed our malicious DLL, we need to drop it in the targeted machine. To do so, our DLL has been compressed and "base64" encoded into the exploit, so that can be decoded and decompressed during its execution to drop it as expected.</p>
<p align="justify">Finally, our crafted "GdiPlus.dll" is copied to the targeted location using "IFileOperation" COM object as previously mentioned.</p>

<h4>1.3.5. Detection Avoidance</h4>
<p align="justify">When an attacker compromises a system, it wants to stay undetected as much time as possible, this means removes every hint of the actions that it performs. Because of that, all the temporary files that are created during the execution of the exploit are removed when they are not needed anymore.</p>

<h4>1.3.6. Goal</h4>
<p align="justify">Finally, we need to determine which process we want to execute with high integrity. In our case, we chose the application "cmd.exe" because it allows us to perform as many operations as we want with high integrity, but in fact, we can execute whatever application we want.</p>

<h2>2. Requirements</h2>
To get a successfully execution of the exploit the targeted machine must comply the following requirements:<br>
&emsp;- Currently, it must be a Windows 8 or 10, no matter what build version (it is expected that the bypass UAC method will work also in Windows 7, but, at this moment, not this exploit).<br>
&emsp;- The UAC settings must not be set to "Always notify".<br>
&emsp;- The compromised user must be in the "Administrator's group".<br>
&emsp;- The architecture must be verified before executing the exploit, if not, it could leave traces in the targeted machine.

<h2>3. Usage</h2>
<p align="justify">To execute the exploit you must be sure that the targeted machine meets the <a href="https://github.com/L3cr0f/DccwBypassUAC#2-requirements">requirements</a>. Also, you will have to point out its processor architecture as argument, whether it is "x86" or "x64":</p>
&emsp;- C:\Users\L3cr0f> DccwBypassUAC.exe x64<br>
&emsp;- C:\Users\L3cr0f> DccwBypassUAC.exe x86<br>
<br>

<p align="center">
<img src="https://github.com/L3cr0f/DccwBypassUAC/blob/release/Pictures/DccwBypassUAC_PoC.gif">
</p>

<h2>4. Disclaimer</h2>
<p align="justify">This exploit has been developed to show how an attacker could gain privileges into a system, not to use it for malicious purposes. This means that I do not take any responsibility if someone uses it to perform criminal activities.</p>

<h2>5. Acknowledgements</h2>
To develop the exploit, I have based on those created by:<br>
&emsp;- Fuzzysecurity: https://github.com/FuzzySecurity/PowerShell-Suite/tree/master/Bypass-UAC.<br>
&emsp;- Cn33liz: https://github.com/Cn33liz/TpmInitUACBypass.<br>
&emsp;- hFireF0X: https://github.com/hfiref0x/UACME.<br>
Many thanks to you!