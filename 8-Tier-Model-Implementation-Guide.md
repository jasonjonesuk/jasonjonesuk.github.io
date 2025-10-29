# Tier Model Implementation Guide

This document provides step-by-step instructions to deploy the ADDS and Tier Model solution into the [CUSTOMER] environment. The overall solution design is out of scope for this document and is detailed in a separate document.

## Audience

The audience for this document includes deployment teams within [CUSTOMER]. It is expected that any reader of this document has a sufficient background and knowledge of Microsoft Windows Server, Active Directory, PowerShell, Microsoft Deployment Toolkit, and related Microsoft technologies.

## References

The contents of this document are based on the following documents/references and email communications:

| Document | Description |
| ---------------- | ---------------- |
| Tier Model - Design and Plan - ##H#.docx | Detailed design and plan for the Tier Model solution. |
| Tier Model – Configuration Settings - ##H#.xlsx | Spreadsheet to track post Tier Model deployment decisions and configurations. |

## General Delivery Guidance

This section provides delivery guidance for the Active Directory Domain Services (ADDS) Tier Model and Deployment Component.

### Tier Model Components

This section details the components required to prepare the ADDS for the Tier Model.

| Item | Description | Part of Core Component |
| ---------------- | ---------------- | ---------------- |
| Dependent component | None | NA |
| Technical Supporting Files required | TierModel-##H#.zip (Provided by Microsoft) | Mandatory |
| Change Required to Production environment | Extension of Active Directory Domain Services Schema for WINDOWS Local Administrator Password Solution (LAPS) support | Optional |
| Change Required to Production environment | Addition of Organizational Unit (OU) structures, users, groups, and group policies to Active Directory Domain Services | Mandatory |
| Privileged Access Workstation (PAW) | Clean source workstation from which to execute changes to the production environment | Mandatory - Tier Model does not function correctly without a Privileged Access Workstation (PAW) device |
| Domain Admin Access to Production environment | Domain Admin access is required to execute the Tier Model deployment PowerShell scripts | Mandatory |
| PowerShell 5.1 | PowerShell 5.1 and Windows Server 2016 or later | Mandatory |

> **IMPORTANT**: PowerShell 6.x and 7.x are not supported for deployment at this time.

## Deployment

This section details the creation of the Active Directory objects in the production environment.

### Prerequisites

The table below details the prerequisites for this section.

| Prerequisite | Description |
| ---------------- | ---------------- |
| TierModel-##H#.zip | Provided by Microsoft |
| Change control for extending the production Active Directory schema for Windows LAPS | Approved by customer |
| Change control for modifying the production Active Directory Domain Services for new objects | Approved by customer |
| If a previous Tier Model has already been deployed, the migration scripts must be executed before deploying the new Tier Model structure. See Appendix for more information. | Approved by customer |

### Deploying the Tier Model

From a clean source workstation, follow the steps below:

If the Domain does not have Windows DNS servers running then the DnsAdmin and DnsUpdateProxy groups do not exist in Active Directory, see Appendix for Tier Model deployment steps.

1. Log on to the Clean Keyboard Workstation as an account that is a member of **Domain Admins**.

1. Save the Microsoft-provided **Tier0TmisScripts-v#.#.zip** to the root of the C drive. Update the version #.# before executing the PowerShell cmdlet to unblock and extract the archive.

    ```powershell
    Unblock-File -Path "C:\ADDS-TierModelAndDeployment-SignedFiles-MM-DD-YYYY"
    Expand-Archive -Path "C:\ADDS-TierModelAndDeployment-SignedFiles-MM-DD-YYYY" -DestinationPath "C:\"
    ```

    The `Tier0Scripts`, `TierModel`, and `TMIS` directories should now be present on the root of the C drive.

    ![Extracted Folders](../.images/TierModel/ImplementationGuide/ExtractedFolders.png)

    ![Child Extracted Folders](../.images/TierModel/ImplementationGuide/ChildExtractedFolders.png)

