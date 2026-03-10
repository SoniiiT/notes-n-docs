---
tags:
  - index
  - microsoft-365
  - azure
  - cloud
  - m365
---

# Microsoft 365 & Azure Index

> [!ABSTRACT] Übersicht
> Vollständige Dokumentationsroadmap für das Microsoft Cloud-Ökosystem — Microsoft 365, Azure und alle integrierten Dienste. Von Identität und Kollaboration über Infrastruktur und Sicherheit bis hin zu Governance und Automatisierung.

---

## 🗂️ Microsoft 365 Grundlagen

| Thema | Beschreibung |
|-------|-------------|
| [[Was ist Microsoft 365]] | Überblick, Architektur, Cloud-Modelle (SaaS/PaaS/IaaS), Abgrenzung zu Azure |
| [[M365 Lizenzmodelle und Pläne]] | E1/E3/E5, Business Basic/Standard/Premium, F-Pläne, Add-ons, Vergleich |
| [[M365 Tenant-Verwaltung]] | Tenant erstellen, Domains hinzufügen, Multi-Tenant, Tenant-Einstellungen |
| [[M365 Admin Center]] | Navigation, Rollen, delegierte Administration, Unteradmin-Center |
| [[Microsoft 365 Apps]] | Word/Excel/PowerPoint Cloud, Click-to-Run, Update-Kanäle, Deployment |
| [[Microsoft Lists]] | Listentypen, Integration in Teams/SharePoint, Automatisierung |
| [[Microsoft Forms]] | Formulare, Quizze, Umfragen, Ergebnisanalyse |
| [[Microsoft Bookings]] | Terminbuchung, Staff Management, Integration mit Teams |
| [[Microsoft Whiteboard]] | Kollaboratives Whiteboard, Teams-Integration |
| [[Microsoft Stream]] | Videoplattform, Recording-Aufbewahrung, SharePoint-Integration |
| [[Microsoft Loop]] | Loop Components, Seiten, Workspaces, Co-Creation |
| [[Microsoft Planner und Project]] | Planner (einfach), Project for the Web (komplex), Roadmap |
| [[Microsoft Viva Connections]] | Intranet-Hub in Teams, Adaptive Cards, Dashboard |
| [[Microsoft Viva Engage]] | Enterprise Social Network (ehem. Yammer), Communities, Stories |
| [[Microsoft Viva Insights]] | Produktivitätsanalysen, Wellbeing, Manager-Insights |
| [[Microsoft Viva Learning]] | Lernplattform, LMS-Integration, LinkedIn Learning |
| [[Microsoft Viva Goals]] | OKR-Management, Ziele und Ergebnisse |
| [[Microsoft Viva Amplify]] | Interne Kommunikation, Kampagnen, Multi-Kanal-Publishing (Teams, SharePoint, E-Mail) |
| [[Microsoft Viva Pulse]] | Mitarbeiter-Feedback, kurze Umfragen, Manager-Insights, Anonymität |
| [[Microsoft Copilot for M365]] | KI-Assistent, Prompting, Governance, Datenzugriff, Lizenzierung |

---

## 🔐 Identität & Zugriff (Microsoft Entra)

| Thema | Beschreibung |
|-------|-------------|
| [[Microsoft Entra ID Grundlagen]] | ehem. Azure AD — Objekte, Verzeichnisstruktur, Lizenzierung |
| [[Benutzer und Gruppen]] | Erstellen, Gruppentypen (M365, Security, Distribution), dynamische Gruppen |
| [[Multi-Factor Authentication]] | MFA-Methoden, SSPR, Authenticator App, FIDO2, Number Matching |
| [[Conditional Access]] | Richtlinien, Named Locations, Sign-in Risk, Device Compliance, Filters |
| [[Privileged Identity Management]] | Just-in-Time-Zugriff, Rollen aktivieren, Access Reviews, Audit |
| [[Identity Protection]] | Risk-basierte Richtlinien, Sign-in Risk, User Risk, Remediation |
| [[Single Sign-On]] | SAML, OIDC, OAuth 2.0, App-Registrierungen, Enterprise Apps |
| [[Hybrid Identity Entra Connect]] | AD Connect, Password Hash Sync, Pass-through Auth, Federation (ADFS) |
| [[Entra Connect Cloud Sync]] | Leichtgewichtige Alternative zu Entra Connect, Konfiguration |
| [[Entra ID Join vs Hybrid Join]] | Gerätebindung, SSO-Szenarien, Registrierung, Primary Refresh Token |
| [[Entra Permissions Management]] | Cloud Infrastructure Entitlement Management (CIEM) |
| [[Microsoft Entra Verified ID]] | Dezentrale Identitäten, Verifiable Credentials |
| [[Entra Private Access und Internet Access]] | ZTNA, VPN-Ersatz, Secure Web Gateway (SWG), CASB, Global Secure Access |
| [[External Identities B2B]] | Gastbenutzer, B2B-Zusammenarbeit, Cross-Tenant-Sync |
| [[Entra ID B2C]] | Kundenidentitäten, Custom Policies, Social Logins |

