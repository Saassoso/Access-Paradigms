MAke a local account 
- change the name dependeing on the bureau : material 
	- `Rename-Computer -NewName "Bureau-1-001" -Force`
- Install Action 1 Agent :
	- `curl -o "action1_agent(PFE-Inter).msi" "https://app.eu.action1.com/agent/58143174-3e20-11f1-9520-bbb5239f219c/Windows/agent(PFE-Inter).msi" && msiexec /i "action1_agent(PFE-Inter).msi" /quiet /qn`