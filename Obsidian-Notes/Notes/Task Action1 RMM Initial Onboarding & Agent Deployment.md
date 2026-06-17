**Status:** 📅 Planned 
**Tags:** #action1 #rmm #endpoint #automation 
**Priority:** Critical (The new foundation for all PC tasks)

## 🎯 Objective

Establish Action1 as the sole endpoint management platform. The only manual task required on a new PC will be joining it to Entra ID and installing the Action1 agent. After that, Action1 takes over completely.

## 🛠️ Execution Steps

- [x] **Action1 Setup:** Log into the Action1 console. 
- [x] **Get the Agent:** Navigate to **Endpoints > Add Endpoints**. Download the Action1 deployment executable.
- [x] **Deploy to Bureau-1 & Bureau-2:** Run the Action1 installer on your test endpoints.
- [x] **Verify Connectivity:** Ensure both VMs show up as "Online" in the Action1 dashboard.
- [x] **Create Endpoint Groups:** In Action1, create a group called `Windows Workstations` and assign `Bureau-1` and `Bureau-2` to it.
![](99%20-%20Attachment/images/Task%20Action1%20RMM%20Initial%20Onboarding%20&%20Agent%20Deployment.png)