---

## 📧 Exchange Online

| Thema | Beschreibung |
|-------|-------------|
| [[Exchange Online Grundlagen]] | Architektur, Postfachtypen, Protokolle (MAPI, EWS, REST, ActiveSync) |
| [[Postfächer verwalten]] | User Mailbox, Shared Mailbox, Resource Mailbox, Archive Mailbox |
| [[Verteilergruppen und M365-Gruppen]] | Gruppentypen, Governance, dynamische Verteiler |
| [[Mail Flow und Transport Rules]] | Konnektoren, Regeln, SMTP-Relay, Message Tracing |
| [[E-Mail-Authentifizierung SPF DKIM DMARC]] | SPF, DKIM, DMARC — Konfiguration und Verifizierung |
| [[Anti-Spam und Anti-Phishing]] | Defender for Office 365, Safe Links, Safe Attachments, Quarantäne |
| [[E-Mail-Migration]] | Cutover, Staged, Hybrid, IMAP, PST-Import |
| [[Exchange Hybrid]] | Hybridkonfiguration, Free/Busy, OAuth, HCW |
| [[Öffentliche Ordner]] | Migration, Verwaltung, Berechtigungen |
| [[Postfach-Berechtigungen]] | Full Access, Send As, Send on Behalf, Delegation |
| [[Exchange Online PowerShell]] | ExchangeOnlineManagement, häufige Befehle, Automatisierung |

---

## 🗂️ SharePoint Online & OneDrive

| Thema | Beschreibung |
|-------|-------------|
| [[SharePoint Online Grundlagen]] | Architektur, Websitetypen, Hub Sites, Inhaltstypen, Term Store |
| [[Websitesammlungen verwalten]] | Erstellen, Vorlagen, Speicherkontingente, Einstellungen |
| [[Berechtigungen und Sharing]] | Vererbung, externe Freigabe, Sharing Policies, Access Reviews |
| [[SharePoint Seiten und Webparts]] | Modern Pages, Webparts, News, Communication Sites |
| [[OneDrive for Business]] | Synchronisation, Known Folder Move, Speicher, Retention |
| [[SharePoint Governance]] | Policies, Lifecycle, Klassifizierung, Aufbewahrung, Expiration |
| [[SharePoint Suche]] | Search Schema, Inhaltsquellen, Suchergebnis-Anpassung |
| [[SharePoint Syntex und Copilot]] | Content AI, Dokumentenverständnis, Metadaten-Extraktion |
| [[SharePoint Premium]] | Advanced Management, Site Lifecycle, Backup |
| [[SharePoint PowerShell und PnP]] | PnP PowerShell, PnP Templates, CLI for Microsoft 365 |

---

## 💬 Microsoft Teams

| Thema | Beschreibung |
|-------|-------------|
| [[Teams Grundlagen]] | Architektur, Teams/Kanäle, Chats, Apps, Tabs |
| [[Teams und Kanäle verwalten]] | Erstellen, Mitglieder, Kanal-Typen (Standard, Privat, Shared) |
| [[Governance und Policies]] | Meeting Policies, Messaging Policies, Teams-Lifecycle, Naming Policies |
| [[Teams-Telefonie]] | Direct Routing, Calling Plans, Operator Connect, Voicemail, Auto Attendant |
| [[Meetings und Live Events]] | Meeting-Optionen, Live Events, Webinare, Town Halls, Q&A |
| [[Teams Rooms]] | Microsoft Teams Rooms (MTR), Hardware, Verwaltung, Pro Management Portal |
| [[Teams Apps und Bots]] | App-Governance, Custom Apps, Power Apps in Teams, Bots |
| [[Teams Phone Mobile]] | SIM-Enablement, Mobile Operator Integration |
| [[Teams Premium]] | Intelligent Recap, Webinar-Features, Erweiterte Meetings |
| [[Teams PowerShell]] | MicrosoftTeams-Modul, Policy-Verwaltung, Bulk-Operationen, Reporting |

---

## 📱 Intune & Endpoint Management

