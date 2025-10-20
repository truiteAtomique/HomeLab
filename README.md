# SOC Homelab — Plan de segmentation, hiérarchie, outils et réseaux

## But : Déployer un SOC homelab entièrement virtualisé sous Proxmox (installé en nested VirtualBox) avec zones segmentées, collecte de logs et scénarios de tests.

### 1. Vue d'ensemble de l'architecture

L'environnement est organisé en zones distinctes, chacune isolée par son propre réseau virtuel (bridge). Les flux entre zones sont strictement contrôlés par FortiGate (périmètre) et pfSense (interne). Le SOC centralise les logs et orchestre la réponse.

Zones principales (avec subnets proposés) :

WAN (Internet simulé) — vmbr0 (NAT VirtualBox)

SOC — vmbr4 — 192.168.11.0/24

DMZ — vmbr1 — 192.168.14.0/24

Intermédiaire / pfSense — vmbr2 — 192.168.20.0/24

Active Directory (LAN interne) — vmbr3 — 192.168.30.0/24

Remarque : ces subnets correspondent à la topologie que tu as indiquée (SOC 192.168.11.0/24, DMZ 192.168.14.0/24, etc.).

### 2. Liste des machines et rôle (suggestion de priorisation)

FortiGate (NGFW) — VM périmètre

Rôle : filtrage « deny all, allow by exception », NAT, inspection couche 7

pfSense + Snort (IDS/IPS) — VM intermédiaire

Rôle : inspection interne, VPN admin, intégration SOAR

Wazuh (SIEM) — VM SOC (peut être co-hébergé avec ELK au départ)

Elastic Stack (ELK) — VM SOC (ingestion / analytics / ML)

Splunk — VM SOC (optionnel/alternatif à ELK) — peut être allégé

Shuffle (SOAR) — VM SOC (automatisation réponses)

DVWA (app vulnérable) — VM LXC ou container dans DMZ

ModSecurity (WAF) — devant DVWA (reverse proxy) dans DMZ

Cowrie (honeypot SSH) — VM LXC dans DMZ

Active Directory (Windows Server) — VM dans zone AD

Poste client Windows (1+) — VM(s) dans AD pour scénario poste compromis

Serveur Filebeat / Wazuh agents — petits conteneurs/agents répartis

### 3. Assignation des bridges & interfaces (Proxmox)

vmbr0 — WAN (Network naté par VirtualBox) — connecte l'interface WAN de FortiGate

vmbr1 — DMZ — interfaces DMZ de FortiGate, DVWA, ModSecurity, Cowrie, route vers pfSense si nécessaire

vmbr2 — Intermédiaire / pfSense — interfaces LAN<->INTERNE de pfSense, connexion SOC pour alertes

vmbr3 — AD / Internal LAN — contrôleur de domaine, postes clients

vmbr4 — SOC — Wazuh, ELK, Splunk, Shuffle

Exemple d’affectation d’interfaces

FortiGate: eth0 -> vmbr0 (WAN), eth1 -> vmbr1 (DMZ)

pfSense: eth0 -> vmbr1 (DMZ), eth1 -> vmbr2 (Intermédiaire), eth2 -> vmbr3 (LAN) optionnel

Wazuh: eth0 -> vmbr4 (SOC)

DVWA: eth0 -> vmbr1 (DMZ)

Cowrie: eth0 -> vmbr1 (DMZ)

AD DC: eth0 -> vmbr3 (AD)

Client Windows: eth0 -> vmbr3 (AD)

### 4. Plan d’adressage (exemples statiques)

WAN (NAT VirtualBox): 10.0.2.0/24 (géré par VirtualBox)

SOC: 192.168.11.0/24

Wazuh: 192.168.11.10

ELK: 192.168.11.11

Shuffle: 192.168.11.12

DMZ: 192.168.14.0/24

FortiGate-DMZ if: 192.168.14.1

DVWA: 192.168.14.10

ModSecurity: 192.168.14.11 (reverse proxy devant DVWA)

Cowrie: 192.168.14.20

Intermédiaire / pfSense: 192.168.20.0/24

pfSense: 192.168.20.1

AD / LAN interne: 192.168.30.0/24

AD DC: 192.168.30.10

Client Win1: 192.168.30.21

Indiquer statiquement les IPs principales facilite la documentation et les règles de firewall.

### 5. Règles de flux et sécurité (résumé)
WAN -> DMZ (via FortiGate)

Par défaut: deny all

Autoriser uniquement:

HTTP/HTTPS -> 192.168.14.10 (DVWA) (ports 80/443)

SSH -> 192.168.14.20 (Cowrie honeypot) (port 22)

Bloquer tout le reste

DMZ -> SOC

Autoriser uniquement l’envoi de logs:

Syslog (UDP 514 / TCP si besoin) -> Wazuh/ELK

