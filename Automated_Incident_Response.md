# Automated SOC Incident Response Workflow
## Overview
This project is an automated SOC incident response workflow designed to improve security operations efficiency by reducing manual alert handling and accelerating incident containment. The workflow was developed using n8n and integrates with Elasticsearch, and FortiGate firewalls.
The automation continuously monitors security alerts, correlates them with existing cases, removes duplicates, and automatically performs containment actions for critical incidents.

## Technologies Used

- n8n
- Elasticsearch
- Kibana
- FortiGate API

## Workflow Process
1. **Schedule Trigger:**
The workflow starts automatically at predefined intervals using a Schedule Trigger node Every 15 Minutes. Continuously monitor the environment, and ensure alerts are processed in near real-time

2. **Request Open Alerts:**
The workflow sends requests to Elasticsearch using *elasticsearch api* `https://<elasticsearch_ip>:9200/.internal.alerts-security.alerts-default-*/_search`
and to retrieve all currently open alerts.
    ```json
    {
      "query": {
        "bool": {
          "must": [
            {
              "range": {
                "@timestamp": {
                  "gte": "now/d",
                  "lt": "now"
                }
              }
            },
            {
              "query_string": {
                "query": "signal.status: open"
              }
            }
          ]
        }
      }
    }
    ```
3. **Retrieve Existing Cases:**
At the same time, the workflow pulls all open cases from the Case Management system in Kibana using *kibana api* `https://<kibana_ip>:5601/api/cases/_find`.
Compare incoming alerts with active investigations to prevent duplicate case creation

4. **Split Alerts and Cases:**
The workflow separates alerts and cases into individual items for processing. Process each alert independently
Also enable alert-by-alert correlation and filtering

5. **[IF Node] Severity Validation:**
The workflow checks the severity level of each alert.
Conditions: `Critical`
Ensuring only critical alerts continue through the workflow.

6. **[Merge-Node] Duplicate Detection & Correlation:**
    The workflow compares incoming alerts against existing cases and removes duplicate IPs or repeated alerts.
    - Avoid duplicate incidents
    - Prevent repeated response actions on the same host
    - Improve SOC operational efficiency

7. **[Create Case Node] Case Management Logic:**
    If No Existing Case Exists = `A new case is automatically created`
    If a Related Case Exists = `The alert is added to the existing case`
    - Maintain organized investigations
    - Centralize related alerts under one incident
8. **FortiGate Automated Response:**
If the alert source is identified as FortiGate, additional response actions are triggered.

    - Step 1 — Retrieve Device Information using *FortiGate API* `https://<FortiGate_IP>/api/v2/monitor/user/device/query`
      ```json
      {
          "filters": [
            [
              "ipv4_address",
              "==",
              "{{ $json._source['kibana.alert.threshold_result'].terms[0].value }}"
            ]
          ]
        } 
      ```

        - The workflow retrieves detailed host information from FortiGate.
        - Data Collected (Device IP, MAC Address, Host Information
    
    - Step 2 — Create Address Object `https:///<FortiGate_IP>/api/v2/cmdb/firewall/address`
      ```
      {
          "name": "INFECTED-{{ $json.results[0].ipv4_address }}",
          "subnet": "{{ $json.results[0].ipv4_address }}/24"
        }
      ```

        - A dynamic address object is created on the FortiGate firewall.

    - Step 3 — Quarantine Host `https:/<FortiGate_IP>/api/v2/cmdb/firewall/addrgrp/QUARANTINE`
      ```
      {
          "member": [
            {
              "name": "{{ $json.mkey }}"
            }
          ]
        }
      ```

        - The affected host is automatically added to a quarantine group or isolation policy.

9. **Notification System:**
The workflow sends automated notifications after quarantine actions are completed.

    - Inform SOC analysts about response status
    - Provide visibility into automated containment actions
  
## Experimentally
1. **Creating Policy to Block `Facbook.com`**
    - This policy acts as a security control layer to enforce application restriction at the edge.
   <p align="center">
      <img src="/images/n8n_incident_Response/facebook_block.png" width="900">
   </p>
2. **Attach it to the LAN Policy**
    - The block rule is associated with the LAN → WAN policy chain to ensure internal users are affected when they attempt outbound traffic.
   <p align="center">
      <img src="/images/n8n_incident_Response/attach_facebook_policy.png" width="900">
   </p>
3. **Create Address group**
    - An address group is created to logically group affected hosts (e.g., quarantined endpoints, suspicious devices, or users under restriction).
   <p align="center">
      <img src="/images/n8n_incident_Response/create_Address_group.png" width="900">
   </p>
4. **Create `QUARANTINE` Policy and attatch the Address Group to it.**
    - A dedicated quarantine policy is created to isolate or restrict compromised or suspicious devices.
   <p align="center">
      <img src="/images/n8n_incident_Response/create_quarantine_policy.png" width="900">
   </p>
> [!NOTE]
> The QUARANTINE policy must be placed at the top of the firewall policy list to ensure it takes priority over general LAN → WAN allow rules due to FortiGate’s top-down rule evaluation model.

5. **Try to access `facebook.com`**
   <p align="center">
      <img src="/images/n8n_incident_Response/win7_oisolated.png" width="900">
   </p>
6. **Create Detection Rule using threshold method**
    - triggers an alert when a single device attempts to access blocked websites multiple times, exceeding a set threshold.
    - **Trigger Condition:** It looks for logs where the block reason is "Local URLfilter Block", aggregated by source.ip.
    - **Threshold:** An alert fires if a single IP address generates 10 or more violations.
   <p align="center">
      <img src="/images/n8n_incident_Response/web_detection_rule.png" width="900">
   </p>

7. **Starting n8n Workflow**
   <p align="center">
      <img src="/images/n8n_incident_Response/n8n.png" width="900">
   </p>
8. **Check Case Creation and Isolating Process**
    - The workflow automatically creates an incident case within Kibana and attaches the relevant alert data for the analyst.
   <p align="center">
      <img src="/images/n8n_incident_Response/case.png" width="900">
   </p>
9. **Target Eliminated**
    - The playbook executes a containment action, successfully isolating the compromised or violating host machine from the network to prevent further risk.
    <p align="center">
      <img src="/images/n8n_incident_Response/target_isolated.png" width="900">
   </p>