| Thema | Beschreibung |
|-------|-------------|
| [[Intune Grundlagen]] | Architektur, MDM vs. MAM, Enrollment-Typen, Co-Management |
| [[Windows Enrollment]] | Autopilot, Bulk Enrollment, BYOD, Hybrid Join, Co-Management mit SCCM |
| [[iOS und iPadOS Enrollment]] | ADE (DEP), Supervised Mode, BYOD, User/Device Enrollment |
| [[Android Enrollment]] | Android Enterprise (Work Profile, COPE, COBO), Samsung Knox |
| [[macOS Enrollment]] | ADE, Platform SSO, Systemerweiterungen |
| [[Compliance Policies]] | Regeln, Aktionen bei Nicht-Konformität, Conditional Access Integration |
| [[Konfigurationsprofile]] | Templates, Settings Catalog, ADMX-Import, Scripting |
| [[App Management MAM]] | App Protection Policies, Managed Apps, App-Deployment, Store-Apps |
| [[Windows Autopilot]] | Deployment Profiles, OOBE, Self-Deploying, Pre-Provisioning, Reset |
| [[Software-Deployment]] | Win32 Apps, MSI, Scripts, Supersedence, Dependency |
| [[Endpoint Security]] | Defender-Einstellungen, Disk Encryption, Firewall, Attack Surface Reduction |
| [[Endpoint Analytics]] | Health Score, Startup Performance, Proactive Remediation |
| [[Windows Update for Business]] | Update Rings, Feature Updates, Expedite, Driver Updates |
| [[Intune PowerShell und Graph]] | Microsoft Graph für Intune, Automatisierung, Bulk-Operationen |

---

## 🔑 Power Platform

| Thema | Beschreibung |
|-------|-------------|
| [[Power Platform Grundlagen]] | Überblick, Lizenzierung, Environments, DLP-Policies, Center of Excellence |
| [[Power Apps]] | Canvas Apps, Model-driven Apps, Portals, Dataverse-Integration |
| [[Power Automate]] | Cloud Flows, Desktop Flows (RPA), Connectors, AI Builder |
| [[Power BI]] | Dashboards, Reports, Datasets, Gateways, Embedding, Microsoft Fabric |
| [[Copilot Studio]] | Chatbots, Topics, Generative AI, Power Automate Integration, Kanäle |
| [[Microsoft Dataverse]] | Tabellen, Beziehungen, Sicherheitsrollen, Lösungen, Dataverse for Teams |
| [[Power Pages]] | Low-Code Webseiten, Authentication, Formulare, API-Integration |
| [[Power Platform Governance]] | CoE Starter Kit, ALM, Umgebungsstrategien, Monitoring, DLP |

---

## 🛡️ Sicherheit & Compliance (Microsoft 365)

| Thema | Beschreibung |
|-------|-------------|
| [[Microsoft Defender for Office 365]] | Plan 1/2, Safe Links, Safe Attachments, Anti-Phishing, AIR |
| [[Microsoft Defender for Endpoint]] | Onboarding, EDR, Attack Surface Reduction, Vulnerability Management |
| [[Microsoft Defender for Identity]] | On-Premises AD-Schutz, Lateral Movement Detection, Alerts |
| [[Microsoft Defender for Cloud Apps]] | CASB, Shadow IT, App-Governance, Session Controls, App Connector |
| [[Microsoft Defender XDR]] | Unified Security Operations, Incidents, Hunting, Automated IR |
| [[Microsoft Sentinel]] | Workspaces, Analytics Rules, Playbooks, Hunting, Connectors |
| [[Microsoft Purview Grundlagen]] | Compliance Center, Datenschutz, Klassifizierung, Governance |
| [[Sensitivity Labels]] | Klassifizierung, Verschlüsselung, Markierungen, Auto-Labeling |
| [[Data Loss Prevention DLP]] | Richtlinien, Bedingungen, Aktionen, Endpunkt-DLP, Teams-DLP |
| [[Retention Policies und Labels]] | Aufbewahrungsfristen, Disposition Review, Regulatory Records |
| [[eDiscovery und Audit]] | Core/Premium eDiscovery, Audit Log, Content Search, Legal Hold |
| [[Information Barriers]] | Kommunikationsbeschränkungen, Segmentierung |
| [[Communication Compliance]] | Richtlinienüberwachung, Reviewer-Workflows, ML-Klassifizierung |
| [[Insider Risk Management]] | Risikorichtlinien, Indikatoren, Alerts, Cases |
| [[Microsoft Priva]] | Privacy Risk Management, Subject Rights Requests |
| [[Microsoft Secure Score]] | Bewertung, Empfehlungen, Vergleich, Verbesserungsmaßnahmen |

---

## 🖥️ Windows 365 & Azure Virtual Desktop

