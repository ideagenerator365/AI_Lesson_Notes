# 5G Core Network Function (NF) Deep Dive Analysis

This document provides a detailed, **cloud-native** operational analysis of key **5G Core** and related network functions. The focus is on the **Service-Based Architecture (SBA)**, **NF Statefulness** (crucial for cloud scalability), and **Distributed Tracing** procedures for modern network troubleshooting.

---

## 1. Packet Core

These components form the backbone of 5G connectivity, session, data, and policy management.

| Acronym & Main Description | Key Functional Areas | 2G/3G/4G Equivalent | Statefulness | Tracing (Example Flow) |
| :--- | :--- | :--- | :--- | :--- |
| **AMF** (Access and Mobility Management Function) | Registration Management, Connection Management (CM-IDLE, CM-CONNECTED). | MME | **Stateful** (Must maintain UE mobility and connection context). | UE → gNB → **AMF** (Authentication) → AUSF → UDM |
| **SMF** (Session Management Function) | PDU Session Management, IP Address Assignment, UPF Selection. | PGW-C and SGW-C | **Stateful** (Maintains the complete state of every active PDU Session). | AMF → **SMF** → UPF (N4 Session Setup) → PCF |
| **UPF** (User Plane Function) | Packet Routing & Forwarding (GTP-U/IP), Traffic Policy Enforcement (QoS, Charging). | PGW-U and SGW-U | **Stateless** (All logic is programmed by the SMF via N4). | SMF → **UPF** (Tunnel ID) → gNB → Data Network |
| **UDM** (Unified Data Management) | Authentication Credentials Storage, User Subscription Data. | HLR/HSS | **Stateful** (Requires high-availability persistent storage). | AMF → **UDM** (Subscription Retrieval) → AMF |
| **UDR** (User Data Repository) | Centralized, Structured Storage for Subscription and Policy Data. | Part of HSS logical function (separated for cloud scalability). | **Stateful** (The central database for all persistent information). | UDM → **UDR** (Nudr\_DR\_Query) → UDM |
| **PCF** (Policy Control Function) | Policy Decision (QoS, Access Rules), Policy Delivery to SMF. | PCRF | **Stateful** (Must maintain session binding and policy state). | UDM → **PCF** (Retrieve Policy) → SMF |
| **CHF** (Charging Function) | Converged Charging (Online/Offline), Charging Data Collection. | OCS and CDF | **Stateful** (Maintains active credit control session state for online charging). | SMF → **CHF** (Credit Request) → SMF |
| **NRF** (Network Repository Function) | NF Registration (NFManagement), NF Discovery (Service Endpoint Lookup). | None (Fundamental new NF for SBA). | **Stateful** (Requires persistent database to store NF profiles). | AMF → **NRF** (NFDiscovery) → AMF → SMF |

---

## 2. IMS - Voice (VoNR/VoLTE)

Components essential for delivering Voice over IP (VoIP) and other multimedia services.

| Acronym & Main Description | Key Functional Areas | 2G/3G/4G Equivalent | Statefulness | Tracing (Example Flow) |
| :--- | :--- | :--- | :--- | :--- |
| **CSCF** (Call Session Control Function) | SIP Session Control, Registration, Routing. | CSCF | **Stateful** (S-CSCF is highly stateful, maintaining registration and service profile). | P-CSCF → I-CSCF → **S-CSCF** → TAS |
| **TAS** (Telephony Application Server) | Hosts Supplementary Voice Services Logic (Call Waiting/Forwarding). | TAS | **Stateful** (Maintains the state of ongoing service logic execution). | S-CSCF → **TAS** (Service Execution) → S-CSCF |
| **SCC-AS** (Service Centralization and Continuity Application Server) | Voice Call Continuity (SRVCC/VoLTE), Access Transfer Control. | SCC-AS | **Stateful** (Maintains anchor point during handover). | MME → **SCC-AS** (Anchor Update) → MSC |
| **MRF** (Media Resource Function) | Media Processing (Conferencing, Announcements, Tone Generation). | MRF | **Stateless** (Media processing is transactional, but conference state is maintained). | S-CSCF → **MRF** (Mg Request) → SBC |
| **SBC** (Session Border Controller) | Security, Interworking (NAT), Topology Hiding. | SBC | **Stateful** (Maintains call state for security and media session establishment). | External IP Network → **SBC** → P-CSCF |

---

## 3. Messaging (SMS/MMS/VM)

Components for the delivery of traditional and IP-based text and multimedia messages.

| Acronym & Main Description | Key Functional Areas | 2G/3G/4G Equivalent | Statefulness | Tracing (Example Flow) |
| :--- | :--- | :--- | :--- | :--- |
| **smsc** (Short Message Service Center) | SMS Processing and Delivery (Store and Forward). | SMS-SC (Legacy) | **Stateful** (Must store messages and delivery state). | UE → MSC/SGSN → **smsc** → HLR → UE |
| **mmsc** (Multimedia Message Service Center) | MMS Handling, Content Adaptation. | MMSC (Legacy) | **Stateful** (Must store multimedia messages). | UE → GGSN/PGW → **mmsc** → UE |
| **ipsm-gw** (IP Short Message Gateway) | Facilitates SMS transmission over the IMS/IP network. | GMSC/SMS-IWMSC | **Stateful** (Maintains correlation between IP session and legacy SMS transaction). | SMF → **ipsm-gw** → smsc |

---

## 4. Roaming and Support Architecture

Components for securing inter-operator signaling and optimizing the signaling flow.

| Acronym & Main Description | Key Functional Areas | 2G/3G/4G Equivalent | Statefulness | Tracing (Example Flow) |
| :--- | :--- | :--- | :--- | :--- |
| **SEPP** (Security Edge Protection Proxy) | Secures SBA inter-operator signaling (**N32**), Topology Hiding. | DEA (Diameter Edge Agent) | **Stateless** (Acts as a secure proxy). | V-NRF → V-**SEPP** → H-**SEPP** → H-NRF |
| **AUSF** (Authentication Server Function) | Primary Authentication Execution (5G EAP-AKA), Key Derivation. | Integrated within HLR/HSS | **Stateless** (Performs a calculation; no persistent session state is maintained). | AMF → **AUSF** → UDM → **AUSF** → AMF |
| **SCP** (Service Communication Proxy) | **API Gateway/Central Router** for all SBA signaling traffic, Load Balancing. | DRA (Diameter Routing Agent) | **Stateless** (Acts as a routing proxy). | AMF → **SCP** (Route to SMF) → SMF |
| **BSF** (Binding Support Function) | Stores the binding information (PDU Session $\to$ PCF) for policy control plane routing. | None (Auxiliary SBA function). | **Stateless** (Acts as a temporary key-value lookup service). | SMF → **BSF** (Store Binding) → PCF → **BSF** |
| **HSS** (Home Subscriber Server) | Master user database for **legacy 2G/3G/4G/VoLTE** networks. | HSS (Master user database) | **Stateful** (The central, persistent database for all legacy user profiles). | MME → **HSS** (Attach Request - Diameter) → MME |











Herramientas

