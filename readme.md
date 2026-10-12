# ECOCERT ANSIBLE

## Contexte

Ce dépôt contient les automatisations Ansible utilisées pour configurer et maintenir
des postes clients Debian de l’environnement Ecocert. Les playbooks permettent de
réaliser le bootstrap initial, d’appliquer une politique de complexité des mots de
passe et de mettre à jour les paquets et applications nécessaires.

Les inventaires disponibles sont `staging` et `production`. Le groupe Ansible ciblé
par les playbooks est `pc`.

---

## Structure du dépôt

L’organisation du dépôt suit la logique suivante :

```text
.
├── .github/
├── inventories/
├── playbooks/
├── roles/
└── secrets/
```

- **`inventories/`** : inventaires et variables propres aux environnements.
- **`playbooks/`** : playbooks d’administration et de configuration des postes.
- **`roles/`** : rôles Ansible réutilisables.
- **`secrets/`** : variables sensibles utilisées par le contrôle des mots de passe.

---

## Utilisation de firetoak_ecocert-ansible

### 1. Cloner le dépôt localement

```bash
git clone https://github.com/FireToak/firetoak_ecocert-ansible.git
cd firetoak_ecocert-ansible
```

### 2. Préparer l’environnement Ansible

Installer Ansible et la collection utilisée par le rôle d’authentification :

```bash
ansible-galaxy collection install ansible.posix
```

Avant toute exécution, récupérer la clé privée SSH dans le gestionnaire de mots
de passe Bitwarden et la placer à la racine du projet sous le nom
`id_ed25519_svc-ansible`. Ne jamais publier cette clé ni la transmettre dans le
dépôt.

### 3. Vérifier la connectivité

Les inventaires définissent le groupe `pc`. Le bootstrap utilise
l’utilisateur initial `debian`. Après le bootstrap, l’inventaire de production
utilise `svc-ansible` et la clé privée configurée dans ses variables ; l’inventaire
de staging doit être complété de la même manière avant une exécution distante.

```bash
ansible -i inventories/staging/hosts.yml pc -m ansible.builtin.ping
ansible -i inventories/production/hosts.yml pc -m ansible.builtin.ping
```

### 4. Exécuter le bootstrap initial

À utiliser sur un poste nouvellement provisionné. Le playbook s’exécute avec
`become: true`, crée `svc-ansible`, lui accorde sudo sans mot de passe, installe la
clé publique et désactive l’authentification SSH par mot de passe. Il applique
ensuite la politique de mots de passe et installe les paquets définis par
`pc_paquets` lorsque cette variable est présente dans l’inventaire.

```bash
# Environnement de staging : 10.0.20.252
ansible-playbook -i inventories/staging/hosts.yml playbooks/pc-bootstrap.yml -K

# Environnement de production : 192.168.4.72
ansible-playbook -i inventories/production/hosts.yml playbooks/pc-bootstrap.yml -K
```

- `ansible-playbook` exécute un playbook Ansible ;
- `-i inventories/.../hosts.yml` sélectionne l’inventaire ciblé ;
- `playbooks/pc-bootstrap.yml` indique le playbook de configuration initiale ;
- `-K` (`--ask-become-pass`) demande le mot de passe nécessaire à l’élévation de
  privilèges via `become`.

> [!INFO]
>
> L’option `-K` est nécessaire lors de ce premier bootstrap, car l’utilisateur `svc-ansible` n’existe pas encore sur le poste distant. La connexion initiale utilise donc l’utilisateur `debian`, qui doit fournir son mot de passe pour obtenir les privilèges administrateur et créer `svc-ansible`. Le rôle configure ensuite `svc-ansible` avec sudo sans mot de passe et une authentification SSH par clé ; les exécutions suivantes peuvent alors utiliser cet utilisateur sans `-K`, sous réserve que l’inventaire soit configuré en conséquence.

### 5. Vérifier la conformité des mots de passe

Le playbook installe `libpam-pwquality` et déploie la configuration définie dans
`roles/pc-password-check/files/pwquality.conf`. Cette politique impose notamment
une longueur minimale de 12 caractères et quatre classes de caractères.

```bash
ansible-playbook -i inventories/staging/hosts.yml playbooks/pc-password-check.yml
ansible-playbook -i inventories/production/hosts.yml playbooks/pc-password-check.yml
```

Le fichier `secrets/pc.yml` est chargé par ce playbook. Il doit être renseigné ou
chiffré avec Ansible Vault selon les variables réellement nécessaires à
l’environnement.

### 6. Mettre à jour le système et les applications

Le playbook met à jour le cache APT, applique une mise à niveau `dist` puis
installe `pc_paquets`. En production, cette liste contient `git`, `curl`, `vim`,
`btop` et `etckeeper`.

```bash
ansible-playbook -i inventories/staging/hosts.yml playbooks/pc-update-apps.yml
ansible-playbook -i inventories/production/hosts.yml playbooks/pc-update-apps.yml
```

La variable `pc_paquets` est actuellement définie dans l’inventaire de production.
Définir cette variable dans staging avant d’y exécuter ce playbook.

### 7. Tester un changement avant application

Utiliser le mode simulation et une vérification de syntaxe avant toute exécution
sur la production :

```bash
ansible-playbook --syntax-check -i inventories/production/hosts.yml playbooks/pc-bootstrap.yml
ansible-playbook --check --diff -i inventories/production/hosts.yml playbooks/pc-update-apps.yml
```

---

## Bonnes pratiques

1. **Ne jamais versionner de secret** : la clé privée SSH et tout mot de passe
   d’amorçage doivent être retirés du dépôt, révoqués s’ils ont été exposés, puis
   remplacés par des secrets injectés localement ou par un gestionnaire de secrets.
2. **Chiffrer les variables sensibles** : utiliser Ansible Vault pour
   `secrets/pc.yml`, par exemple :

   ```bash
   ansible-vault encrypt secrets/pc.yml
   ansible-playbook --ask-vault-pass -i inventories/production/hosts.yml playbooks/pc-password-check.yml
   ```

3. **Limiter les privilèges** : exécuter les playbooks uniquement sur le groupe
   attendu et vérifier l’inventaire avant d’utiliser `become: true`.
4. **Valider avant déploiement** : utiliser `--syntax-check`, puis `--check --diff`
   sur la production.
5. **Protéger la clé privée** : lorsqu’elle est stockée localement, ses permissions
   doivent être limitées à son propriétaire :

   ```bash
   chmod 600 id_ed25519_svc-ansible
   ```

-----

## 👨‍💻 Mainteneurs

- **Louis MEDO** | [LinkedIn](https://www.linkedin.com/in/louismedo/) | [Portfolio](https://louis.loutik.fr/) | [GitHub](https://github.com/FireToak) | [louis.medo@loutik.fr](mailto:louis.medo@loutik.fr)

-----

<div align="center">
<br>
<small><i>Dernière mise à jour : 12 octobre 2026</i></small>
</div>