| Thema | Beschreibung |
|-------|-------------|
| [[Windows 365 Cloud PC]] | Grundlagen, Pläne, Provisioning, Management über Intune |
| [[Windows 365 Boot und Frontline]] | Shared-Modus, Offline-Fähigkeit, Frontline-Worker |
| [[Azure Virtual Desktop AVD]] | Host Pools, Session Hosts, App Groups, Workspaces |
| [[AVD Netzwerk und Identität]] | Verbindungsarchitektur, Entra ID Join, FSLogix Profile Container |
| [[FSLogix]] | Profile Container, Office Container, Cloud Cache, Konfiguration |
| [[RDS vs AVD vs W365 Vergleich]] | Vergleich der Desktop-Virtualisierungs-Optionen |

---

## ☁️ Azure Grundlagen

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Überblick]] | Regionen, Availability Zones, Dienste-Kategorien, SLAs |
| [[Subscriptions und Management Groups]] | Hierarchie, Billing, EA/MCA, Cost Management |
| [[Resource Groups]] | Konzept, Tagging, Locks, Deployment-Scopes |
| [[Azure RBAC]] | Built-in Roles, Custom Roles, Scope-Hierarchie, PIM für Azure |
| [[Azure Policy]] | Definitionen, Initiatives, Assignments, Compliance, Remediation |
| [[Azure Landing Zones]] | Cloud Adoption Framework, Enterprise-Scale, Hub-and-Spoke |
| [[Azure Cost Management]] | Budgets, Alerts, Cost Analysis, Advisor, Reservations, Savings Plans |
| [[Azure Advisor]] | Empfehlungen für Kosten, Sicherheit, Performance, Reliability |
| [[Azure Well-Architected Framework]] | 5 Säulen (Cost, Reliability, Security, Performance, Operations), Assessments, Review Tool |

---

## 🔐 Azure Identität & Schlüsselverwaltung

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Managed Identities]] | System-assigned vs. User-assigned, Anwendungsfälle, Best Practices |
| [[Azure Key Vault]] | Secrets, Keys, Certificates, Access Policies, RBAC, Private Endpoint |
| [[Azure AD Domain Services]] | Managed Domain Services, LDAP, Kerberos, Migration von on-prem |
| [[Microsoft Entra Workload ID]] | Service Principals, App Registrations, Federated Credentials |

---

## 🌐 Azure Netzwerk

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Virtual Network]] | Adressräume, Subnets, Peering, Service Endpoints, Private Endpoints |
| [[Network Security Groups]] | Regeln, Application Security Groups, Flow Logs, Diagnose |
| [[Azure Firewall]] | SKUs, DNAT/SNAT, Policy, Threat Intelligence, Premium-Features |
| [[Azure Load Balancer]] | SKUs, Frontend/Backend, Health Probes, Outbound Rules |
| [[Azure Application Gateway]] | WAF, URL-Routing, SSL-Offloading, Autoscaling |
| [[Azure Front Door]] | Global Load Balancing, CDN, WAF, Private Link Origin |
| [[Azure Traffic Manager]] | DNS-basiertes Load Balancing, Routing-Methoden |
| [[Azure VPN Gateway]] | Site-to-Site, Point-to-Site, VNet-to-VNet, BGP |
| [[Azure ExpressRoute]] | Private Peering, Microsoft Peering, Circuit-SKUs, FastPath |
| [[Azure Virtual WAN]] | Hub-and-Spoke in der Cloud, Routing-Policies, Secured Hub |
| [[Azure Bastion]] | Sicherer RDP/SSH-Zugang ohne öffentliche IP, SKUs |
| [[Azure DDoS Protection]] | Basic vs. Standard, Metriken, Adaptive Tuning |
| [[Azure Private Link]] | Private Endpoints, Private Link Service, DNS-Integration |
| [[Azure DNS]] | Public/Private Zones, DNS Private Resolver, Forwarding Rules |
| [[Azure Network Watcher]] | Connection Monitor, NSG Flow Logs, Packet Capture, Topology |

---

## 💻 Azure Compute

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Virtual Machines]] | VM-Typen, Größen, Images, Availability Sets/Zones, Scale Sets |
| [[Azure VM Erweiterungen]] | Custom Script, Azure Monitor Agent, Disk Encryption, Domain Join |
| [[Azure Spot VMs]] | Preemptible Workloads, Eviction Policies, Cost Savings |
| [[Azure App Service]] | Web Apps, API Apps, Plan-Typen, Deployment Slots, Autoscaling |
| [[Azure Functions]] | Serverless, Trigger-Typen, Durable Functions, Hosting-Pläne |
| [[Azure Container Instances]] | Schnelle Container-Deployments, Container Groups, Sidecars |
| [[Azure Kubernetes Service]] | Cluster-Management, Node Pools, Networking, Upgrades, KEDA |
| [[Azure Container Apps]] | Serverless Kubernetes, KEDA, Dapr, Revision Management |
| [[Azure Container Registry]] | Image-Storage, Geo-Replication, Tasks, Webhooks, Vulnerability Scan |
| [[Azure Batch]] | HPC-Workloads, Jobs, Pools, Task-Scheduling, Low Priority |

