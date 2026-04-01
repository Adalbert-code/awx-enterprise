# AWX Enterprise sur OKD 4.10 — Guide complet DevSecOps

> **Portfolio DevSecOps** — Adalbert | Cluster OKD sur Hetzner bare-metal  
> Cas d'usage enterprise : Patch OS automatisé + Hardening CIS + Pipeline CI/CD multi-approches

---

## Table des matières

1. [Architecture globale](#1-architecture-globale)
2. [Prérequis](#2-prérequis)
3. [Installation AWX sur OKD](#3-installation-awx-sur-okd)
4. [Configuration initiale AWX](#4-configuration-initiale-awx)
5. [Cas 1 — Workflow Patch OS avec approbation](#5-cas-1--workflow-patch-os-avec-approbation)
6. [Cas 2 — Hardening CIS des workers OKD](#6-cas-2--hardening-cis-des-workers-okd)
7. [Cas 3 — Pipeline CI/CD multi-approches](#7-cas-3--pipeline-cicd-multi-approches)
8. [Résultats et validation](#8-résultats-et-validation)
9. [Dépannage](#9-dépannage)
10. [Références](#10-références)

---

## 1. Architecture globale
![AWX Dashboard](docs/screenshots/01-awx-dashboard.png)
*Interface AWX — Tableau de bord*


```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster OKD 4.10 (Hetzner)               │
│                                                             │
│  ┌─────────────┐    ┌─────────────────────────────────┐    │
│  │   Bastion   │    │         Namespace AWX            │    │
│  │ local-reg.  │    │  ┌──────────┐  ┌─────────────┐  │    │
│  │ :5000       │    │  │ awx-web  │  │  awx-task   │  │    │
│  └─────────────┘    │  └──────────┘  └─────────────┘  │    │
│                     │  ┌──────────┐  ┌─────────────┐  │    │
│  ┌─────────────┐    │  │postgres  │  │    redis    │  │    │
│  │   Masters   │    │  └──────────┘  └─────────────┘  │    │
│  │ x3 (control)│    └─────────────────────────────────┘    │
│  └─────────────┘                                           │
│                     ┌─────────────────────────────────┐    │
│  ┌─────────────┐    │         Workers OKD              │    │
│  │   NFS PVC   │    │  worker-01  worker-02            │    │
│  │ /shares/    │    │  worker-03  worker-04            │    │
│  └─────────────┘    └─────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Stack technique

| Composant | Version | Rôle |
|---|---|---|
| OKD | 4.10.0 | Plateforme Kubernetes enterprise |
| AWX | 23.4.0 | Orchestrateur Ansible (open source Ansible Tower) |
| AWX Operator | 2.7.2 | Déploiement AWX sur Kubernetes |
| PostgreSQL | 13 | Base de données AWX |
| Redis | 7 | Cache et broker de messages |
| NFS | — | Stockage persistant (PVC) |
| RHCOS | Fedora CoreOS | OS immutable des workers OKD |

---

## 2. Prérequis

### Cluster OKD opérationnel

```bash
# Vérifier que tous les nodes sont Ready
oc get nodes
# Attendu : tous Ready, aucun SchedulingDisabled

# Vérifier les MachineConfigPools
oc get mcp
# Attendu : master et worker UPDATED=True, DEGRADED=False

# Vérifier la StorageClass NFS
oc get sc
# Attendu : nfs-client (default)
```

### Registry locale (environnement air-gapped)

Ce cluster est en environnement **disconnected** (pas d'accès direct aux registries publiques).
Toutes les images sont mirrorées sur `local-registry.caas.eazytraining.lab:5000`.

```bash
# Vérifier la registry locale
curl -sk -H "Authorization: Basic $(echo -n 'evreguser2:NewRegistryPass123!' | base64)" \
  https://local-registry.caas.eazytraining.lab:5000/v2/_catalog \
  | python3 -m json.tool
```

### Images requises pour AWX

Mirroer depuis une machine avec accès internet (ex: bastion `ocp`) :

```bash
for image in \
  "quay.io/ansible/awx-operator:2.7.2|ansible/awx-operator:2.7.2" \
  "quay.io/ansible/awx:23.4.0|ansible/awx:23.4.0" \
  "quay.io/ansible/awx-ee:latest|ansible/awx-ee:latest" \
  "quay.io/brancz/kube-rbac-proxy:v0.15.0|kubebuilder/kube-rbac-proxy:v0.15.0" \
  "docker.io/postgres:13|postgres:13" \
  "docker.io/redis:7|redis:7"; do
  src="${image%%|*}"
  dst="${image##*|}"
  skopeo copy \
    --src-tls-verify=false \
    --dest-tls-verify=false \
    --dest-creds="evreguser2:NewRegistryPass123!" \
    docker://$src \
    docker://192.168.110.9:5000/$dst
done
```

---

## 3. Installation AWX sur OKD

### 3.1 Créer le namespace AWX

```bash
oc new-project awx

# Permissions SCC nécessaires sur OpenShift/OKD
oc adm policy add-scc-to-serviceaccount anyuid \
  -z default -n awx

oc adm policy add-scc-to-user anyuid \
  system:serviceaccount:awx:default

oc adm policy add-scc-to-user anyuid \
  system:serviceaccount:awx:awx
```

### 3.2 Télécharger AWX Operator

> Sur OKD en environnement air-gapped, GitHub est inaccessible.
> Télécharger l'archive depuis un poste avec accès internet et transférer via SCP.

```powershell
# Sur Windows — télécharger l'archive
Invoke-WebRequest `
  -Uri "https://github.com/ansible/awx-operator/archive/refs/tags/2.7.2.zip" `
  -OutFile "$env:USERPROFILE\Downloads\awx-operator-2.7.2.zip"

# Transférer vers le bastion
scp "$env:USERPROFILE\Downloads\awx-operator-2.7.2.zip" `
  root@<IP_BASTION>:/root/awx-operator-2.7.2.zip
```

```bash
# Sur le bastion — extraire
cd /root
unzip awx-operator-2.7.2.zip
mv awx-operator-2.7.2 awx-operator
```

### 3.3 Créer les fichiers d'installation

```bash
mkdir -p /root/awx-install && cd /root/awx-install

# Copier la structure config
cp -r /root/awx-operator/config /root/awx-install/config
```

```bash
# kustomization.yaml
cat > /root/awx-install/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: awx
resources:
  - config/default
  - awx-instance.yaml
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.7.2
EOF
```

```bash
# awx-instance.yaml — configuration de l'instance AWX
cat > /root/awx-install/awx-instance.yaml << 'EOF'
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: ClusterIP
  ingress_type: Route
  hostname: awx.apps.caas.eazytraining.lab

  # Images depuis la registry locale (air-gapped)
  image: local-registry.caas.eazytraining.lab:5000/ansible/awx
  image_version: "23.4.0"
  redis_image: local-registry.caas.eazytraining.lab:5000/redis
  redis_image_version: "7"
  postgres_image: local-registry.caas.eazytraining.lab:5000/postgres
  postgres_image_version: "13"
  init_container_image: local-registry.caas.eazytraining.lab:5000/ansible/awx-ee
  init_container_image_version: latest
  control_plane_ee_image: local-registry.caas.eazytraining.lab:5000/ansible/awx-ee:latest

  # Stockage persistant NFS
  postgres_storage_class: nfs-client
  postgres_storage_requirements:
    requests:
      storage: 8Gi
  projects_persistence: true
  projects_storage_class: nfs-client
  projects_storage_size: 5Gi

  # Ressources
  web_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
  task_resource_requirements:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
EOF
```

### 3.4 Déployer AWX Operator

```bash
cd /root/awx-install

# Appliquer via kustomize
kustomize build . | oc apply -f -

# Surveiller l'operator
watch oc get pods -n awx
# Attendre : awx-operator-controller-manager 2/2 Running
```

### 3.5 Déployer l'instance AWX

```bash
# Une fois l'operator Running — appliquer l'instance
oc apply -f /root/awx-install/awx-instance.yaml

# Surveiller le déploiement (~15-20 min)
watch oc get pods -n awx
```

État final attendu :

```
NAME                                               READY   STATUS
awx-operator-controller-manager-xxx               2/2     Running
awx-postgres-13-0                                 1/1     Running
awx-task-xxx                                      4/4     Running
awx-web-xxx                                       3/3     Running
```

### 3.6 Problème connu — authentification PostgreSQL

Sur OKD avec RHCOS, PostgreSQL utilise `scram-sha-256` par défaut mais AWX 23.x
utilise psycopg qui ne le supporte pas toujours. Fix :

```bash
# Récupérer le mot de passe PostgreSQL
AWX_PASS=$(oc get secret awx-postgres-configuration -n awx \
  -o jsonpath='{.data.password}' | base64 -d)

# Changer l'auth method en md5
oc exec awx-postgres-13-0 -n awx -- \
  psql -U awx -d awx -c "ALTER SYSTEM SET password_encryption = 'md5';"

oc exec awx-postgres-13-0 -n awx -- \
  psql -U awx -d awx -c "SELECT pg_reload_conf();"

# Modifier pg_hba.conf
oc exec awx-postgres-13-0 -n awx -- bash -c \
  "sed -i 's/scram-sha-256/md5/g' \
  /var/lib/postgresql/data/pgdata/pg_hba.conf"

# Remettre le mot de passe et redémarrer
oc exec awx-postgres-13-0 -n awx -- \
  psql -U awx -d awx \
  -c "ALTER USER awx WITH PASSWORD '$AWX_PASS';"

oc delete pod awx-postgres-13-0 -n awx
```

### 3.7 Accès à AWX

```bash
# Récupérer l'URL
oc get route awx -n awx -o jsonpath='{.spec.host}' && echo ""

# Récupérer le mot de passe admin
oc get secret awx-admin-password -n awx \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

> **Accès depuis un poste externe** : L'URL AWX est en IP privée (`192.168.110.x`).
> Utiliser un tunnel SSH :
> ```bash
> ssh -L 8443:192.168.110.9:443 root@<IP_PUBLIQUE_BASTION> -N
> ```
> Puis accéder via `https://awx-awx.apps.caas.eazytraining.lab:8443`

---

## 4. Configuration initiale AWX

### 4.1 Credential SSH

```
Ressources → Informations d'identification → Ajouter
  Nom              : okd-ssh-core
  Type             : Machine
  Nom d'utilisateur : core
  Clé privée SSH   : [contenu de /root/.ssh/id_rsa]
→ Enregistrer
```

> **Point critique** : Sur OKD, AWX exécute les jobs dans des containers runner
> qui n'ont pas accès aux volumes montés sur `awx-task`. La clé SSH doit être
> accessible depuis le répertoire projet NFS.

```bash
# Copier la clé SSH dans le répertoire projet NFS
NFS_PROJ="<chemin_NFS_du_projet_AWX>"
mkdir -p $NFS_PROJ/.ssh
cp /root/.ssh/id_rsa $NFS_PROJ/.ssh/id_rsa
chown 1000:0 $NFS_PROJ/.ssh/id_rsa
chmod 400 $NFS_PROJ/.ssh/id_rsa
```

Configuration inventaire pour utiliser la clé :
```yaml
# Variables de l'inventaire OKD-Cluster
---
ansible_user: core
ansible_ssh_extra_args: >-
  -i /runner/project/.ssh/id_rsa
  -o StrictHostKeyChecking=no
  -o UserKnownHostsFile=/dev/null
```

### 4.2 Inventaire OKD

```
Ressources → Inventaires → Ajouter → Inventaire
  Nom : OKD-Cluster
→ Enregistrer

→ Hôtes → Ajouter (répéter pour chaque worker) :
  ocp-worker-01.caas.eazytraining.lab  → ansible_host: 192.168.110.114
  ocp-worker-02.caas.eazytraining.lab  → ansible_host: 192.168.110.115
  ocp-worker-03.caas.eazytraining.lab  → ansible_host: 192.168.110.116
  ocp-worker-04.caas.eazytraining.lab  → ansible_host: 192.168.110.117

→ Groupes → Ajouter → okd_workers
→ Associer les 4 workers au groupe okd_workers
```

### 4.3 Projet AWX (mode Manuel)

> En environnement air-gapped, utiliser le mode Manuel au lieu de Git.
> Les playbooks sont copiés directement dans le dossier NFS du projet.

```
Ressources → Projets → Ajouter
  Nom                     : AWX-Enterprise
  Type SCM                : Manuel
  Répertoire Playbook     : _8__awx_enterprise
→ Enregistrer
```

Structure du projet :
```
/var/lib/awx/projects/_8__awx_enterprise/
├── .ssh/
│   └── id_rsa              ← clé SSH (mode 400, owner 1000:0)
├── inventory/
│   └── hosts.yml
└── playbooks/
    ├── 01-pre-patch-scan.yml
    ├── 02-patch-os.yml
    ├── 03-post-patch-report.yml
    ├── 04-cis-audit.yml
    ├── 05-cis-hardening.yml
    └── 06-cis-verify.yml
```

> **Note importante sur RHCOS** : Fedora CoreOS est un OS immutable.
> Il n'y a **aucun Python** installé. Tous les playbooks utilisent
> `gather_facts: false` et le module `ansible.builtin.raw` au lieu
> des modules standard Ansible.

---

## 5. Cas 1 — Workflow Patch OS avec approbation

### Contexte enterprise

En production bancaire (BNP Paribas, RATP), tout patch nécessite :
1. Un rapport d'état avant intervention
2. Une approbation formelle du Change Manager
3. Une exécution contrôlée (rolling, un node à la fois)
4. Un rapport de clôture (preuve d'intervention)

### 5.1 Playbook 01 — Pre-patch scan

```yaml
# playbooks/01-pre-patch-scan.yml
---
- name: Pre-patch scan — inventaire des mises à jour disponibles
  hosts: okd_workers
  gather_facts: false

  tasks:

    - name: Récupérer le hostname
      ansible.builtin.raw: hostname
      register: host_name
      changed_when: false

    - name: Récupérer la version du kernel
      ansible.builtin.raw: uname -r
      register: kernel_version
      changed_when: false

    - name: Récupérer l'uptime
      ansible.builtin.raw: uptime
      register: uptime_info
      changed_when: false

    - name: Vérifier les services OKD
      ansible.builtin.raw: |
        systemctl is-active kubelet 2>/dev/null && \
        systemctl is-active crio 2>/dev/null
      register: services_status
      changed_when: false
      ignore_errors: true

    - name: Vérifier l'espace disque
      ansible.builtin.raw: df -h /
      register: disk_space
      changed_when: false

    - name: Statut rpm-ostree
      ansible.builtin.raw: rpm-ostree status 2>/dev/null | head -10
      register: ostree_status
      changed_when: false
      ignore_errors: true

    - name: Afficher le rapport pre-patch
      ansible.builtin.debug:
        msg:
          - "================================================"
          - "HOST     : {{ host_name.stdout | trim }}"
          - "KERNEL   : {{ kernel_version.stdout | trim }}"
          - "UPTIME   : {{ uptime_info.stdout | trim }}"
          - "DISQUE   : {{ disk_space.stdout_lines[-1] }}"
          - "KUBELET  : {{ services_status.stdout_lines[0] | default('unknown') }}"
          - "CRIO     : {{ services_status.stdout_lines[1] | default('unknown') }}"
          - "RPM-OSTREE:"
          - "{{ ostree_status.stdout_lines[:5] | default(['N/A']) }}"
          - "================================================"
```

### 5.2 Playbook 02 — Patch OS (rolling)

```yaml
# playbooks/02-patch-os.yml
---
- name: Patch OS — mise à jour sécurisée rolling RHCOS
  hosts: okd_workers
  gather_facts: false
  serial: 1   # UN NODE À LA FOIS — critique pour la HA

  tasks:

    - name: Récupérer le hostname
      ansible.builtin.raw: hostname
      register: host_name
      changed_when: false

    - name: Cordon du node avant patch
      ansible.builtin.shell: |
        oc adm cordon {{ inventory_hostname }} \
          --kubeconfig /root/.kube/config
      delegate_to: localhost
      changed_when: true
      ignore_errors: true

    - name: Drain du node
      ansible.builtin.shell: |
        oc adm drain {{ inventory_hostname }} \
          --ignore-daemonsets \
          --delete-emptydir-data \
          --force \
          --timeout=120s \
          --kubeconfig /root/.kube/config
      delegate_to: localhost
      changed_when: true
      ignore_errors: true

    - name: Vérifier statut rpm-ostree
      ansible.builtin.raw: rpm-ostree status 2>/dev/null | head -5
      register: ostree_status
      changed_when: false
      ignore_errors: true

    - name: Afficher statut rpm-ostree
      ansible.builtin.debug:
        msg: "{{ ostree_status.stdout_lines | default(['rpm-ostree N/A']) }}"

    - name: Uncordon du node après patch
      ansible.builtin.shell: |
        oc adm uncordon {{ inventory_hostname }} \
          --kubeconfig /root/.kube/config
      delegate_to: localhost
      changed_when: true
      ignore_errors: true

    - name: Résultat du patch
      ansible.builtin.debug:
        msg:
          - "Node   : {{ host_name.stdout | trim }}"
          - "Status : PATCH COMPLETED ✓"
```

### 5.3 Playbook 03 — Post-patch report

```yaml
# playbooks/03-post-patch-report.yml
---
- name: Post-patch report — vérification finale RHCOS
  hosts: okd_workers
  gather_facts: false

  tasks:

    - name: Hostname post-patch
      ansible.builtin.raw: hostname
      register: host_name
      changed_when: false

    - name: Kernel post-patch
      ansible.builtin.raw: uname -r
      register: kernel_post
      changed_when: false

    - name: Statut services OKD
      ansible.builtin.raw: |
        echo "kubelet=$(systemctl is-active kubelet 2>/dev/null)"
        echo "crio=$(systemctl is-active crio 2>/dev/null)"
      register: services_post
      changed_when: false
      ignore_errors: true

    - name: Rapport final
      ansible.builtin.debug:
        msg:
          - "================================================"
          - "POST-PATCH REPORT"
          - "HOST     : {{ host_name.stdout | trim }}"
          - "KERNEL   : {{ kernel_post.stdout | trim }}"
          - "SERVICES :"
          - "{{ services_post.stdout_lines | default(['N/A']) }}"
          - "STATUS   : COMPLETED ✓"
          - "================================================"
```

### 5.4 Créer les Job Templates
![Workflow Patch OS Visualiseur](docs/screenshots/03-workflow-patch-os-visualizer.png)
*Visualiseur du workflow Patch OS avec approbation*


```
Ressources → Modèles → Ajouter → Modèle de job

─── 01-Pre-Patch-Scan ───
  Inventaire  : OKD-Cluster
  Projet      : AWX-Enterprise
  Playbook    : playbooks/01-pre-patch-scan.yml
  Credentials : okd-ssh-core
  Verbosité   : 0 (Normal)

─── 02-Patch-OS ───
  Inventaire  : OKD-Cluster
  Projet      : AWX-Enterprise
  Playbook    : playbooks/02-patch-os.yml
  Credentials : okd-ssh-core

─── 03-Post-Patch-Report ───
  Inventaire  : OKD-Cluster
  Projet      : AWX-Enterprise
  Playbook    : playbooks/03-post-patch-report.yml
  Credentials : okd-ssh-core
```

### 5.5 Créer le Workflow avec approbation

```
Ressources → Modèles → Ajouter → Modèle de flux de travail
  Nom : WORKFLOW-Patch-OS-Enterprise
→ Enregistrer → Visualiseur
```

Construction du workflow :

```
[START]
   ↓
[01-Pre-Patch-Scan]
   ↓ En cas de succès
[⏸ Approbation : "Valider le patch en production ?"]
   Timeout : 3600 secondes
   ↓ Approuvé
[02-Patch-OS]
   ↓ En cas de succès
[03-Post-Patch-Report]
```

### 5.6 Résultat obtenu
![Workflow Patch OS Résultat](docs/screenshots/04-workflow-patch-os-result.png)
*Résultat du workflow Patch OS — 4 nœuds réussis*

![Pre-Patch Scan Output](docs/screenshots/07-pre-patch-scan-output.png)
*Sortie du job Pre-Patch Scan — rapport par worker*


```
WORKFLOW-Patch-OS-Enterprise   ✅ Réussi   Total Nœuds : 4

DÉMARRER → 01-Pre-Patch-Scan (6s) → Approbation (5m26s)
        → 02-Patch-OS (14s) → 03-Post-Patch-Report (6s)
```

**Rapport pre-patch (extrait) :**
```
HOST     : ocp-worker-01.caas.eazytraining.lab
KERNEL   : 5.14.0-284.11.1.el9_2.x86_64
UPTIME   : up 7 days, 2:14
DISQUE   : /dev/sda  50G  12G  38G  25% /
KUBELET  : active
CRIO     : active
```

---

## 6. Cas 2 — Hardening CIS des workers OKD

### Contexte enterprise

Le **CIS Benchmark** (Center for Internet Security) est la référence de sécurité
exigée dans les environnements régulés (banque, défense, télécoms).
Ce workflow implémente un cycle complet : audit → approbation RSSI → hardening → vérification.

### 6.1 Playbook 04 — CIS Audit

```yaml
# playbooks/04-cis-audit.yml
---
- name: CIS Audit — évaluation conformité sécurité OKD workers
  hosts: okd_workers
  gather_facts: false

  tasks:

    - name: "CIS 5.2.4 — Vérifier SSH X11Forwarding"
      ansible.builtin.raw: |
        grep -i "^X11Forwarding" /etc/ssh/sshd_config 2>/dev/null \
          || echo "NOT_SET"
      register: ssh_x11
      changed_when: false

    - name: "CIS 5.2.5 — Vérifier SSH MaxAuthTries"
      ansible.builtin.raw: |
        grep -i "^MaxAuthTries" /etc/ssh/sshd_config 2>/dev/null \
          || echo "NOT_SET"
      register: ssh_maxauth
      changed_when: false

    - name: "CIS 5.2.6 — Vérifier SSH IgnoreRhosts"
      ansible.builtin.raw: |
        grep -i "^IgnoreRhosts" /etc/ssh/sshd_config 2>/dev/null \
          || echo "NOT_SET"
      register: ssh_rhosts
      changed_when: false

    - name: "CIS 3.1.1 — Vérifier IP forwarding"
      ansible.builtin.raw: sysctl net.ipv4.ip_forward 2>/dev/null
      register: ip_forward
      changed_when: false

    - name: "CIS 3.2.2 — Vérifier ICMP redirects"
      ansible.builtin.raw: |
        sysctl net.ipv4.conf.all.accept_redirects 2>/dev/null
      register: icmp_redirect
      changed_when: false

    - name: "CIS — Permissions /etc/passwd et /etc/shadow"
      ansible.builtin.raw: |
        stat -c "%n : %a %U %G" /etc/passwd /etc/shadow 2>/dev/null
      register: file_perms
      changed_when: false

    - name: Afficher le rapport CIS
      ansible.builtin.debug:
        msg:
          - "╔══════════════════════════════════════════════╗"
          - "║     RAPPORT AUDIT CIS - PRE-HARDENING        ║"
          - "╠══════════════════════════════════════════════╣"
          - "║ HOST : {{ inventory_hostname }}"
          - "╠══════════════════════════════════════════════╣"
          - "║ SSH CONTROLS                                  ║"
          - "║ X11Forward  : {{ ssh_x11.stdout | trim }}"
          - "║ MaxAuthTries: {{ ssh_maxauth.stdout | trim }}"
          - "║ IgnoreRhosts: {{ ssh_rhosts.stdout | trim }}"
          - "╠══════════════════════════════════════════════╣"
          - "║ NETWORK CONTROLS                              ║"
          - "║ IP Forward  : {{ ip_forward.stdout | trim }}"
          - "║ ICMP Redir  : {{ icmp_redirect.stdout | trim }}"
          - "╠══════════════════════════════════════════════╣"
          - "║ FILE PERMISSIONS                              ║"
          - "{{ file_perms.stdout_lines }}"
          - "╚══════════════════════════════════════════════╝"
```

### 6.2 Playbook 05 — CIS Hardening

```yaml
# playbooks/05-cis-hardening.yml
---
- name: CIS Hardening — application des contrôles sécurité
  hosts: okd_workers
  gather_facts: false
  serial: 1

  tasks:

    - name: "CIS 5.2.4 — Désactiver SSH X11Forwarding"
      ansible.builtin.raw: |
        if grep -q "^X11Forwarding" /etc/ssh/sshd_config; then
          sed -i 's/^X11Forwarding.*/X11Forwarding no/' /etc/ssh/sshd_config
        else
          echo "X11Forwarding no" >> /etc/ssh/sshd_config
        fi && echo "DONE"
      register: x11_result
      changed_when: "'DONE' in x11_result.stdout"

    - name: "CIS 5.2.5 — Configurer MaxAuthTries à 4"
      ansible.builtin.raw: |
        if grep -q "^MaxAuthTries" /etc/ssh/sshd_config; then
          sed -i 's/^MaxAuthTries.*/MaxAuthTries 4/' /etc/ssh/sshd_config
        else
          echo "MaxAuthTries 4" >> /etc/ssh/sshd_config
        fi && echo "DONE"
      register: maxauth_result
      changed_when: "'DONE' in maxauth_result.stdout"

    - name: "CIS 5.2.8 — Désactiver login root SSH"
      ansible.builtin.raw: |
        if grep -q "^PermitRootLogin" /etc/ssh/sshd_config; then
          sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
        else
          echo "PermitRootLogin no" >> /etc/ssh/sshd_config
        fi && echo "DONE"
      register: rootlogin_result
      changed_when: "'DONE' in rootlogin_result.stdout"

    - name: "CIS 3.2.2 — Désactiver ICMP redirects"
      ansible.builtin.raw: |
        sysctl -w net.ipv4.conf.all.accept_redirects=0 2>/dev/null
        sysctl -w net.ipv4.conf.default.accept_redirects=0 2>/dev/null
        echo "DONE"
      register: icmp_result
      changed_when: "'DONE' in icmp_result.stdout"
      ignore_errors: true

    - name: Rapport hardening
      ansible.builtin.debug:
        msg:
          - "╔══════════════════════════════════════════╗"
          - "║      HARDENING CIS APPLIQUÉ              ║"
          - "║ HOST : {{ inventory_hostname }}"
          - "║ X11Forwarding no        : ✓"
          - "║ MaxAuthTries 4          : ✓"
          - "║ PermitRootLogin no      : ✓"
          - "║ ICMP redirects=0        : ✓"
          - "╚══════════════════════════════════════════╝"
```

### 6.3 Hardening SSH via MachineConfig (méthode RHCOS)

> Sur RHCOS (OS immutable), la méthode recommandée est via `MachineConfig`
> plutôt que de modifier directement `/etc/ssh/sshd_config`.

```bash
# Générer le contenu base64
B64=$(echo -n "X11Forwarding no
MaxAuthTries 4
IgnoreRhosts yes
HostbasedAuthentication no
PermitRootLogin no
PermitEmptyPasswords no" | base64 -w0)

# Créer le MachineConfig
cat > /tmp/mc-ssh-hardening.yaml << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-ssh-hardening
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - path: /etc/ssh/sshd_config.d/99-cis-hardening.conf
        mode: 0600
        contents:
          source: data:text/plain;charset=utf-8;base64,${B64}
        overwrite: true
EOF

oc apply -f /tmp/mc-ssh-hardening.yaml

# Surveiller la propagation (rolling reboot des workers)
watch oc get mcp
```

### 6.4 Workflow CIS Hardening

```
Ressources → Modèles → Ajouter → Modèle de flux de travail
  Nom : WORKFLOW-CIS-Hardening

[START]
   ↓
[04-CIS-Audit]
   ↓ En cas de succès
[⏸ Approbation RSSI : "Valider le hardening CIS ?"]
   ↓ Approuvé
[05-CIS-Hardening]
   ↓ En cas de succès
[06-CIS-Verify]
```

### 6.5 Résultats obtenus
![Workflow CIS Résultat](docs/screenshots/05-workflow-cis-result.png)
*Résultat du workflow CIS Hardening — approbation RSSI*

![CIS Audit Output](docs/screenshots/08-cis-audit-output.png)
*Rapport CIS Audit — non-conformités identifiées*


**Audit pre-hardening (non-conformités identifiées) :**

| Contrôle CIS | Statut avant | Risque |
|---|---|---|
| X11Forwarding | NOT_SET | Moyen |
| MaxAuthTries | NOT_SET | Élevé |
| IgnoreRhosts | NOT_SET | Élevé |
| PermitRootLogin | NOT_SET | Critique |
| IP Forward | = 1 | Élevé |

**Post-hardening (conformités validées) :**

```bash
# Vérification via sshd -T
sudo sshd -T | grep -E 'maxauthtries|permitrootlogin|ignorerhosts|permitemptypasswords'

# Résultat :
maxauthtries 4           ✓
permitrootlogin no       ✓
ignorerhosts yes         ✓
permitemptypasswords no  ✓
```

---

## 7. Résultats et validation

### Récapitulatif des workflows déployés

| Workflow | Jobs | Approbation | Durée totale |
|---|---|---|---|
| WORKFLOW-Patch-OS-Enterprise | 3 | 1 (Change Manager) | ~6 min |
| WORKFLOW-CIS-Hardening | 3 | 1 (RSSI) | ~8 min |
| WORKFLOW-CICD-Deploy-A | 3 | 0 (automatique) | ~1 min |

### Récapitulatif des approches CI/CD

| Approche | Mécanisme | Cas d'usage |
|---|---|---|
| A — Trigger manuel | AWX UI → build → deploy | Déploiement contrôlé, maintenance |
| B — Webhook GitHub | GitHub push → nginx → AWX | Équipes utilisant GitHub.com |
| C — Gitea on-premise | Gitea push → AWX API token | Environnements air-gapped, souverains |

### Valeur ajoutée enterprise

```
Sans AWX                          Avec AWX
─────────────────────────────     ─────────────────────────────
Connexion manuelle node par node  Automatisé sur N nodes
Pas de validation formelle        Approbation traçée + timestamp
Aucun rapport                     Rapport pré/post intervention
Risque d'erreur humaine           Playbook testé et reproductible
Intervention heure ouvrée         Programmable la nuit
Aucune traçabilité                Audit trail complet (ISO 27001)
Rollback manuel long              Rollback automatique en 30s
Pas de CI/CD                      Pipeline complet multi-trigger
```

---

## 9. Dépannage

### Comparaison pattern enterprise

Ce workflow reproduit exactement le cycle utilisé dans les grandes entreprises :

```
Ticket ITSM (ServiceNow)
       ↓
Rapport d'état automatique (AWX Pre-scan)
       ↓
Approbation Change Manager
       ↓
Exécution automatisée (AWX Job)
       ↓
Rapport de clôture (AWX Post-report)
       ↓
Fermeture ticket ITSM
```

---

## 8. Dépannage

### AWX ne peut pas se connecter via SSH aux workers

```bash
# 1. Vérifier que la clé est dans le répertoire projet NFS
ls -la $NFS_PROJ/.ssh/id_rsa
# Attendu : -r-------- 1000 root

# 2. Tester SSH depuis le pod AWX
POD=$(oc get pods -n awx | grep awx-task | grep Running | awk '{print $1}')
oc exec -n awx $POD -c awx-task -- \
  ssh -o StrictHostKeyChecking=no \
      -i /runner/project/.ssh/id_rsa \
      core@192.168.110.114 hostname

# 3. Vérifier les variables d'inventaire AWX
# ansible_ssh_extra_args doit contenir -i /runner/project/.ssh/id_rsa
```

### Pods AWX en ImagePullBackOff

```bash
# Identifier le digest manquant
oc describe pod <pod> -n awx | grep "Failed to pull"

# Mirroer le digest depuis ocp (qui a accès à quay.io)
skopeo copy \
  --dest-creds="evreguser2:NewRegistryPass123!" \
  docker://quay.io/openshift/okd-content@sha256:<digest> \
  docker://192.168.110.9:5000/okd@sha256:<digest>
```

### MCP master bloqué (drain timeout)

```bash
# Identifier les PDBs bloquants
oc get pdb -A | awk 'NR==1 || $5=="0"'

# Scaler l'operator storage pour libérer les PDBs
oc scale deployment cluster-storage-operator \
  -n openshift-cluster-storage-operator --replicas=0
oc delete pdb --all -n openshift-cluster-storage-operator

# Drainer le node bloqué
oc adm drain <node> --ignore-daemonsets \
  --delete-emptydir-data --force --timeout=300s

# Remettre l'operator
oc scale deployment cluster-storage-operator \
  -n openshift-cluster-storage-operator --replicas=1
```

### Python absent sur RHCOS

RHCOS est un OS immutable sans Python. Utiliser dans tous les playbooks :

```yaml
gather_facts: false   # pas de module setup

# Utiliser ansible.builtin.raw au lieu de shell/command
- ansible.builtin.raw: hostname
  register: result
  changed_when: false
```

---

## 7. Cas 3 — Pipeline CI/CD multi-approches

### Contexte enterprise

Un pipeline CI/CD enterprise doit supporter plusieurs modes de déclenchement selon le contexte :
- **Trigger manuel** pour les déploiements contrôlés
- **Webhook GitHub** pour les équipes utilisant GitHub.com
- **Gitea on-premise** pour les environnements air-gapped ou souverains

### Architecture CI/CD

```
┌─────────────────────────────────────────────────────────────┐
│                   Pipeline CI/CD AWX                        │
│                                                             │
│  Approche A        Approche B         Approche C            │
│  (Manuel)          (GitHub)           (Gitea on-prem)       │
│     │                  │                    │               │
│     └──────────────────┴────────────────────┘               │
│                         ↓                                   │
│              WORKFLOW-CICD-Deploy-A                         │
│         ┌────────────────────────────┐                      │
│         │  07-Build-App (S2I OKD)    │                      │
│         │  ↓ succès    ↓ échec       │                      │
│         │  08-Deploy   09-Rollback   │                      │
│         └────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

### Playbook 07 — Build App (S2I via SSH delegation)

```yaml
# playbooks/07-build-app.yml
---
- name: CI/CD — Build application sur OKD
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    app_name: "simple-webapp"
    okd_namespace: "adalnanda"
    source_dir: "/root/openshift-administrator-training/tp4-deployment-app/s2i/sample-app-httpd/docker"
    ssh_cmd: "ssh -i /runner/project/.ssh/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 -o BatchMode=yes root@192.168.110.9"
    kubeconfig: "/root/ocp-install/auth/kubeconfig"

  tasks:

    - name: Vérifier que la BuildConfig existe
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc get bc/{{ app_name }} -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           -o jsonpath='{.metadata.name}' 2>/dev/null \
           && echo EXISTS || echo NOT_FOUND"
      register: bc_check
      changed_when: false

    - name: Lancer le build depuis le répertoire source
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc start-build bc/{{ app_name }} \
           -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           --from-dir={{ source_dir }} \
           --follow \
           --wait 2>&1"
      register: build_result
      changed_when: true
      timeout: 300

    - name: Récupérer le statut du build
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc get builds -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           -l buildconfig={{ app_name }} \
           --sort-by='.metadata.creationTimestamp' \
           -o jsonpath='{.items[-1].status.phase}'"
      register: build_status
      changed_when: false

    - name: Rapport build
      ansible.builtin.debug:
        msg:
          - "╔══════════════════════════════════════════╗"
          - "║         BUILD REPORT                     ║"
          - "║ Application : {{ app_name }}"
          - "║ Status      : {{ build_status.stdout | trim }}"
          - "╚══════════════════════════════════════════╝"

    - name: Échouer si build échoué
      ansible.builtin.fail:
        msg: "Build échoué : {{ build_status.stdout | trim }}"
      when: build_status.stdout | trim != "Complete"
```

### Playbook 08 — Deploy Verify

```yaml
# playbooks/08-deploy-verify.yml
---
- name: CI/CD — Vérification déploiement OKD
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    app_name: "simple-webapp"
    okd_namespace: "adalnanda"
    ssh_cmd: "ssh -i /runner/project/.ssh/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 -o BatchMode=yes root@192.168.110.9"
    kubeconfig: "/root/ocp-install/auth/kubeconfig"

  tasks:

    - name: Attendre que le rollout soit complet
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc rollout status deployment/{{ app_name }} \
           -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           --timeout=120s 2>&1"
      register: rollout_status
      changed_when: false

    - name: Récupérer l'URL de la route
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc get route {{ app_name }} \
           -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           -o jsonpath='{.spec.host}' 2>/dev/null"
      register: app_url
      changed_when: false

    - name: Tester l'URL HTTP
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "curl -sk -o /dev/null -w '%{http_code}' \
           http://{{ app_url.stdout | trim }} \
           --connect-timeout 10"
      register: http_status
      changed_when: false
      ignore_errors: true

    - name: Rapport déploiement
      ansible.builtin.debug:
        msg:
          - "╔══════════════════════════════════════════╗"
          - "║    DEPLOYMENT VERIFICATION REPORT        ║"
          - "║ Application : {{ app_name }}"
          - "║ URL         : http://{{ app_url.stdout | trim }}"
          - "║ HTTP Status : {{ http_status.stdout | default('N/A') | trim }}"
          - "║ STATUS : DEPLOYMENT VERIFIED ✓           ║"
          - "╚══════════════════════════════════════════╝"

    - name: Échouer si HTTP != 200
      ansible.builtin.fail:
        msg: "Application inaccessible — HTTP {{ http_status.stdout | trim }}"
      when:
        - http_status.stdout is defined
        - http_status.stdout | trim != "200"
```

### Playbook 09 — Rollback automatique

```yaml
# playbooks/09-rollback.yml
---
- name: CI/CD — Rollback automatique
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    app_name: "simple-webapp"
    okd_namespace: "adalnanda"
    ssh_cmd: "ssh -i /runner/project/.ssh/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10 -o BatchMode=yes root@192.168.110.9"
    kubeconfig: "/root/ocp-install/auth/kubeconfig"

  tasks:

    - name: Exécuter le rollback
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc rollout undo deployment/{{ app_name }} \
           -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} 2>&1"
      register: rollback_result
      changed_when: true
      ignore_errors: true

    - name: Attendre la stabilisation
      ansible.builtin.shell: |
        {{ ssh_cmd }} \
          "oc rollout status deployment/{{ app_name }} \
           -n {{ okd_namespace }} \
           --kubeconfig {{ kubeconfig }} \
           --timeout=120s 2>&1"
      register: rollback_status
      changed_when: false
      ignore_errors: true

    - name: Rapport rollback
      ansible.builtin.debug:
        msg:
          - "╔══════════════════════════════════════════╗"
          - "║         ROLLBACK REPORT                  ║"
          - "║ Application : {{ app_name }}"
          - "║ Status      : {{ rollback_status.stdout_lines[0] | default('N/A') }}"
          - "║ ⚠️  ROLLBACK EFFECTUÉ — Vérifier les logs ║"
          - "╚══════════════════════════════════════════╝"
```

### Workflow CI/CD avec gestion d'erreur

```
Ressources → Modèles → Ajouter → Modèle de flux de travail
  Nom : WORKFLOW-CICD-Deploy-A

[START]
   ↓
[07-Build-App]
   ↓ succès (vert)      ↓ échec (rouge)
[08-Deploy-Verify]    [09-Rollback]
```

> **Pattern enterprise** : le nœud `09-Rollback` sur la branche échec garantit
> la disponibilité de l'application — si le build échoue, la version précédente
> est automatiquement restaurée (self-healing).

### Approche A — Trigger manuel
![Workflow CICD Résultat](docs/screenshots/06-workflow-cicd-result.png)
*Résultat du workflow CI/CD — Build + Deploy réussis*


```
Ressources → Modèles → WORKFLOW-CICD-Deploy-A → Lancer ▶
```

**Résultat obtenu :**
```
07-Build-App      ✅  simple-webapp-13 → Complete
08-Deploy-Verify  ✅  HTTP 200 — Rollout successful
URL : http://simple-webapp-adalnanda.apps.caas.eazytraining.lab
```

### Approche B — Webhook GitHub → AWX

**Prérequis :**
- Port 9443 ouvert sur firewall Hetzner
- nginx configuré sur `ocp` comme reverse proxy SSL

```bash
# Configurer nginx sur ocp (176.9.223.229)
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/awx.key \
  -out /etc/nginx/ssl/awx.crt \
  -subj "/CN=awx-webhook/O=AWX" 2>/dev/null

cat > /etc/nginx/conf.d/awx-webhook.conf << 'EOF'
server {
    listen 9444 ssl;
    server_name _;
    ssl_certificate     /etc/nginx/ssl/awx.crt;
    ssl_certificate_key /etc/nginx/ssl/awx.key;
    location / {
        proxy_pass https://192.168.110.9:443;
        proxy_ssl_verify off;
        proxy_set_header Host awx-awx.apps.caas.eazytraining.lab;
    }
}
EOF

systemctl enable --now nginx
firewall-cmd --permanent --add-port=9444/tcp && firewall-cmd --reload
```

```bash
# Activer le webhook sur le workflow AWX
curl -sk -u "admin:<password>" \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"webhook_service": "github"}' \
  "https://awx-awx.apps.caas.eazytraining.lab/api/v2/workflow_job_templates/25/"

# Récupérer la clé webhook
curl -sk -u "admin:<password>" \
  -X POST \
  "https://awx-awx.apps.caas.eazytraining.lab/api/v2/workflow_job_templates/25/webhook_key/"
```

**Configuration GitHub :**
```
GitHub → repo → Settings → Webhooks → Add webhook
  Payload URL  : https://176.9.223.229:9444/api/v2/workflow_job_templates/25/github/
  Content type : application/json
  Secret       : <webhook_key>
  SSL verify   : ☐ Disable
  Events       : ☑ Just the push event
```

**Résultat obtenu :**
```
Push GitHub sur main
       ↓
GitHub POST webhook → nginx:9444 → AWX
       ↓
WORKFLOW-CICD-Deploy-A  ✅ Réussi  launch_type: webhook
```

### Approche C — Gitea on-premise
![Gitea Repository](docs/screenshots/09-gitea-repo.png)
*Gitea on-premise — repo awx-enterprise*
 + AWX API Token

**Architecture :**
```
Developer push → Gitea (OKD, namespace gitea)
                      ↓ webhook HTTP interne
              AWX /api/v2/workflow_job_templates/25/launch/
                      ↓ Bearer Token auth
              WORKFLOW-CICD-Deploy-A → Build → Deploy
```

**Déploiement Gitea sur OKD :**

```bash
# Créer le namespace et les permissions
oc new-project gitea
oc adm policy add-scc-to-user anyuid \
  system:serviceaccount:gitea:default

# PVC pour les données
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-data
  namespace: gitea
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 5Gi
EOF
```

```bash
# Déployer Gitea (image mirrorée localement)
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      securityContext:
        fsGroup: 1000
      containers:
      - name: gitea
        image: local-registry.caas.eazytraining.lab:5000/gitea/gitea:1.21
        ports:
        - containerPort: 3000
        env:
        - name: GITEA__database__DB_TYPE
          value: sqlite3
        - name: GITEA__server__ROOT_URL
          value: https://gitea-gitea.apps.caas.eazytraining.lab
        - name: GITEA__security__INSTALL_LOCK
          value: "true"
        volumeMounts:
        - name: gitea-data
          mountPath: /data
      volumes:
      - name: gitea-data
        persistentVolumeClaim:
          claimName: gitea-data
---
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: gitea
spec:
  selector:
    app: gitea
  ports:
  - name: http
    port: 3000
    targetPort: 3000
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: gitea
  namespace: gitea
spec:
  to:
    kind: Service
    name: gitea
  port:
    targetPort: http
  tls:
    termination: edge
EOF
```

```bash
# Créer l'admin Gitea
POD=$(oc get pods -n gitea -l app=gitea -o jsonpath='{.items[0].metadata.name}')
oc cp /tmp/create-admin.sh gitea/$POD:/tmp/create-admin.sh
oc exec -n gitea $POD -- \
  su git -s /bin/sh -c "sh /tmp/create-admin.sh"

# Créer le repo et pousser le code
curl -sk -u "adalbert:Adalbert2024" \
  -X POST -H "Content-Type: application/json" \
  -d '{"name":"awx-enterprise","private":false}' \
  https://gitea-gitea.apps.caas.eazytraining.lab/api/v1/user/repos

cd /root/awx-enterprise
git remote add gitea \
  https://adalbert:Adalbert2024@gitea-gitea.apps.caas.eazytraining.lab/adalbert/awx-enterprise.git
git push gitea main
```

**Créer le token API AWX et le webhook Gitea :**

```bash
# Créer un token API AWX
TOKEN=$(curl -sk -u "admin:<password>" \
  -X POST -H "Content-Type: application/json" \
  -d '{"description":"Gitea webhook token","scope":"write"}' \
  "https://awx-awx.apps.caas.eazytraining.lab/api/v2/users/1/personal_tokens/" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Créer le webhook Gitea → AWX
curl -sk -u "adalbert:Adalbert2024" \
  -X POST -H "Content-Type: application/json" \
  -d "{
    \"type\": \"gitea\",
    \"active\": true,
    \"events\": [\"push\"],
    \"config\": {
      \"url\": \"https://awx-awx.apps.caas.eazytraining.lab/api/v2/workflow_job_templates/25/launch/\",
      \"content_type\": \"json\",
      \"insecure_ssl\": \"1\"
    },
    \"authorization_header\": \"Bearer $TOKEN\"
  }" \
  https://gitea-gitea.apps.caas.eazytraining.lab/api/v1/repos/adalbert/awx-enterprise/hooks
```

**Résultat obtenu :**
```
Push Gitea sur main
       ↓
Gitea webhook POST + Bearer token
       ↓
AWX /api/v2/workflow_job_templates/25/launch/
       ↓
WORKFLOW-CICD-Deploy-A  ✅ Réussi
  ├── 07-Build-App      ✅ simple-webapp Complete
  └── 08-Deploy-Verify  ✅ HTTP 200
```

> **Point clé Gitea** : Les permissions NFS doivent être `chown -R 1000:1000`
> et le NFS provisioner doit être opérationnel. L'image httpd dans le Dockerfile
> doit pointer sur la registry locale.

---

## 8. Résultats et validation
![AWX Workflows List](docs/screenshots/02-awx-workflows-list.png)
*Liste des 3 workflows enterprise déployés dans AWX*

![Simple WebApp Déployée](docs/screenshots/10-webapp-deployed.png)
*Application simple-webapp déployée sur OKD via AWX CI/CD*


### Récapitulatif des workflows déployés

| Workflow | Jobs | Approbation | Durée totale |
|---|---|---|---|
| WORKFLOW-Patch-OS-Enterprise | 3 | 1 (Change Manager) | ~6 min |
| WORKFLOW-CIS-Hardening | 3 | 1 (RSSI) | ~8 min |
| WORKFLOW-CICD-Deploy-A | 3 | 0 (automatique) | ~1 min |

### Récapitulatif des approches CI/CD

| Approche | Mécanisme | Cas d'usage |
|---|---|---|
| A — Trigger manuel | AWX UI → build → deploy | Déploiement contrôlé |
| B — Webhook GitHub | GitHub push → nginx → AWX | Équipes GitHub.com |
| C — Gitea on-premise | Gitea push → AWX API token | Air-gapped / souverain |

### Valeur ajoutée enterprise

```
Sans AWX                          Avec AWX
─────────────────────────────     ─────────────────────────────
Connexion manuelle node par node  Automatisé sur N nodes
Pas de validation formelle        Approbation traçée + timestamp
Aucun rapport                     Rapport pré/post intervention
Risque d'erreur humaine           Playbook testé et reproductible
Intervention heure ouvrée         Programmable la nuit
Aucune traçabilité                Audit trail complet (ISO 27001)
Rollback manuel long              Rollback automatique en 30s
Pas de CI/CD                      Pipeline complet multi-trigger
```

### Pattern enterprise reproduit

```
Ticket ITSM (ServiceNow)
       ↓
Rapport d'état automatique (AWX Pre-scan)
       ↓
Approbation Change Manager / RSSI
       ↓
Exécution automatisée (AWX Workflow)
       ↓
Rapport de clôture (AWX Post-report)
       ↓
Fermeture ticket ITSM
```

---

## 9. Dépannage

### AWX ne peut pas se connecter via SSH aux workers

```bash
# Vérifier que la clé est dans le répertoire projet NFS
ls -la $NFS_PROJ/.ssh/id_rsa
# Attendu : -r-------- 1000 root

# Tester SSH depuis le pod AWX
POD=$(oc get pods -n awx | grep awx-task | grep Running | awk '{print $1}')
oc exec -n awx $POD -c awx-task -- \
  ssh -o StrictHostKeyChecking=no \
      -i /runner/project/.ssh/id_rsa \
      core@192.168.110.114 hostname

# Vérifier les variables d'inventaire AWX
# ansible_ssh_extra_args doit contenir -i /runner/project/.ssh/id_rsa
```

### oc absent dans le runner AWX

```bash
# Solution : déléguer via SSH vers local-registry qui a oc
# Utiliser dans les playbooks :
ssh_cmd: "ssh -i /runner/project/.ssh/id_rsa \
  -o StrictHostKeyChecking=no \
  -o BatchMode=yes \
  root@192.168.110.9"

# La clé publique de local-registry doit être dans authorized_keys
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
```

### Pods AWX en ImagePullBackOff

```bash
# Identifier le digest manquant
oc describe pod <pod> -n awx | grep "Failed to pull"

# Mirroer le digest depuis ocp (qui a accès à quay.io)
skopeo copy \
  --dest-creds="evreguser2:NewRegistryPass123!" \
  docker://quay.io/openshift/okd-content@sha256:<digest> \
  docker://192.168.110.9:5000/okd@sha256:<digest>
```

### MCP master bloqué (drain timeout)

```bash
# Identifier les PDBs bloquants
oc get pdb -A | awk 'NR==1 || $5=="0"'

# Scaler l'operator storage pour libérer les PDBs
oc scale deployment cluster-storage-operator \
  -n openshift-cluster-storage-operator --replicas=0
oc delete pdb --all -n openshift-cluster-storage-operator

# Drainer le node bloqué
oc adm drain <node> --ignore-daemonsets \
  --delete-emptydir-data --force --timeout=300s

# Remettre l'operator
oc scale deployment cluster-storage-operator \
  -n openshift-cluster-storage-operator --replicas=1
```

### Python absent sur RHCOS

RHCOS est un OS immutable sans Python. Utiliser dans tous les playbooks :

```yaml
gather_facts: false   # pas de module setup

# Utiliser ansible.builtin.raw au lieu de shell/command
- ansible.builtin.raw: hostname
  register: result
  changed_when: false
```

### NFS Provisioner en ImagePullBackOff

```bash
# Mirroer l'image NFS provisioner
skopeo copy \
  --dest-creds="evreguser2:NewRegistryPass123!" \
  docker://registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 \
  docker://192.168.110.9:5000/sig-storage/nfs-subdir-external-provisioner:v4.0.2

# Patcher le deployment
oc set image deployment/nfs-subdir-external-provisioner \
  -n nfs-provisioner \
  nfs-subdir-external-provisioner=local-registry.caas.eazytraining.lab:5000/sig-storage/nfs-subdir-external-provisioner:v4.0.2
```

### Gitea en CrashLoopBackOff (permissions NFS)

```bash
# Corriger les permissions du répertoire NFS Gitea
NFS_PATH=$(oc get pv $(oc get pvc gitea-data -n gitea \
  -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.nfs.path}')

chown -R 1000:1000 $NFS_PATH
chmod -R 755 $NFS_PATH
chmod 700 $NFS_PATH/ssh

oc rollout restart deployment/gitea -n gitea
```

---

## 10. Références

| Ressource | URL |
|---|---|
| AWX Operator | https://github.com/ansible/awx-operator |
| OKD Documentation | https://docs.okd.io/4.10 |
| CIS Benchmarks | https://www.cisecurity.org/cis-benchmarks |
| RHCOS MachineConfig | https://docs.okd.io/4.10/post_installation_configuration/machine-configuration-tasks.html |
| Ansible raw module | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/raw_module.html |
| Gitea API | https://gitea.io/api/swagger |
| AWX API | https://docs.ansible.com/automation-controller/latest/html/controllerapi |

---

## Auteur

**Adalbert** — DevOps / Platform Engineer  
Cluster OKD 4.10 UPI sur Hetzner bare-metal  
Formation EazyTraining — OpenShift Administrator

---

*Ce README documente une implémentation réelle sur un cluster OKD de production-like.
Toutes les commandes ont été testées et validées.*
