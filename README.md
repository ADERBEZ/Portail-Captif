# Portail Captif — LaSalle Avignon

Documentation technique complète du portail captif Wi-Fi de l'établissement.

---

## Sommaire

1. [Partie 1 — La page web du portail](#partie-1--la-page-web-du-portail)
   - [Structure des fichiers](#structure-des-fichiers)
   - [Fonctionnement de la page](#fonctionnement-de-la-page)
   - [Variables pfSense injectées](#variables-pfsense-injectées)
   - [Logique de validation](#logique-de-validation)
   - [Internationalisation (i18n)](#internationalisation-i18n)
   - [Déploiement des fichiers](#déploiement-des-fichiers)
2. [Partie 2 — Fonctionnement global et mise en service](#partie-2--fonctionnement-global-et-mise-en-service)
   - [Architecture générale](#architecture-générale)
   - [Flux d'authentification](#flux-dauthentification)
   - [Prérequis](#prérequis)
   - [Initialisation — étape par étape](#initialisation--étape-par-étape)
   - [Utilisation au quotidien](#utilisation-au-quotidien)
   - [Dépannage](#dépannage)

---

## Partie 1 — La page web du portail

### Structure des fichiers

La page web du portail captif repose sur trois fichiers déployés dans pfSense via l'onglet **File Manager** de la zone Captive Portal :

```
/usr/local/captiveportal/
├── index.php                        # Page principale d'authentification
├── captiveportal-logo_lasall_avi.jpg    # Logo de l'établissement
└── captiveportal-charte_lasalle.pdf     # Charte informatique téléchargeable
```

> **Attention au préfixe `captiveportal-`** : pfSense renomme automatiquement tous les fichiers statiques uploadés via le File Manager en leur ajoutant ce préfixe. Les références dans le code (`src`, `href`) doivent utiliser ce nom renommé.

---

### Fonctionnement de la page

La page `index.php` est un formulaire d'authentification qui s'affiche à tout utilisateur tentant d'accéder à Internet sur le réseau Wi-Fi de l'établissement, avant que son accès ne soit accordé.

Elle contient :

- Un **en-tête** avec le logo et le nom de l'établissement
- Un **formulaire de connexion** avec les champs identifiant et mot de passe
- Une **section "Charte informatique"** repliable, avec une case à cocher obligatoire et un bouton de téléchargement du PDF
- Une **fenêtre modale "Mot de passe oublié"** (information de contact, pas de reset automatique)
- Un **sélecteur de langue** (Français / English)
- Un **pied de page** avec mentions légales

---

### Variables pfSense injectées

pfSense remplace automatiquement certaines variables dans le fichier `index.php` au moment où il sert la page. Ce ne sont **pas** des variables PHP — pfSense effectue une simple substitution textuelle avant l'envoi au navigateur.

| Variable | Rôle |
|---|---|
| `$PORTAL_ACTION$` | URL de l'endpoint d'authentification pfSense (action du formulaire) |
| `$PORTAL_REDIRURL$` | URL vers laquelle l'utilisateur est redirigé après connexion réussie |
| `$PORTAL_ZONE$` | Identifiant de la zone du portail captif |

**Exemple dans le code :**

```html
<form method="post" action="$PORTAL_ACTION$">
  <input type="hidden" name="redirurl" value="$PORTAL_REDIRURL$" />
  <input type="hidden" name="zone"     value="$PORTAL_ZONE$" />
  ...
</form>
```

> Ne jamais modifier ces variables — pfSense les remplace avant d'envoyer la page au client.

---

### Logique de validation

La validation se fait côté client en JavaScript, **avant** la soumission du formulaire à pfSense. La fonction `handleLogin()` vérifie dans l'ordre :

```
1. Les champs identifiant et mot de passe sont-ils remplis ?
   → Non : affiche une alerte d'erreur, bloque la soumission

2. La case "Charte informatique" est-elle cochée ?
   → Non : affiche une alerte, ouvre automatiquement l'accordéon de la charte, bloque la soumission

3. Tout est valide
   → Laisse pfSense traiter la soumission (return true)
   → Indicateur visuel de chargement sur le bouton (⏳)
```

La charte n'est **jamais transmise** au serveur RADIUS — c'est un contrôle local côté web uniquement. RADIUS ne reçoit que l'identifiant (`auth_user`) et le mot de passe (`auth_pass`).

**Noms des champs HTML attendus par pfSense :**

```html
<input type="text"     name="auth_user" />   <!-- identifiant -->
<input type="password" name="auth_pass" />   <!-- mot de passe -->
```

---

### Internationalisation (i18n)

La page supporte le **français** et l'**anglais**, commutable à la volée via les boutons `FR` / `EN` dans l'en-tête. Les textes sont stockés dans un objet JavaScript `translations` :

```javascript
const translations = {
  fr: {
    login_title: "Connexion",
    err_fields:  "Veuillez remplir tous les champs.",
    err_charter: "Vous devez accepter la charte informatique.",
    // ...
  },
  en: {
    login_title: "Sign In",
    err_fields:  "Please fill in all fields.",
    err_charter: "You must accept the acceptable use policy.",
    // ...
  }
};
```

La fonction `setLang(lang)` met à jour dynamiquement tous les éléments portant l'attribut `data-i18n` avec la valeur de la clé correspondante. La langue est initialisée en français au chargement.

---

### Déploiement des fichiers

Dans pfSense, naviguer vers **Services → Captive Portal → [Nom de la zone]** :

**Onglet "Page Contents"** — uploader `index.php` :
- Ce fichier sera servi comme page de connexion du portail

**Onglet "File Manager"** — uploader les fichiers statiques :

| Fichier local | Nom après upload (pfSense préfixe automatiquement) |
|---|---|
| `logo_lasall_avi.jpg` | `captiveportal-logo_lasall_avi.jpg` |
| `charte_lasalle.pdf` | `captiveportal-charte_lasalle.pdf` |

> Les fichiers ne doivent **pas** contenir d'espaces dans leur nom. Renommer avant upload si nécessaire.

---

## Partie 2 — Fonctionnement global et mise en service

### Architecture générale

```
[Client Wi-Fi]
      │
      ▼
[Point d'accès Wi-Fi]
      │  (réseau OPT1 / Wireless LAN)
      ▼
[pfSense — Portail Captif]
      │           │
      │           ▼
      │     [Serveur RADIUS]
      │     IP : 172.16.0.6
      │     Port : 1812 (auth) / 1813 (accounting)
      │           │
      │           ▼
      │     [Base de données]
      │     (FreeRADIUS + MySQL / NPS + Active Directory)
      │
      ▼ (si authentification OK)
[Internet / Réseau de l'établissement]
```

**Interfaces pfSense impliquées :**

| Interface | Rôle | IP exemple |
|---|---|---|
| WAN | Accès Internet | 192.168.40.15 |
| LAN | Réseau d'administration | 192.168.1.1 |
| OPT1 (Wireless LAN) | Réseau Wi-Fi invité/élèves | 192.168.16.202 |

---

### Flux d'authentification

Voici le déroulé complet d'une connexion réussie :

```
1. Le client se connecte au Wi-Fi et obtient une IP via DHCP
   (routeur DHCP : 192.168.16.254)

2. Le client ouvre un navigateur et tente d'accéder à http://example.com

3. pfSense intercepte la requête HTTP et redirige le client vers :
   http://192.168.16.202:8002/index.php?zone=...

4. Le client remplit le formulaire (identifiant + mot de passe + charte) et valide

5. pfSense reçoit auth_user et auth_pass, puis envoie une requête RADIUS
   (protocole PAP, UDP port 1812) vers 172.16.0.6

6. Le serveur RADIUS vérifie les credentials dans sa base de données
   - Succès → Access-Accept → pfSense débloque l'accès réseau pour l'IP/MAC du client
   - Échec   → Access-Reject → pfSense affiche une erreur sur la page de connexion

7. Le client est redirigé vers l'URL d'origine (ou la page de bienvenue)
```

---

### Prérequis

Avant de démarrer la mise en service, vérifier que les éléments suivants sont en place :

- **pfSense CE** installé et accessible (par défaut sur `https://192.168.1.1`)
- **Interface Wireless LAN (OPT1)** configurée avec une IP statique sur pfSense
- **Serveur RADIUS** accessible depuis pfSense (FreeRADIUS ou NPS Windows)
- **Règles de pare-feu** autorisant UDP 1812/1813 de pfSense vers le serveur RADIUS
- **Base de données RADIUS** peuplée avec les comptes utilisateurs
- Fichiers du portail prêts : `index.php`, `logo_lasall_avi.jpg`, `charte_lasalle.pdf`

---

### Initialisation — étape par étape

#### Étape 1 — Configurer l'interface Wireless LAN

Naviguer dans **Interfaces → OPT1** :

- Activer l'interface
- Définir l'IP statique (ex : `192.168.16.202 /24`)
- Donner un nom descriptif : `Wireless_LAN`
- Sauvegarder et appliquer

#### Étape 2 — Configurer le serveur DHCP sur OPT1

Naviguer dans **Services → DHCP Server → Wireless_LAN** :

- Activer le serveur DHCP
- Plage d'adresses : ex. `192.168.16.10` à `192.168.16.200`
- Passerelle par défaut : `192.168.16.202` (IP du pfSense sur cette interface)
- DNS : au choix (8.8.8.8 ou le DNS interne)
- Sauvegarder

#### Étape 3 — Déclarer le serveur RADIUS dans pfSense

Naviguer dans **System → User Manager → Authentication Servers** → **Add** :

| Champ | Valeur |
|---|---|
| Descriptive name | RADIUS |
| Type | RADIUS |
| Hostname or IP address | 172.16.0.6 |
| Shared Secret | `votre_secret_partage` |
| Authentication port | 1812 |
| Accounting port | 1813 |
| Authentication protocol | PAP |

> **Important** : utiliser **PAP** et non MS-CHAPv2. Le portail captif transmet les credentials via HTTP, MS-CHAPv2 est incompatible avec ce mécanisme.

Sauvegarder puis tester via le bouton **Test** pour valider la connectivité.

#### Étape 4 — Créer la zone du portail captif

Naviguer dans **Services → Captive Portal** → **Add** :

**Onglet Configuration :**

| Champ | Valeur |
|---|---|
| Zone name | ex. `wifi_eleves` |
| Interfaces | OPT1 (Wireless_LAN) |
| Maximum concurrent connections | selon capacité |
| Idle timeout | ex. `3600` (secondes) |
| Hard timeout | ex. `28800` (8 heures) |

**Onglet Authentication :**

| Champ | Valeur |
|---|---|
| Authentication Method | RADIUS Authentication |
| Primary RADIUS server | RADIUS (serveur configuré à l'étape 3) |
| NAS IP Attribute | OPT1 (ou l'IP `192.168.16.202`) |

#### Étape 5 — Déployer les fichiers web

**Onglet "Page Contents"** :
- Uploader `index.php` → ce fichier devient la page d'authentification

**Onglet "File Manager"** :
- Uploader `logo_lasall_avi.jpg`
- Uploader `charte_lasalle.pdf`

> Vérifier après upload que les fichiers apparaissent bien avec le préfixe `captiveportal-` dans la liste.

#### Étape 6 — Configurer les règles de pare-feu

Naviguer dans **Firewall → Rules → Wireless_LAN** et vérifier/créer :

```
Règle 1 — Accès au portail (automatique par pfSense)
  Source : Wireless_LAN net
  Destination : This firewall (OPT1 address)
  Port : 8002 (ou le port attribué à la zone)

Règle 2 — Autoriser le DNS avant authentification
  Source : Wireless_LAN net
  Destination : any
  Port : 53 (UDP/TCP)

Règle 3 — Bloquer le reste (implicite, mais vérifier)
  pfSense bloque automatiquement les clients non authentifiés
```

#### Étape 7 — Vérifier la configuration côté RADIUS

Sur le serveur FreeRADIUS (`172.16.0.6`), vérifier que pfSense est bien déclaré comme client autorisé dans `/etc/freeradius/3.0/clients.conf` :

```
client pfsense {
    ipaddr    = 192.168.40.15    # IP WAN de pfSense (NAS IP)
    secret    = votre_secret_partage
    shortname = pfsense
}
```

Vérifier les logs en cas de problème :

```bash
tail -f /var/log/freeradius/radius.log
```

---

### Utilisation au quotidien

#### Pour un utilisateur (élève / prof)

1. Se connecter au réseau Wi-Fi de l'établissement
2. Ouvrir un navigateur et accéder à n'importe quelle URL en `http://`
3. La page de connexion s'affiche automatiquement
4. Saisir son **identifiant** et son **mot de passe** (comptes gérés par l'administrateur réseau)
5. Lire et **accepter la charte informatique** (case à cocher obligatoire)
6. Cliquer sur **Se connecter**
7. La navigation est débloquée jusqu'à expiration de la session (timeout configuré)

> Si la page ne s'affiche pas : s'assurer d'accéder à une URL en `http://` (pas `https://`). Les navigateurs modernes peuvent bloquer la redirection sur HTTPS.

#### Pour un administrateur — accès direct à la page

L'URL directe du portail captif (hors redirection automatique) est :

```
http://192.168.16.202:8002/index.php
```

Le port dépend du numéro de zone. pfSense attribue les ports dans l'ordre :

| Zone | Port HTTP | Port HTTPS |
|---|---|---|
| 1ère zone | 8000 | 8001 |
| 2ème zone | 8002 | 8003 |
| 3ème zone | 8004 | 8005 |

#### Pour un administrateur — gestion des sessions actives

Naviguer dans **Status → Captive Portal** pour voir les clients connectés, leur IP, leur MAC et leur durée de session. Il est possible de déconnecter manuellement un client depuis cette vue.

#### Ajouter ou modifier un compte utilisateur

Si l'authentification est gérée par **FreeRADIUS + MySQL**, ajouter un utilisateur :

```sql
INSERT INTO radcheck (username, attribute, op, value)
VALUES ('prenom.nom', 'Cleartext-Password', ':=', 'mot_de_passe');
```

Si l'authentification est gérée par **NPS + Active Directory**, créer ou modifier le compte dans l'Active Directory et s'assurer que l'utilisateur appartient au groupe autorisé dans la politique NPS (ex. `LASALLE\LYCEEBTS-TS1CIELI` ou `LASALLE\ADM-PROFS`).

---

### Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Page de connexion inaccessible | Mauvaise interface dans la zone | Vérifier que OPT1 est bien sélectionné dans la configuration de la zone |
| Erreur 414 "URI Too Long" | Liens `href="#"` ou base64 dans la page | Utiliser `href="javascript:void(0)"` et référencer les fichiers externes plutôt qu'en base64 |
| Authentification échoue toujours | Mauvais protocole (MS-CHAPv2) | Passer en PAP dans la configuration du serveur RADIUS dans pfSense |
| Authentification échoue (Access-Reject) | Client RADIUS non déclaré | Ajouter l'IP WAN de pfSense dans `clients.conf` du serveur RADIUS |
| Authentification échoue (timeout) | Problème de routage | Vérifier que pfSense peut pinguer `172.16.0.6` via Diagnostics → Ping |
| Le logo ne s'affiche pas | Mauvais nom de fichier | Vérifier que le fichier est uploadé et s'appelle bien `captiveportal-logo_lasall_avi.jpg` |
| Le PDF de la charte ne se télécharge pas | Mauvais nom de fichier | Vérifier que le PDF s'appelle bien `captiveportal-charte_lasalle.pdf` |
| La redirection ne se fait pas | L'utilisateur utilise HTTPS | Demander d'accéder à une URL `http://` pour déclencher la redirection |

---

*Documentation rédigée pour le projet portail captif — LaSalle Avignon.*