---

## 🗄️ Azure Storage

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Storage Grundlagen]] | Account-Typen, Redundanz (LRS/ZRS/GRS/GZRS), Access Tiers |
| [[Azure Blob Storage]] | Container, Blobs, Lifecycle Management, Versioning, Soft Delete |
| [[Azure Files]] | SMB/NFS Shares, Azure File Sync, AD-Authentifizierung |
| [[Azure Queue Storage]] | Message Queuing, Grundlagen, Anwendungsfälle |
| [[Azure Data Lake Storage Gen2]] | Hierarchical Namespace, Analytics-Workloads, ACLs |
| [[Azure Managed Disks]] | HDD/SSD/Ultra, Snapshots, Disk Encryption, Shared Disks |
| [[Azure NetApp Files]] | Enterprise NFS/SMB, Performance-Tiers, Volumes, Cross-Region |

---

## 🗃️ Azure Datenbanken

| Thema | Beschreibung |
|-------|-------------|
| [[Azure SQL Database]] | DTU vs. vCore, Single/Elastic Pool, Hyperscale, Business Critical |
| [[Azure SQL Managed Instance]] | Near 100% SQL Compatibility, VNet-Integration, Migration |
| [[SQL Server auf Azure VMs]] | IaaS, Availability Groups, FCI, SQL IaaS Extension |
| [[Azure Database for PostgreSQL]] | Flexible Server, Hochverfügbarkeit, Extensions, Citus |
| [[Azure Database for MySQL]] | Flexible Server, Replikation, Backup, Zone Redundancy |
| [[Azure Cosmos DB]] | Multi-Model (SQL/MongoDB/Cassandra/Gremlin/Table), RUs, Verteilung |
| [[Azure Cache for Redis]] | Caching-Szenarien, Tiers, Persistenz, Clustering, Geo-Replication |
| [[Azure Synapse Analytics]] | Data Warehouse, Spark, Serverless SQL, Pipelines, Link |
| [[Azure Data Factory]] | ETL/ELT, Pipelines, Integration Runtimes, Monitoring |

---

## 📊 Microsoft Fabric

| Thema | Beschreibung |
|-------|-------------|
| [[Microsoft Fabric Grundlagen]] | Unified Analytics Platform, Kapazitäten, Workspaces, Lizenzierung, F-SKUs |
| [[OneLake]] | Zentraler Data Lake für Fabric, Shortcuts, OneCopy-Prinzip, ADLS-Kompatibilität |
| [[Fabric Lakehouse]] | Delta-Tabellen, SQL Analytics Endpoint, Notebooks, Medaillon-Architektur |
| [[Fabric Data Warehouse]] | T-SQL Warehouse, Cross-Database Queries, Mirroring, Copilot-Integration |
| [[Fabric Data Factory]] | Pipelines, Dataflows Gen2, Copy Activity, Monitoring |
| [[Fabric Real-Time Intelligence]] | Eventstreams, KQL-Datenbank, Real-Time Dashboards, Activator (Reflexe) |
| [[Power BI in Fabric]] | Semantic Models, Direct Lake Mode, Deployment Pipelines, Datamarts |
| [[Fabric Notebooks und Spark]] | PySpark, Scala, SparkR, Lakehouse-Anbindung, Umgebungen |
| [[Microsoft Purview in Fabric]] | Data Governance, Sensitivity Labels, Data Lineage, Information Protection |
| [[Fabric Copilot]] | KI-gestützte Datenanalyse, DAX-Generierung, Report-Erstellung, Notebook-Hilfe |
| [[Fabric Administration]] | Kapazitätsverwaltung, Tenant-Einstellungen, Monitoring Hub, Alerting |

---

## 🔗 Azure Integration & Messaging

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Logic Apps]] | Low-Code Workflows, Connectors, Standard vs. Consumption |
| [[Azure API Management]] | API Gateway, Policies, Developer Portal, Versioning, APIM-Tiers |
| [[Azure Service Bus]] | Queues, Topics, Subscriptions, Dead Letter, Sessions, Geo-DR |
| [[Azure Event Grid]] | Ereignisbasierte Architektur, Topics, Event Subscriptions, CloudEvents |
| [[Azure Event Hubs]] | Big Data Streaming, Capture, Kafka-Kompatibilität, Dedicated |
| [[Azure Web PubSub]] | Real-Time Web-Anwendungen, WebSocket, SignalR Service |
| [[Azure Communication Services]] | CPaaS — Voice, Video, Chat, SMS, E-Mail als APIs; Teams-Interop, Calling SDK |

