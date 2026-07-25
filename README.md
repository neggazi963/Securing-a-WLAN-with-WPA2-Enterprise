# Lab 5 – Securing a WLAN with WPA2 Enterprise

## Context and Objectives

This lab demonstrates the deployment of a centralized authentication infrastructure for a secure Wi‑Fi network based on the **WPA2 Enterprise** standard and **EAP‑TLS** / **PEAP‑MSCHAPv2** protocols.  
The architecture relies on an **Active Directory** directory service, an internal **Certification Authority (CA)**, and a **RADIUS (NPS)** server to enforce controlled access for wireless clients.

The initial environment uses the following machines and addresses:

| Role                          | IP Address      | Subnet Mask     |
|-------------------------------|-----------------|-----------------|
| AD (Active Directory + DNS)   | 172.20.10.1     | 255.255.255.240 |
| CA (Certification Authority)  | 172.20.10.4     |                 |
| AS (NPS / RADIUS)             | 172.20.10.5     |                 |
| Windows 10 client             | 172.20.10.7     |                 |
| Access Point (AP)             | 172.20.10.6     |                 |

**Domain:** `Mehdikc.ssi`  
**Accounts:** `student1` / `student2` (password: `P@$$w0rd`)  
**Local administrator password on workstations:** `Mehdikc.ssi` or `P@$$w0rd`



---

## Part 1 – Setting Up Active Directory and the Domain

### 1.1 Configure the AD Server 

From **Server Manager**, click the machine name in the upper‑right corner and select **Local Server**.  
Rename the server to `AD-Server` and reboot.

Configure the Ethernet interface with the following static IP settings:
- IP address: `172.20.10.1`
- Subnet mask: `255.255.255.240`
- Preferred DNS server: `127.0.0.1` (the server will act as the DNS server for the domain)

### 1.2 Install the AD DS Role

1. In **Server Manager**, select **Add Roles and Features**.
2. Proceed through the wizard until the **Server Roles** page.
3. Check **Active Directory Domain Services**, accept the addition of required features, and continue to the installation.
4. After installation completes, click the **Promote this server to a domain controller** link in the notification.

### 1.3 Promote to Domain Controller

The promotion wizard offers the option to add a new forest.  
- **Root domain name:** `example.local`  
- **Forest and domain functional level:** Windows Server 2016  
- Options checked: **DNS server** and **Global Catalog**  
- DSRM password: `Admin@1234`  

Ignore DNS warnings and proceed. The server reboots automatically; log on with `EXAMPLE\Administrator`.

### 1.4 Create Users in the Directory

From **Active Directory Users and Computers** (`example.local` > `Users`):
- Create the user **student1**:  
  `First name: student1`, `User logon name: student1`, password `Student@1234`  
  Uncheck *User must change password at next logon*, check *Password never expires*.
- Repeat for **student2**.

### 1.5 Security Group for WLAN Policies

Still in **Active Directory Users and Computers**:
- Create a new group named **WLAN-Users** (scope *Global*, type *Security*).
- Add `student1` and `student2` as members.

---

## Part 1 (continued) – Installing the Certification Authority (CA)

### 1.6 Prepare the CA Server 

Network configuration:
- Static IP: `172.20.10.4 / 255.255.255.240`
- DNS: `172.20.10.1`
- Rename the machine to `CA-Server`.

### 1.7 Join the Domain

Via **Server Manager** > **Local Server** > **Workgroup**, change the membership to the `example.local` domain (credentials `EXAMPLE\Administrator`). Reboot the server.

### 1.8 Install the AD CS Role

1. **Add Roles and Features** > **Server Roles**: check **Active Directory Certificate Services**.
2. In the role services, select both **Certification Authority** and **Certification Authority Web Enrollment**.
3. After installation, launch the configuration via the notification link **Configure Active Directory Certificate Services**.

### 1.9 Configure the Certification Authority

- Keep the default credentials (`EXAMPLE\Administrator`).
- Select the two role services mentioned above.
- Choose **Enterprise CA** then **Root CA**.
- Create a new private key (RSA 2048, default settings).
- CA name: `example-CA`, validity period: 10 years.
- Complete the configuration.

### 1.10 Certificate Template for EAP‑TLS

In the **Certification Authority** console (`example-CA` > **Certificate Templates** > **Manage**):
- Duplicate the **User** template and rename it to `WLAN-User-Cert`.
- Set the validity period to one year.
- **Subject Name** tab: select *Build from this Active Directory information* and *User principal name (UPN)*.
- **Security** tab: add the **WLAN-Users** group and grant *Read*, *Enroll*, and *Autoenroll* permissions.
- Publish the template by right‑clicking **Certificate Templates** > **New > Certificate Template to Issue** > `WLAN-User-Cert`.

---

## Part 2 – Configuring the NPS / RADIUS Server 

### 2.1 Prepare the Server

- Static IP: `172.20.10.5`, DNS: `172.20.10.1`
- Rename to `AS-Server` and join the `example.local` domain (reboot, logon `EXAMPLE\Administrator`).

### 2.2 Install the NPS Role