1. Open an Administrative PowerShell Command Prompt. Type and execute the following command after replacing **##H#** with the current version of the Tier Model provided by Microsoft:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\TM-##H#"
    .\Manage-AdObjects.ps1 -Restore -SettingsXml settings.xml -OU -Group -User -GPO All -GpoLinks LinkEnabled -WmiFilter -Permissions -Admx
    ```

1. Confirm the execution of the script by typing **Yes**.

1. Open Active Directory Users and Computers.

1. Navigate to _**Tier Model Administration\\Tier 0\\Tier 0 Service Accounts**_.

1. Right-click the **svc-pawdomainjoin** account and choose "Reset Password."

1. Set the password as desired.

1. Navigate to _**Tier Model Administration\\Tier 1\\Tier 1 Service Accounts**_.

1. Right-click the **svc-t1srvdomainjoin** account and choose "Reset Password."

1. Set the password as desired.

1. Disable inheritance on all the newly created Tier Model OUs by running the following script:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    .\Disable-TierModelInheritance.ps1
    ```

    ---

    **NOTE:** The script won't produce any output.

    ---

### Redirect Default User and Computer Containers

#### Explanation of Redirecting Default Computer Container

By default, when a new computer is joined to an Active Directory domain, it is placed into the built-in _Computers_ container. Redirecting the default computer container changes this behavior, automatically placing newly joined computers into a specified Organizational Unit (OU). This approach simplifies management by ensuring new computer objects receive appropriate Group Policy settings and security configurations immediately upon joining the domain.

#### Explanation of Redirecting Default Users

By default, when a new user is created in an Active Directory domain, it is placed into the built-in _Users_ container. Redirecting the default user container changes this behavior, automatically placing newly created users into a specified Organizational Unit (OU). This approach simplifies management by ensuring new user objects receive appropriate Group Policy settings and security configurations immediately upon creation.

> **IMPORTANT**: Before performing this step, confirm with the customer as this can impact automated server deployments along with user and client provisioning solutions.

From a clean source workstation, follow the steps below:

1. Log on to the Clean Keyboard Workstation as an account that is a member of **Domain Admins**.

1. Open PowerShell as an Administrator and run the following cmdlet to redirect users and computer containers:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    .\Redirect-DefaultContainers.ps1 -RedirectUsers $true -RedirectComputers $true
    ```

### Windows LAPS

#### Explanation of Windows LAPS

Windows Local Administrator Password Solution (LAPS) is a Windows feature that automatically manages and backs up the password of a local administrator account on your Microsoft Entra-joined or Windows Server Active Directory-joined devices. You can also use Windows LAPS to automatically manage and back up the Directory Services Restore Mode (DSRM) account password on your Windows Server Active Directory domain controllers.

From a clean source workstation, follow the steps below:

1. Use Remote Desktop Protocol (RDP) to connect to a Domain Controller that is at least Windows Server 2019 or later from the clean keyboard workstation.

1. Add the current user to the **Schema Admins** group.

1. Refresh the token by **logging off** and RDP back into the Domain Controller.

1. Open PowerShell as an Administrator and run the following cmdlet for Windows LAPS:

    ```powershell
    Import-Module LAPS
    Update-LapsADSchema
    ```

1. Press 'A' to confirm the schema extension.

1. Once the schema is successfully extended, remove the user account from the **Schema Admins** group.

1. Open an Administrative PowerShell Command Prompt. Type and execute the following command after replacing **DC01.contoso.com**:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    .\Deploy-WindowsLaps.ps1 -PreferredDC DC01.contoso.com
    ```

### Authentication Silos

#### What are Authentication Silos?

Authentication Silos are a security feature within Active Directory that create security boundaries designed to isolate and protect privileged accounts. They act as security perimeters that:

- Restrict where privileged accounts (e.g., Tier 0 accounts) can log in.
- Ensure these accounts are only used on authorized computers within the same tier.
- Prevent credential theft attacks by limiting exposure when lower-security systems are compromised.

#### How the Scheduled Task Enhances Security

