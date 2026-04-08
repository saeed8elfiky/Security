## Technical Documentation: Malware Behavior and Removal

This document outlines the execution behavior and manual remediation steps for the identified executable.

---

### 1. Execution Behavior

Upon execution of the exported file, the program performs a single visible action:

- **User Notification:** A message box will trigger on the screen with the following text:
    
    > "You are Hacked Bro!"
    > 

This popup serves as a confirmation that the script or executable has successfully run on the system.

---

### 2. Persistence Mechanism

To ensure the program runs automatically every time the user logs into Windows, it modifies the system registry. It creates an entry in the following location:

**Registry Path:**

`HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`

This "Run" key is a common method for software (and malware) to maintain persistence without manual user intervention.

---

### 3. Manual Removal Instructions

To completely remove the entry and prevent the program from launching on startup, follow these steps:

1. Press `Win + R`, type **regedit**, and press **Enter** to open the Registry Editor.
2. Navigate to the following path:
    
    `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
    
3. Locate the value corresponding to the malware in the right-hand pane.
4. **Right-click** the entry and select **Delete**.
5. Confirm the deletion and restart your computer to verify the popup no longer appears.

---

> **Note:** Always exercise caution when editing the Windows Registry. Deleting the wrong key can cause system instability. Ensure you have identified the specific entry related to this program before removal.
>