---

## 🛡️ Azure Sicherheit

| Thema | Beschreibung |
|-------|-------------|
| [[Microsoft Defender for Cloud]] | CSPM, Workload Protection, Secure Score, Regulatory Compliance |
| [[Microsoft Defender for Containers]] | Image Scanning, Runtime Protection, AKS, Registry-Scans |
| [[Azure Firewall Manager]] | Zentrale Richtlinien, Virtual WAN-Integration, DDoS |
| [[Azure Web Application Firewall]] | OWASP-Regelsets, Custom Rules, Bot Protection, Exclusions |
| [[Azure Security Benchmark]] | Best-Practice-Richtlinien, CIS, NIST-Mapping |

---

## 📊 Azure Monitor & Management

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Monitor Grundlagen]] | Metriken, Logs, Alerts, Dashboards, Workbooks |
| [[Log Analytics Workspace]] | Tabellen, KQL-Abfragen, Datenaufbewahrung, Workspace-Design |
| [[Azure Monitor Agent]] | DCR-basierte Konfiguration, Ablösung von MMA/OMS |
| [[Application Insights]] | APM, Dependency Tracking, Live Metrics, Smart Detection |
| [[Azure Alerts]] | Metric Alerts, Log Alerts, Activity Log Alerts, Action Groups |
| [[Azure Automation]] | Runbooks, State Configuration (DSC), Hybrid Worker |
| [[Azure Update Manager]] | Patch Management für Azure und Arc-fähige Server |
| [[Azure Backup]] | Recovery Services Vault, Policies, MARS Agent, MABS |
| [[Azure Site Recovery]] | Disaster Recovery, Replikation, Failover/Failback, RPO/RTO |
| [[Azure Migrate]] | Assessment, Server Migration, Database Migration Service |
| [[Azure Resource Graph]] | Ressourcen abfragen, KQL, Compliance-Übersicht, Änderungshistorie |

---

## 🌉 Azure Arc

| Thema | Beschreibung |
|-------|-------------|
| [[Azure Arc Grundlagen]] | Konzept, Architektur, unterstützte Ressourcentypen, Lizenzierung |
| [[Arc-fähige Server]] | Onboarding Windows/Linux, Connected Machine Agent, Extensions |
| [[Arc-fähige Kubernetes]] | Cluster-Verbindung, GitOps (Flux), Cluster Extensions, Azure Policy |
| [[Arc-fähige SQL Server]] | Management über Arc, Best Practices Assessment, Lizenzierung |
| [[Arc-enabled Data Services]] | Azure SQL MI auf Arc, PostgreSQL Hyperscale, Controller |
| [[Azure Stack HCI]] | Hyper-V Cluster mit Azure-Integration, Arc-basiertes Management |
| [[Azure Stack Edge]] | Edge-Computing, IoT, lokale Azure-Dienste, GPU-SKUs |

---

## ⚙️ Administration & Automatisierung

| Thema | Beschreibung |
|-------|-------------|
| [[Microsoft Graph API]] | REST API, Berechtigungen, Delegated vs. Application, SDKs, Throttling |
| [[PowerShell für Microsoft 365]] | Graph PowerShell, ExchangeOnline, Teams, SharePoint PnP |
| [[Azure PowerShell]] | Az-Modul, Authentifizierung, häufige Befehle, Scripting-Patterns |
| [[Azure CLI]] | Installation, Befehle, Skripte, JSON-Ausgabe, Best Practices |
| [[Azure Bicep]] | Infrastructure as Code, Module, Deployment, Parameter Files |
| [[Terraform für Azure]] | AzureRM Provider, State Management, Module, Workspaces |
| [[GitHub Actions für Azure]] | CI/CD Pipelines, OIDC Auth, Deployment Workflows |
| [[Azure DevOps]] | Pipelines, Repos, Boards, Artifacts, Service Connections |
| [[Microsoft 365 DSC]] | Desired State Configuration für M365-Tenant-Konfiguration |
| [[Microsoft 365 Reporting]] | Admin Center Reports, Graph Reports, Power BI Integration |
| [[Service Health und Monitoring]] | Admin Center, Service Health Dashboard, Incidents, Post-Mortems |
| [[Backup-Strategien für M365]] | Veeam, Drittanbieter, native Retention vs. echtes Backup |
| [[Lizenzmanagement]] | Gruppen-basierte Lizenzierung, Kostenoptimierung, Automatisierung |
| [[Azure Lighthouse]] | Delegiertes Ressourcen-Management für MSPs, Multi-Tenant Azure-Verwaltung, Marketplace-Angebote |