Filebeat -> ELK

Agents Wazuh -> Wazuh manager (port 1514/1515 selon config)

SOC -> AD

Communication sortante limitée et chiffrée

Autoriser: collecte Winlog (Wazuh agent), WinRM/Winlogbeat sur ports sécurisés

pfSense <-> SOC

Snort alerts -> Wazuh (syslog/API)

SOAR (Shuffle) autorisé à émettre commandes de blocage vers pfSense API

AD -> Internet

Aucune communication directe.

Tous les flux sortants passent par pfSense puis FortiGate.

### 6. Journalisation & flux de logs (pipeline suggéré)

Agents Wazuh / Filebeat sur DVWA, ModSecurity, Cowrie, pfSense, Windows -> Wazuh manager (SOC)

Wazuh -> indexation dans Elastic (ELK)

Splunk (si utilisé) reçoit ses événements via forwarder (optionnel)

Shuffle surveille Wazuh/ELK pour alertes et déclenche actions (API pfSense, blocklist FortiGate)

### 7. Scénarios de test (checklist)

Brute force SSH (Cowrie)

Lancer une attaque depuis WAN -> Cowrie

Vérifier logs Cowrie -> Wazuh

Vérifier déclenchement Shuffle -> blocage IP sur pfSense

Injection SQL (DVWA)

Lancer payload SQL -> DVWA

Vérifier ModSecurity logging

Vérifier corrélation Wazuh -> dashboard Splunk/ELK

Brute force interne AD

Simuler échecs répétés de connexion sur AD (poste client)

Vérifier Sysmon/Winlog -> Wazuh -> détection anomalie ELK

Processus suspect sur poste client

Lancer un binaire de test (EICAR-style benign test) ou script

Vérifier Sysmon event -> Wazuh -> isolation via SOAR (coupure de port ou mise en quarantaine VM)

### 8. Ressources VM recommandées (première itération)

FortiGate: 2 vCPU / 2 GB RAM / 10 GB

pfSense: 2 vCPU / 2-4 GB RAM / 10-20 GB

Wazuh + ELK (initial co-location): 4 vCPU / 12-16 GB RAM / 150-200 GB

Splunk (si utilisé séparément): 4 vCPU / 8-12 GB RAM / 100+ GB

DVWA (LXC): 1 vCPU / 1-2 GB RAM / 8-16 GB

ModSecurity (reverse proxy): 1 vCPU / 1-2 GB RAM / 8-16 GB

Cowrie (LXC): 1 vCPU / 1 GB RAM / 8 GB

AD DC (Windows): 2 vCPU / 4 GB RAM / 40 GB

Win client: 2 vCPU / 4 GB RAM / 40 GB

Tu peux compresser ces valeurs au début (ex : Wazuh+ELK 8-12GB) et augmenter si besoin.

### 9. Plan de déploiement étape par étape (suggestion)

Préparer bridges vmbr0..vmbr4 dans Proxmox

Déployer FortiGate (ou VM firewall) et configurer interfaces WAN+DMZ

Déployer pfSense, configurer interfaces DMZ/Intermédiaire/LAN

Déployer SOC (Wazuh + ELK co-localisés)

Déployer DMZ services (Cowrie, ModSecurity, DVWA) et agents Wazuh/Filebeat

Déployer Active Directory et un poste client

Tester flux de logs et créer dashboards basiques

Déployer Shuffle et lier à Wazuh + pfSense pour automatisation

Lancer scénarios de test progressifs

### 10. Sauvegardes, snapshots et restauration

Prendre snapshot avant chaque étape critique (ex: configuration AD, installation ELK)

Exporter VM importantes (backup vzdump) régulièrement

Documenter les points de restauration (journaux, dashboards, playbooks SOAR)

### 11. Automatisation et infra-as-code (optionnel)

Ansible: déploiement des agents Wazuh, configuration de ModSecurity, déploiement DVWA

Terraform + Proxmox provider: créer VMs et attacher disques/network automatiquement

Exemple rapide (Ansible role pour installer Wazuh agent) — à implémenter plus tard si souhaité.

### 12. Notes spécifiques à l'environnement nested (VirtualBox)

Active la virtualisation imbriquée (nested-hw-virt) sur la VM Proxmox pour de meilleures performances.

Surveille la latence et l'utilisation CPU/RAM : la virtualisation imbriquée introduit une surcharge.

Teste les règles réseau progressivement — VirtualBox NAT/bridged peut modifier le comportement réel des paquets.

### 13. Checklist rapide avant attaque test




### 14. Annexes utiles (références rapides)

Ports Wazuh: 1514/1515 (selon config), agents -> manager

Syslog: UDP 514 / TCP 514

Filebeat: port configurable (souvent 5044 pour Logstash)