A scheduled task is used to automate the management of Authentication Silos, ensuring consistent security. This task performs the following actions:

1. **Adds Accounts to the Silo**: Automatically includes user accounts from the `Tier 0\Tier 0 Accounts` Organizational Unit (OU) into the Tier 0 Authentication Silo.
2. **Removes Unauthorized Accounts**: Removes accounts from the Silo if they no longer belong in the Tier 0 OU.
3. **Secures Tier 0 Accounts**:
    - Marks accounts as sensitive.
    - Prevents credential delegation.
    - Adds accounts to the "Protected Users" security group.

> **Note:** Managed Service Accounts (MSA, gMSA, dMSA) are not automatically handled by the task. You must manually add these accounts to the Tier 0 Authentication Silo policy.

This automation helps maintain strong security boundaries between administrative tiers, reducing the risk of credential theft and unauthorized access.

From a clean source workstation, follow the steps below:

1. RDP to a Domain Controller that is at least Windows Server 2019 or later from the clean keyboard workstation.

1. Add the current user to the **Enterprise Admins** group.

1. Refresh the token by **logging off** and RDP back into the Domain Controller.

1. Open an Administrative PowerShell Command Prompt. Type and execute the following command after replacing **DC01.contoso.com**:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    .\Deploy-TierModelAuthSilo.ps1 -PreferredDC DC01.contoso.com
    ```

1. Copy the TierModelAuthSilo scripts to the root of the C: drive that will be used within scheduled tasks:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    New-Item -Path "C:\TierModelAuthSilos" -ItemType Directory -Force
    Copy-Item -Path .\TierModelAuthSilos\*.ps1 -Destination C:\TierModelAuthSilos -Recurse
    ```

1. Run the following commands to import the local scheduled tasks for Tier 0. GPO Schedule Task and Tier 1 Auth Silo details can be found in the Appendix.

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    $Tier0Task = Get-Content ".\TierModelAuthSilos\ScheduleTask-Local\Tier 0 Auth Silo.xml" | Out-String
    Register-ScheduledTask -Xml $Tier0Task -TaskName "Tier 0 Auth Silo" -Force
    ```

---

**NOTE:** The Appendix contains additional details related to Authentication Silo, such as Tier 1, storing scripts in NETLOGON, using a GPO to schedule the task, and multi-forest mode.

---

### Post Deployment Configuration

The initial deployment of the Tier Model into Active Directory is just to get the structure in place. Think of it as building a new building—the walls are built first, but doors and locks still need to be installed to secure access to the building.

During the workshops, Microsoft presented the Tier Model - Configuration Settings - ##H#.xlsx Excel sheet and captured decisions. Those decisions should now be implemented before starting to use the Tier Model.

From a clean source workstation, follow the steps below:

1. RDP to a Domain Controller that is at least Windows Server 2019 or later from the clean keyboard workstation.

1. Link-enable GPOs within the Tier Model.

1. Perform any cleanup activities.

1. The ADH process should be performed by Microsoft, Partner, or Customer before the Tier Model is trusted for daily operational use.

1. The PAW device deployment process should be performed by Microsoft, Partner, or Customer. For the Tier Model to function correctly, PAW devices must be deployed.

### Known Issues

This section details known issues:

- Working with Windows 2008 R2 domains: These can run the Tier Model, but the script must be run from a Windows 6.3 or more recent computer (Windows 2012 R2/Windows 8.1 with PowerShell 4).

## Appendix

Additional documentation related to the Tier Model.

### No Windows DNS

If there are no Windows DNS deployed into the Domain then there will be two Tier 0 groups no present in Active Directory which are DnsAdmin and DnsUpdateProxy. This will cause the GPOs to fail to import. A version of the Tier Model can be import that does not contain these two groups in the Deny User Rights Assignments. After extract execute the following command instead:

1. Open an Administrative PowerShell Command Prompt. Type and execute the following command after replacing **##H#** with the current version of the Tier Model provided by Microsoft:

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\TM-##H#"
    .\Manage-AdObjects.ps1 -Restore -SettingsXml settings-nodns.xml -OU -Group -User -GPO All -GpoLinks LinkEnabled -WmiFilter -Permissions -Admx
    ```

