**Tags:** #windows #deployment #oobe

**What is it?** 
The Out-of-Box Experience (OOBE) is the initial initialization phase and setup wizard that a user interacts with when starting a new Windows machine for the very first time.

**Usage:** 
It handles the final stages of OS configuration, specifically prompting for region selection, keyboard layout, network connection, and the creation of the initial local Administrator account or Azure AD/Entra ID join credentials.

**Why we did it:** 
We appended the `/oobe` flag to the Sysprep command (`sysprep /generalize /oobe /reboot`). By doing this, we instructed the generalized OS to boot directly into this setup wizard. This was necessary to simulate a "factory new" PC environment, allowing you to create a clean local admin account to serve as the baseline for installing GCPW and simulating a fresh user onboarding.