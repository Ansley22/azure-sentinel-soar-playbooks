# azure-sentinel-soar-playbooks
Automated cloud incident response framework built on Microsoft Sentinel and Azure Logic Apps with SOAR playbooks for suspicious login response with auto user disable and malicious IP response with auto IP blocking.


# 🛡️ Azure Sentinel cloud-native SOAR Incident Response Framework

An automated cloud incident response and containment framework built on **Microsoft Sentinel** and **Azure Logic Apps** that automatically triages, documents, alerts, and neutralizes active security threats in under 60 seconds without manual intervention.

---

## 🎯 Project Overview

This project demonstrates a production-grade **SOAR** (Security Orchestration, Automation, and Response) implementation inside a multi-tenant hybrid cloud infrastructure. Instead of requiring human analysts to manually investigate routine alerts, these playbooks immediately orchestrate containment workflows the exact second a threat is detected—slashing Mean Time to Remediate (MTTR) from hours to seconds.

---

## 🏗️ Architecture & Core Components

| Component | Purpose |
| :--- | :--- |
| **Microsoft Sentinel** | Cloud-native SIEM/SOAR platform providing threat detection and playbook triggering. |
| **Azure Logic Apps** | Serverless orchestration engine running the automated workflow design. |
| **Microsoft Graph API** | Programmatic identity layer used to disable compromised user access in Microsoft Entra ID. |
| **Azure Network API** | Programmatic infrastructure layer used to inject live block rules into Network Security Groups (NSGs). |
| **Gmail Connector** | Outbound notification pipeline providing formatted emergency communications to the SOC team. |
| **Microsoft Teams** | Collaborative ChatOps framework routing high-severity cards to dedicated triage channels. |
| **Managed Identities** | System-assigned passwordless authentication enforcing strict Least Privilege access. |

---

## 🔒 Permissions & Security Design Configuration

To eliminate credentials leakage risks, all workflows reject static API keys or connection strings in favor of **System-Assigned Managed Identities** configured with the following Azure Role-Based Access Control (RBAC) structures:

| Identity Scope | Required Role | Execution Function |
| :--- | :--- | :--- |
| **Log Analytics Workspace / Sentinel** | Microsoft Sentinel Responder | Incident metadata triage, status updates, timeline audit logging |
| **Microsoft Entra ID (Azure AD)** | User Administrator | Target account status manipulation (Disabling accounts) |
| **Virtual Network Resource Group** | Network Contributor | Programmatic injection of Inbound Deny firewall rules into the NSG |

---

## 🛠️ Step-by-Step Playbook Workflow Documentation

### 👤 Playbook 1: Suspicious Login Auto-Response

**Target Detections:** Password Spray Attacks, Impossible Travel Anomalies.

#### 1. Configuration & Triage Base
*   **Trigger:** Microsoft Sentinel incident trigger.
*   **Action 1 (Get Incident Details):** 
    *   *Parameters:* Linked to `soc-lab-rg` and `soc-lab-workspace`.
    *   *Dynamic Token:* `Incident ARM id` selected via the designer parameter panel lightning bolt ($⚡$).
*   **Action 2 (Add Audit Comment):** Appends an initial execution log directly to the Sentinel incident timeline to inform human analysts that automation has assumed control of the ticket.

#### 2. Communication & ChatOps Layer
*   **Action 3 (Send Email via Gmail):** Programmatically connects to the SOC inbox using the `Send email (V2)` connector. Maps the following body dynamically:
    ```text
    SECURITY ALERT - SUSPICIOUS LOGIN DETECTED
    Incident Title: [Incident title]
    Severity: [Severity]
    Status: [Status]
    Time Created: [Created time]

    Action Required:
    1. Review the incident in Microsoft Sentinel.
    2. Verify if the login is legitimate.
    ```
*   **Action 4 (Teams Notification):** Uses the `Post message in a chat or channel` action. Pushes an alert card as a `Flow bot` directly into the `SOC Lab` Team under the `#security-alerts` channel.

#### 3. Identity Containment Layer
*   **Action 5 (Extract Identity Data):** Invokes the Sentinel `Get accounts (Entities)` action, passing the incident's `Entities` array.
*   **Action 6 (Account Lockdown):** Invokes the Microsoft Entra ID `Update user` action. 
    *   *Looping Structure:* The designer automatically wraps this block inside a **For each** loop to account for multiple identity entities.
    *   *Execution parameter:* `User ID` maps to **Account User Principal Name**. The `Account Enabled` optional parameter is explicitly passed and evaluated to **No**.

---

### 🌐 Playbook 2: Malicious IP Auto-Containment

**Target Detections:** Hostile C2 Communications, Malicious IP Authentications.

#### 1. Network Enrichment Base
*   **Action 1 (Get Incident Details):** Queries the master incident metadata using the dynamic `Incident ARM id` token.
*   **Action 2 (Extract Network Entities):** Invokes the Sentinel `Get IPs (Entities)` action to parse and isolate hostile source IP addresses from the telemetry block.

#### 2. Triage Escalation & Notification
*   **Action 3 (Escalate Severity):** Programmatically interacts with the Sentinel incident management engine to elevate the incident severity to **High**.
*   **Action 4 (Gmail Broadcast):** Dispatches an automated incident report sheet containing the isolated attacker IP straight to the engineering alerts inbox.

#### 3. Perimeter Firewall Containment Layer
*   **Action 5 (Dynamic NSG Injection):** Invokes the Azure Network Security Group connector using the `Create or update a security rule` action.
    *   *Looping Structure:* Wrapped in a **For each** control loop targeting every parsed IP element.
    *   *Configuration Rules:*
        *   **Resource Group:** `soc-lab-rg`
        *   **Security Rule Name:** `Block-IP-[IP Address]`
        *   **Access:** Deny
        *   **Direction:** Inbound
        *   **Protocol:** * (Any)
        *   **Source IP Address:** Dynamically assigned to the extracted **IP Address** entity token.

---

## ⚙️ Automation Routing Rules (The Gatekeepers)

These engine rules evaluate incoming system telemetry in real time, serving as the conditional traffic controllers that decide exactly when to run a specific containment playbook.

```text
[Incoming Azure Security Signal]
               │
               ▼
   [Microsoft Sentinel Engine] ── Creates Incident ──► [Automation Rules Evaluation]
                                                               │
                    ┌──────────────────────────────────────────┴──────────────────────────────────────────┐
                    ▼                                                                                     ▼
     Matches: "Suspicious Login Response"                                                  Matches: "Malicious IP Connection"
                    │                                                                                     │
                    ▼                                                                                     ▼
   Execute Playbook-SuspiciousLogin-Response                                             Execute Playbook-MaliciousIP-Response
                    │                                                                                     │
                    ▼                                                                                     ▼
    [Lock User + Email + Teams Alert]                                                     [Block IP + Escalate + Teams Alert]