### Legacy Tier Model Migration

If a previous version of the Tier Model has been deployed into the domain, then before deploying the latest version of the Tier Model, some of the Groups and GPOs must be renamed to avoid conflicts during deployment. RM stands for Remove. After migration, these AD objects can safely be deleted.

```powershell
Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
.\Migrate-LegacyTierModel.ps1 -PreferredDC DC01.contoso.com
```

- Groups will be renamed with a new prefix: RM_
  - Example: Tier 1 Admins will become RM_Tier 1 Admins.
- GPOs will be renamed with a new prefix: *RM-
  - Example: *- Tier 0 PAWs will become RM- Tier 0 PAWs.
- After execution of the script:
  - Move users from the RM_Tier # group members to the same name Tier # group members.
  - Move T#-Accounts into the new OU structure.
  - Move T#-Devices into the new OU structure.
  - Move any T#-Groups that were not renamed RM_ (e.g., T0-ADCS-Admins or T1-FileServer-Admins added by the customer).
  - Move any T#-Service accounts to the new OU structure.
  - Delete RM_ groups after they are empty.
  - Unlink GPO *RM- GPOs.
  - Delete *RM- GPOs that are not linked to any OUs.
  - Delete all previous Legacy Tier Model OUs after confirming all AD objects have been migrated.

---

**NOTE:** This script has only been tested for migration from the latest Legacy Version of the Tier Model IP Kit, which was version 23H1. Previous versions may generate errors if specific Groups or GPOs do not exist. It is advised to test the migration in a lab by using the ManageObjects.ps1 script to back up the current production domain, deploy it into a lab, and then test the migration.

---

### Tier 1 Authentication Silo Schedule Task

If Tier 1 will also be using Authentication Silo Policies and the same local scheduled task method, execute the following PowerShell to import the local scheduled task:

1. Import the local scheduled tasks for Tier 1.

    ```powershell
    Set-Location -Path "C:\ADDS-TierModelAndDeployment\TierModel\"
    $Tier1Task = Get-Content ".\TierModelAuthSilos\ScheduleTask-Local\Tier 1 Auth Silo.xml" | Out-String
    Register-ScheduledTask -Xml $Tier1Task -TaskName "Tier 1 Auth Silo" -Force
    ```

### Tier Model Authentication Silo GPO Schedule Task

In large Active Directory domains with hundreds of Domain Controllers, it might require a scheduled task to run on more than one Domain Controller or on all Domain Controllers. There are a couple of decisions to make, and this document will serve as a guide.

- Will the scripts be stored on the local C drive of all Domain Controllers?
- How will the script files be copied to all Domain Controllers?
- Instead of local script files, could the files be safely stored in the NETLOGON directory, which is replicated to all Domain Controllers?
- Ensure the scripts are signed.

A GPO has already been exported that contains both Tier 0 and Tier 1 scheduled tasks, which can be found in the **C:\\TierModel\\ScheduleTask-GPO** directory.

- Re-use the ***- Tier 0 DCs Authentication Silo** GPO and import the settings into this GPO. This GPO will contain both the Kerberos arming and scheduled tasks, expecting the scripts to be located on the C drive of every Domain Controller.
- If NETLOGON will be used, copy the folder from the C drive to the NETLOGON share and then update the GPO scheduled tasks to point to the NETLOGON path instead of the C drive.
- Other changes, such as the default trigger (10 minutes), can be increased or decreased as needed.

### Multi-Forest Tier Model Authentication Silos

All of the Update-Tier 0 and Tier 1 scripts for Authentication Silos have an additional parameter called `-MultiDomainForest`, which accepts a value of `False` (default) or `True`. In this configuration, all Tier 0 user accounts are in the root of the domain and are used to control the child domains. All Tier 0 and Tier 1 computer objects in the child domains are nested in the root domain groups. This would require only the root domain to have the scheduled tasks.
