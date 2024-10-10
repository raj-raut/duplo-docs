## DuploCloud Automated Duplo-Master Windows Updates

This documentation provides an end-to-end guide on how DuploCloud automates Windows updates for its duplo-master instances using AWS Systems Manager (SSM), PowerShell scripts, and a scheduled task framework. It covers everything from setting up the automation for new customers to understanding the role of each component in the architecture.

### Overview
DuploCloud automates the management of Windows updates on its `duplo-master` instances by using a combination of:
- **PowerShell scripts**: For checking, installing Windows updates, and rebooting the instance if necessary.
- **AWS Systems Manager (SSM)**: To manage and execute tasks on the Windows instances without manual intervention.
- **Scheduled Tasks**: On each Windows instance, these ensure that PowerShell scripts are run regularly.
- **Reporting Mechanism**: A farm-board service that monitors and reports the update status back to DuploCloud.

### Architecture
- **Windows Scheduled Task**: Runs every 30 minutes on each `duplo-master` instance, triggering the PowerShell script.
- **PowerShell Script**: This script checks for pending Windows updates, installs them, reboots if necessary, and reports the Git hashes of DuploCloud services.
- **Farm-board in Updates Tenant**: This service collects status updates from instances and reports to DuploCloud using an API key.
- **Mongo-Narc**: A service that logs any required reboots to `#duplo-errors` for further action.
- **Viewer Mode**: A read-only interface is available via an internal ELB (load balancer), accessible only through the DuploCloud Prod VPN.

---

## Step-by-Step Procedure to Add New Customers

### 1. Load Customer List into Windows Update Gizmo (WUG)
WUG is a Python-based tool that helps automate and monitor Windows updates for customer instances.

1. **Command**: 
   ```bash
   ./gizmo.py --load
   ```
2. **Description**: This command loads a list of customers into WUG. It ensures that WUG is aware of the instances that need to be managed.

3. **Background**: When a new customer is added, their instances (duplo-master) need to be registered with WUG. This command initializes the process and keeps track of the customer instances for further operations.

---

### 2. Plan Register for API Key Generation
Once customers are loaded, the system generates unique API keys based on EC2 instance IDs and the DNS names of the customer portals.

1. **Command**:
   ```bash
   ./gizmo.py --plan-register
   ```
2. **Description**: 
   - This command generates the API key required for each `duplo-master` instance.
   - It checks which customer instances already have the scheduled task installed and registers those without it.
   
3. **Background**:
   - API keys are crucial for secure communication between `duplo-master` instances and the farm-board service.
   - When this command is run, the system internally stores the key in a local `instance_id_cache.db` and registers the instance for future tasks.

---

### 3. Upload API Keys
After generating the API keys, they need to be uploaded to the central database for secure storage and further use.

1. **Command**:
   ```bash
   ./gizmo.py --upload-api-keys
   ```
2. **Description**:
   - This command uploads the API keys generated in the previous step to MongoDB.
   
3. **Background**:
   - For secure communication, the API keys need to be stored in a central location (MongoDB) so that the farm-board service can authenticate updates from the instances.
   - This command requires WUG credentials, which are securely stored in 1Password.

4. **VPN Requirement**:
   - You need to be connected to either the US or India DuploCloud perimeter using the **Perimeter 81 VPN** to upload the API keys.

---

### 4. Restart the Farm-board Service in Updates Tenant
Once the API keys are uploaded, the farm-board service must be restarted to pick up the newly added API keys.

1. **Step**:
   - Go to the DuploCloud portal, navigate to the **Updates tenant**, and restart the **in-service** (farm-board container).
   
2. **Background**:
   - Restarting ensures the service reloads its configuration, including the new API keys.
   - Farm-board is responsible for receiving update reports from customer instances. Without a restart, it may not authenticate newly registered instances.

---

### 5. Register the Scheduled Task
After the API keys are uploaded and the farm-board service is restarted, you can register the scheduled task on the customer instances.

1. **Command**:
   ```bash
   ./gizmo.py --register
   ```
2. **Description**:
   - This command registers the Windows Scheduled Task on each customer’s `duplo-master` instance.
   - It ensures the PowerShell script is scheduled to run every 30 minutes.

3. **Background**:
   - The Scheduled Task ensures the PowerShell script executes automatically to check for updates, install them, and report status.
   - No manual intervention is needed once the task is registered.

---

### 6. Check the Dashboard for Status
After registering the customer instances, verify that the instances are reporting their status correctly.

1. **Access**: 
   - Open the farm-board viewer interface: `https://viewer-updates.prod-apps.duplocloud.net/` (accessible only via the DuploCloud Prod VPN).

2. **Check Status**:
   - On the left panel, you will see a list of customer portals.
   - Click on a portal to view the latest update reports from that instance.

3. **Background**:
   - This viewer mode provides a read-only interface to check the status and history of updates for each customer instance.
   - It shows up to 100 records of the most recent reports.

---

## Additional Commands

### Deregister
If you need to deregister a customer instance from the Windows Update automation system, you can use the following command:

```bash
./gizmo.py --deregister
```
This removes the scheduled task from the instance and stops it from reporting updates.

---

### Restarting the In-Service (Farm-board) Service

1. **Why It's Required**:
   - The farm-board service acts as the receiver for status reports from the customer instances. 
   - After registering new API keys or making changes, the service must reload its configuration to pick up these changes.

2. **How to Restart**:
   - Go to the DuploCloud portal, find the **Updates tenant**, and restart the `in-service` docker container.

3. **Background**:
   - The restart ensures that any new API keys or configurations are loaded properly. Without this step, the system may not authenticate reports from newly registered instances.

---

## Key Components Explained

### 1. **PowerShell Script**
   - The PowerShell script checks for Windows updates and installs them.
   - It also checks whether a reboot is necessary after updates.
   - The script reports the Git hashes of the DuploCloud services and the update status back to farm-board.

### 2. **AWS Systems Manager (SSM)**
   - SSM allows for remote management and execution of tasks on EC2 instances.
   - It’s used to execute the PowerShell scripts and configure the Scheduled Tasks on `duplo-master` instances without manually logging into each one.

### 3. **Scheduled Tasks**
   - Scheduled tasks run the PowerShell script on each Windows server every 30 minutes.
   - The task is automatically registered by the WUG system after the `--register` command.

### 4. **Farm-board**
   - A service running in the Updates tenant that receives and stores update reports from customer instances.
   - It uses API keys to authenticate requests from the instances.

### 5. **Mongo-Narc**
   - This service monitors for required reboots and logs the information to the `#duplo-errors` Slack channel.
   
---

## Conclusion
By following these steps, you can add new customers to DuploCloud’s automated Windows update management system. The process involves loading customers into WUG, generating and uploading API keys, registering the scheduled task, and verifying that the customer portals are reporting updates successfully. This ensures the customer’s Windows instances remain updated and properly managed through automation.