**Add Roles and Features** > **Network Policy and Access Services** > **Network Policy Server**. Complete the installation.

### 2.3 Register NPS in Active Directory

In the **Network Policy Server** (NPS) console, right‑click **NPS (Local)** > **Register server in Active Directory**. This step is mandatory for NPS to read user account properties.

### 2.4 Add the Access Point as a RADIUS Client

In the NPS console, under **RADIUS Clients and Servers**:
- New RADIUS client:
  - **Friendly name:** `AP-SecureWLAN`
  - **Address:** IP address of the access point 
  - **Shared Secret:** `MonSecret123` (must be configured identically on the AP)

### 2.5 Network Policy for EAP‑TLS

Create a new network policy:
- **Policy name:** `WLAN-EAP-TLS`
- **Type of network access server:** *Remote Access Server (VPN-Dial up)*
- **Condition:** *Windows Groups* → `WLAN-Users`
- **Access permission:** *Access granted*
- **Authentication methods:** remove existing methods, add **Microsoft: Smart Card or other certificate (EAP‑TLS)**.

### 2.6 Network Policy for EAP‑TTLS (PEAP‑MSCHAPv2)

Create a second policy:
- **Policy name:** `WLAN-EAP-TTLS`
- **Condition:** *Windows Groups* → `WLAN-Users`
- **Access permission:** *Access granted*
- **Authentication methods:** add **Microsoft: Protected EAP (PEAP)**, edit it to ensure the inner authentication method is **EAP‑MSCHAPv2**.

### 2.7 Obtain a Server Certificate for NPS

For the RADIUS server to establish a TLS tunnel with clients, it needs a machine certificate.

- On `AS-Server`, run `mmc` and add the **Certificates** snap‑in (*Computer account*, *Local computer*).
- Under **Personal** > **Certificates**, request a new certificate using the default **Computer** template.
- Enroll the certificate.

---

## Part 3 – Configuring the Access Point

### 3.1 Access the Management Interface

Using a web browser, connect to the AP’s administration IP address .

### 3.2 Set Up the SSID with WPA2 Enterprise

- **SSID Name:** `SecureWLAN`  
- **Security Mode:** `WPA2 Enterprise` (or `WPA2-802.1X`)  
- **RADIUS Server settings:**
  - **Server IP:** `172.20.10.5`
  - **Port:** `1812`
  - **Shared Secret:** `MonSecret123`

Apply and save the configuration.

---

## Part 4 – Configuring Windows 10 Clients

### 4.1 Obtain a User Certificate (EAP‑TLS)

The client must be joined to the `example.local` domain and logged in as `EXAMPLE\student1`.

The recommended method to obtain a certificate is:
- Open `certmgr.msc` (Win+R).
- Under **Personal** > **Certificates**, right‑click > **All Tasks** > **Request New Certificate**.
- Select the **WLAN-User-Cert** template and click **Enroll**.

*This procedure replaces the older method using the `certsrv` web interface.*

### 4.2 Connect Using EAP‑TLS

- Open Wi‑Fi settings, select the `SecureWLAN` network.
- Choose the authentication method **EAP‑TLS (Smart Card or other certificate)**.
- Select the previously installed user certificate.
- Confirm the connection.

### 4.3 Connect Using EAP‑TTLS (PEAP‑MSCHAPv2)

- For the same SSID, open the connection properties and configure **PEAP**.
- Ensure the inner authentication is **EAP‑MSCHAPv2**.
- Enter credentials: `student2` and `Student@1234`.
- Establish the connection.

---

## Part 5 – Validation and Testing

### 5.1 Examine NPS Logs

On `AS-Server`, open **Event Viewer** and navigate to **Applications and Services Logs > Microsoft > Windows > NPS**.  
Key event IDs are:
- **6272**: authentication granted  
- **6273**: authentication denied

### 5.2 Network Capture with Wireshark (Optional)

By placing a probe on the RADIUS server or a mirror port, filter for `radius` or `eap` traffic.  
Observable exchanges include *Access‑Request / Access‑Accept* pairs and EAP‑TLS dialogues with certificates.

---

## Part 6 – Comparison of EAP‑TLS and EAP‑TTLS

| Criterion                         | EAP‑TLS                                     | EAP‑TTLS (PEAP‑MSCHAPv2)               |
|-----------------------------------|---------------------------------------------|----------------------------------------|
| Client authentication             | Digital certificate                         | Username + password                    |
| Client‑side certificate required  | Yes                                         | No                                     |
| Server‑side certificate required  | Yes                                         | Yes                                    |
| Security level                    | Very high                                   | High                                   |
| Deployment complexity             | More complex (certificate management)       | Simpler                                |
| Protection against phishing       | Yes                                         | Partial (TLS tunnel protects exchange) |

**Summary of the authentication flow:**  
`Client` → EAP‑Request → `AP` → RADIUS Access‑Request → `NPS` → AD verification → RADIUS Access‑Accept → `AP` → connection granted → `Client`

---

*This documentation was produced for educational purposes. The passwords and shared secrets used here do not comply with security best practices and must be adapted for any production environment.*