**Tags:** #windows #deployment #sysprep #generalize

**What is it?** 
Sysprep (`sysprep.exe`) is Microsoft's native deployment utility used to prepare an installation of Windows for imaging or delivery. The `/generalize` switch is the specific function that forcefully removes all unique system information, stripping the OS down to a hardware-agnostic baseline.

**Usage:** 
It is strictly used by system administrators to create "Golden Images" (templates). Before shutting down a configured VM to be cloned, Sysprep must be executed so that any future clones generated from that disk spin up without inheriting the original machine's identity.

**Why we did it:** 
Because the clone was made _without_ generalizing first, the system was tainted with duplicated identities. We had to run `C:\Windows\System32\Sysprep\sysprep.exe /generalize` post-cloning while offline to forcefully scrub the duplicated SID, wipe the event logs, uninstall device-specific drivers, and strip the Entra ID device state. This forced `Bureau-1` to generate a fresh, mathematically unique identity upon its next boot.