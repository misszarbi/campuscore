# CampusCore — Réseau mobile privé 5G SA open source avec network slicing

Projet 5 (ESMT Dakar, cycle ingénieur Télécommunications & Services) — cœur réseau 5G Standalone open source (Open5GS), RAN simulée (UERANSIM), network slicing avec QoS différenciée, portail d'administration, supervision et sécurisation du service.

**Rapport d'ingénierie complet** : voir `CampusCore_Rapport.docx` (contexte, conception, réalisation détaillée, mesures, limites).

---

## 1. Architecture

Le système repose sur deux machines virtuelles distinctes, séparant le cœur réseau de l'accès radio, reliées entre elles via les interfaces normalisées N2 et N3.

Architecture : deux VM distinctes reliées en réseau (N2/N3)

VM "ueransim" (RAN simulée)
  - gNB (UERANSIM)
  - gNB2 (handover)
  - UE (2 sessions PDU : eMBB + critique)

VM "coeur" (cœur 5G + services)
  - Cœur 5G SA (Open5GS) : AMF, SMF, SMF2, UPF, UPF2, NRF, AUSF, UDM, UDR, PCF, BSF
  - Administration : Portail Flask + API REST, WebUI Open5GS, Nginx (HTTPS)
  - Supervision : Prometheus + Grafana
  - Base de données : MongoDB

Deux tranches réseau (network slicing) :
- **eMBB** (S-NSSAI SST=1, SD=000001) — débit permissif (AMBR 100 Mbit/s), non prioritaire
- **Critique** (S-NSSAI SST=2, SD=000002) — débit plus restreint (AMBR 20 Mbit/s) mais prioritaire (5QI 1, ARP 1)

Détail complet de l'architecture, des choix techniques et de leurs justifications : voir section 2 du rapport.

---

## 2. Structure du dépôt

campuscore/
- core/
  - open5gs/          (docker-compose.yml du cœur, configs des NF)
  - portal/            (Portail Flask : routes, templates, API, Dockerfile)
  - monitoring/        (Config Prometheus, secrets Grafana .env)
  - nginx/              (Config reverse proxy + certificats)
  - upf-custom/        (Dockerfile UPF personnalisé, iperf3 intégré)
- ran/
  - ueransim/           (docker-compose.yml RAN, configs gNB/gNB2/UE)
  - setup-network.sh   (Script réseau : IP secondaire du second gNB)
- README.md

---

## 3. Prérequis

- Deux machines (physiques ou virtuelles) sous Ubuntu 22.04+, reliées sur le même réseau
- Docker Engine + plugin Docker Compose installés sur les deux machines
- Accès sudo sur les deux machines (configuration réseau)
- Ports ouverts entre les deux machines : 38412/sctp (N2), 2152/udp (N3)

---

## 4. Procédure de déploiement

### 4.1 Sur la VM "coeur"

git clone <url-du-depot> campuscore
cd campuscore/core/open5gs

cp .env.example .env
cp ../portal/.env.example ../portal/.env
cp ../monitoring/.env.example ../monitoring/.env

docker compose up -d
docker compose ps

### 4.2 Sur la VM "ueransim"

git clone <url-du-depot> campuscore
cd campuscore/ran

./setup-network.sh

cd ueransim
docker compose up -d
docker compose logs -f ue1

### 4.3 Configuration des secrets (.env, jamais versionnés)

Fichier core/portal/.env : MONGO_URI, FLASK_SECRET_KEY, ADMIN_USERNAME, ADMIN_PASSWORD, API_KEY
Fichier core/monitoring/.env : GF_SECURITY_ADMIN_PASSWORD

### 4.4 Accès aux services

- Portail d'administration : https://<IP-coeur>
- Documentation API (Swagger) : https://<IP-coeur>/docs
- Supervision Grafana : http://<IP-coeur>:3001
- WebUI Open5GS : http://<IP-coeur>:3000

---

## 5. Limites connues

- Déploiement en VM locale plutôt que VPS/serveur de laboratoire (budget AWS épuisé en cours de projet) : pas de certificat Let's Encrypt, service non accessible depuis l'extérieur du réseau local
- Compteurs de débit UPF désactivés dans le code source d'Open5GS (issue GitHub #3100) : contournés par une métrique de remplacement (sessions actives)
- UERANSIM n'implémente pas le handover N2/Xn natif : démontré via réétablissement de session (mécanisme de repli 3GPP réel)
- Adresse IP secondaire du script setup-network.sh non persistante entre redémarrages de VM (à relancer si besoin)

Détail complet, chiffres et diagnostics : voir le rapport d'ingénierie, sections 3 et 5.

---

## 6. Auteure

Marieme Dieye — ESMT Dakar, cycle ingénieur Télécommunications & Services — encadrement : Professeur NIANG BOUDAL.