---

## 🤖 Azure KI & Machine Learning

| Thema | Beschreibung |
|-------|-------------|
| [[Azure OpenAI Service]] | GPT-4/o, DALL-E, Whisper, Embeddings, Deployments, Quotas, Private Endpoints |
| [[Azure AI Foundry]] | Unified AI-Entwicklungsplattform, Prompt Flow, Model Catalog, Evaluation |
| [[Azure AI Services]] | Vision (OCR, Face), Speech (STT/TTS), Language (NLP, Übersetzer), Anomaly Detector |
| [[Azure AI Document Intelligence]] | Formularerkennung, Custom Models, Prebuilt Models (Rechnungen, IDs, Verträge) |
| [[Azure AI Search]] | Volltext + Vektorsuche, Skillsets, Indexer, Semantic Ranking, RAG-Integration |
| [[Azure Machine Learning]] | Workspaces, Compute, MLOps, Automated ML, Designer, Pipeline, Deployment |
| [[Azure Bot Service]] | Chatbot-Plattform, Direktkanal, Teams-Integration, Composer |
| [[Microsoft Security Copilot]] | KI-gestützter Security-Assistent, Skills, Promptbooks, MSSP-Szenarien |
| [[Azure KI Zusammenfassung]] | Überblick aller KI-Dienste, Auswahl-Guide (welcher Dienst wann?) |

---

## 🌐 Azure IoT

| Thema | Beschreibung |
|-------|-------------|
| [[Azure IoT Hub]] | Geräteverwaltung, D2C/C2D Messaging, Device Twins, Routes, DPS |
| [[Azure IoT Central]] | Managed IoT-Plattform, Device Templates, Dashboards, Regeln, Export |
| [[Azure Digital Twins]] | Digitale Modellierung, DTDL, Graphen, Event Routes, Integration |
| [[Azure Sphere]] | Sicheres Microcontroller-Ökosystem, Guardian Module, AS3-Updates |
| [[Azure IoT Edge]] | Container auf Edge-Geräten, Module, Offline-Fähigkeit, Azure Functions |

---

## 🐙 GitHub (Microsoft)

| Thema | Beschreibung |
|-------|-------------|
| [[GitHub Grundlagen]] | Repositories, Branches, Pull Requests, Issues, Projects, Wikis |
| [[GitHub Actions]] | Workflows, Jobs, Steps, Runner-Typen (hosted/self-hosted), Secrets, Environments |
| [[GitHub Copilot]] | Copilot für Entwickler, Chat, Workspace, CLI, Extensions, Enterprise-Governance |
| [[GitHub Advanced Security]] | Code Scanning (CodeQL), Secret Scanning, Dependency Review, Security Overview |
| [[GitHub Packages]] | Container Registry, npm/Maven/NuGet/PyPI Hosting, Authentifizierung |
| [[GitHub Enterprise]] | Enterprise Server vs. Cloud, SAML/SCIM, Policies, Audit Log, Migration |
| [[GitHub Codespaces]] | Cloud-Dev-Umgebungen, Dev Container, Prebuilds, Limits |
| [[GitHub Pages]] | Statisches Hosting, Jekyll, Custom Domains, Actions-Deployment |
| [[GitHub vs Azure DevOps Vergleich]] | Pipelines vs. Actions, Repos vs. GitHub, wann welche Plattform |

---

## 📦 Dynamics 365

| Thema | Beschreibung |
|-------|-------------|
| Dynamics 365 Überblick | Architektur, Dataverse, Lizenzierung, Power Platform Integration |
| Dynamics 365 Sales | Lead-Management, Opportunities, Forecast, Copilot-Integration |
| Dynamics 365 Customer Service | Cases, Queues, SLA, Omnichannel, Knowledge Management |
| Dynamics 365 Field Service | Work Orders, Scheduling, IoT-Integration, Copilot-Dispatcher |
| Dynamics 365 Finance | Hauptbuch, Debitoren/Kreditoren, Budgetierung, Reporting |
| Dynamics 365 Supply Chain Management | Lagerverwaltung, Produktion, Beschaffung, Prognose |
| Dynamics 365 Business Central | SMB ERP, Finanzen, Einkauf, Lager, Produktion, Projekte |
| Dynamics 365 Customer Insights | Customer Data Platform (CDP), Journeys (ehem. Marketing), Segmentierung |
| Dynamics 365 HR | Mitarbeiterverwaltung, Urlaub, Benefits, Lernpfade |
| Dynamics 365 Administration | Umgebungen, Lösungen, ALM, Monitoring, Dataverse-Sicherheit |

---

## 🏢 System Center (On-Premises Management)

| Thema | Beschreibung |
|-------|-------------|
| MECM / SCCM Grundlagen | Configuration Manager, Site-Hierarchie, Client-Deployment, Discovery |
| MECM Software Deployment | Pakete, Anwendungen, Task Sequences, OS Deployment (OSD), PXE |
| MECM Patch Management | SUP, Software Update Groups, Wartungsfenster, Compliance |
| MECM Co-Management | Workload-Verschiebung zu Intune, Piloting, Anforderungen |
| System Center Operations Manager (SCOM) | Monitoring, Management Packs, Alerting, Dashboards, AMA-Integration |
| System Center Virtual Machine Manager (SCVMM) | Hyper-V Fabric-Management, Templates, SDN, VMM-Bibliothek |
| System Center Data Protection Manager (DPM) | Backup für Hyper-V, SQL, Exchange, SharePoint, Azure-Integration |
| System Center Orchestrator | Runbooks, Integration Packs, Workflows, Automatisierung |
| System Center → Azure-Migration | Ablösung durch Azure Arc, Azure Monitor, Intune, Update Manager |

---

## 🔧 Intune Suite, Autopatch & Lighthouse

| Thema | Beschreibung |
|-------|-------------|
| Endpoint Privilege Management (EPM) | Least-Privilege für Windows, Elevation Rules, Approval Workflows |
| Microsoft Cloud PKI | Managed PKI in Intune, Root/Issuing CA, SCEP/PKCS, Zertifikat-Deployment |
| Microsoft Tunnel | VPN-Gateway für iOS/Android, Split Tunneling, Conditional Access, MAM |
| Remote Help | Remotesupport über Intune, Unattended Access, Rollen, Audit |
| Advanced Endpoint Analytics | Anomaly Detection, Enhanced Reporting, Custom Compliance |
| Microsoft Autopatch | Automatisiertes Patch Management (Windows/M365/Edge/Teams), Ringe, Reporting |
| Microsoft 365 Lighthouse | MSP Multi-Tenant-Verwaltung, Tenants, Baselines, Alerts, Delegated Access |
| Microsoft Dev Box | Cloud-Entwicklerworkstations, Dev Center, Projekte, Pools, Netzwerk |
| Microsoft Intune Suite Überblick | Alle Add-ons im Überblick, Lizenzierung, wann welcher Plan sinnvoll ist |

---

## 🗺️ Lernpfade

### Microsoft 365 Einsteiger
```
Was ist Microsoft 365?
  └─► Entra ID Grundlagen           ← Identität verstehen
       └─► MFA + Conditional Access  ← Zugriff absichern
            ├─► Exchange Online       ← E-Mail
            ├─► Teams                ← Kollaboration
            └─► SharePoint/OneDrive  ← Dateiablage
```

### Azure Einsteiger
```
Azure Überblick
  └─► Subscriptions & Resource Groups
       └─► Azure RBAC + Policy       ← Governance
            ├─► Azure VNet           ← Netzwerk
            ├─► Azure VMs            ← Compute
            └─► Azure Storage        ← Speicher
```

### Hybrid & Arc
```
Hybrid Identity (Entra Connect)
  └─► Arc-fähige Server              ← On-Premises verwalten
       └─► Azure Stack HCI            ← Hyper-V + Azure
            └─► Arc-fähige Kubernetes ← Container On-Premises
```

### GitHub & DevOps
```
GitHub Grundlagen
  └─► GitHub Actions                   ← CI/CD
       ├─► GitHub Advanced Security    ← DevSecOps
       ├─► GitHub Copilot              ← KI-gestützte Entwicklung
       └─► Azure DevOps                ← Alternativ/Hybrid
```

### Azure KI Einstieg
```
Azure OpenAI Service
  └─► Azure AI Foundry                 ← Entwicklungsplattform
       ├─► Azure AI Search             ← RAG-Architektur
       ├─► Azure Machine Learning      ← MLOps
       └─► Microsoft Security Copilot ← KI für Security
```

### Security & Compliance
```
Microsoft Secure Score
  ├─► Defender XDR                   ← Unified SecOps
  ├─► Microsoft Sentinel             ← SIEM/SOAR
  ├─► Microsoft Purview              ← Compliance & Data
  └─► Defender for Cloud             ← Azure-Sicherheit
```

---

## Verwandte Notizen

- [[Hyper-V Index]] — Lokale Virtualisierung als Basis für Hybrid-Szenarien
- [[Windows Server Index]] — Grundwissen rundum Windows Server
