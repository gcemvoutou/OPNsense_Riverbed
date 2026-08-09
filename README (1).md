# TP_Parefeu_OPNsense

**BTS SIO – Option SISR — Travaux Pratiques Réseau**

## Contexte

Ce projet pratique consistait à transformer une appliance réseau **Riverbed** en un pare-feu opérationnel via la distribution open-source **OPNsense**, basée sur FreeBSD. Il s'est déroulé en trois volets successifs : l'installation du système, la configuration des règles de filtrage, puis la mise en place d'un mécanisme de sauvegarde et d'un accès distant sécurisé.

## Objectifs

- Installer OPNsense sur une appliance dépourvue de sortie vidéo (accès console série uniquement)
- Configurer une segmentation réseau LAN / WLAN maîtrisée, avec des règles de filtrage précises
- Sécuriser l'exploitation du pare-feu via un mécanisme de sauvegarde/restauration et un accès distant en SSH

## Matériel et prérequis

- Appliance Riverbed avec accès console
- Câble console (RS-232/USB)
- Clé USB (64 Go minimum)
- PC équipé de Rufus et PuTTY
- Image ISO d'OPNsense (version amd64, format série)

---

## Volet 1 — Installation et configuration initiale

### Préparation et démarrage

Téléchargement de l'image d'installation OPNsense au format série (nécessaire car l'appliance ne dispose pas de sortie vidéo) et gravure sur clé USB avec **Rufus** (schéma de partition MBR, mode BIOS/UEFI-CSM).

![Configuration Rufus pour la gravure de la clé USB](screenshots/01-rufus-gravure.png)

Connexion à l'appliance via **PuTTY** :
- Connection type : `Serial`
- Serial line : `COM4` (port identifié via le Gestionnaire de périphériques Windows)
- Speed : `115200` bauds

Accès au BIOS (Aptio Setup Utility) et modification de l'ordre de démarrage pour placer la clé USB en priorité.

### Installation du système

Installation en format **ZFS** (GPT/UEFI Hybrid, configuration Stripe sans redondance) sur le disque interne de l'appliance, avec destruction du contenu existant.

### Configuration initiale via la console

**Assignation des interfaces (menu console, option 1)** :
```
Do you want to configure LAGGs now? [y/N]: n
Do you want to configure VLANs now? [y/N]: n
Enter the WAN interface name or 'a' for auto-detection: igb5
Enter the LAN interface name or 'a' for auto-detection: igb4
Enter the Optional interface 1 name (or nothing if finished): [Entrée]
Do you want to proceed? [y/N]: y
```

**Configuration IP du WAN (option 2, interface 2)** :
- IPv4 via DHCP : `y`
- IPv6 via DHCP6 : `y`
- Activer l'interface web (passage HTTPS → HTTP) : `y`

**Configuration IP du LAN (option 2, interface 1)** :
- IPv4 via DHCP : `n`
- Adresse IPv4 statique : `192.168.6.254`
- Masque de sous-réseau (bit count) : `24`
- Passerelle : laissée vide (interface LAN)
- IPv6 : `n`
- Serveur DHCP sur le LAN : `n`
- Restaurer les accès GUI par défaut : `y`

![Confirmation des adresses IP configurées en console (WAN + LAN)](screenshots/06-config-ip-lan.png)

### Accès à l'interface web

Configuration manuelle de la carte réseau du PC de test (`192.168.6.2/24`, passerelle `192.168.6.254`), vérification par `ping`, puis connexion à `http://192.168.6.254`.

![Tableau de bord OPNsense après connexion réussie](screenshots/08-dashboard-final.png)

**Résultat du volet 1** : interface WAN (`igb5`) en DHCP, interface LAN (`igb4`) en statique `192.168.6.254/24`, interface web accessible.

---

## Volet 2 — Configuration des règles de filtrage

### Principe fondamental

Dans OPNsense, les règles de filtrage sont évaluées **de haut en bas** sur l'interface d'entrée du trafic ; la première règle qui correspond à un paquet s'applique. Une règle se place toujours sur l'interface par laquelle le trafic **entre** dans le pare-feu.

### Exercice 1 — Autoriser DNS, HTTP et HTTPS depuis le LAN

Trois règles créées sur l'interface LAN (`Pass`, direction `in`, source `LAN network`, destination `any`), une par protocole : DNS (port 53), HTTP (port 80), HTTPS (port 443). Tout autre trafic reste implicitement bloqué — appliquer `any` en port aurait annulé toute la politique de sécurité.

![Liste des règles LAN après création](screenshots/02-liste-regles-lan.png)

### Exercice 2 — Bloquer l'accès du WLAN vers le LAN

Après assignation de l'interface WLAN (`Interfaces → Assignments`), création d'une règle de blocage total :

| Champ | Valeur |
|---|---|
| Action | `Block` |
| Interface | `WLAN` |
| Direction | `in` |
| Protocole | `any` |
| Source | `WLAN net` |
| Destination | `LAN net` |
| Port destination | `any` |

![Configuration de la règle de blocage WLAN vers LAN](screenshots/03-regle-blocage-wlan-lan.png)

### Exercice 3 — Autoriser l'accès à un serveur web spécifique depuis le WLAN

Règle placée **au-dessus** de la règle de blocage pour être évaluée en priorité :

| Champ | Valeur |
|---|---|
| Action | `Pass` |
| Interface | `WLAN` |
| Protocole | `TCP` |
| Source | `WLAN net` |
| Destination | `192.168.10.100` (hôte unique, masque `/32`) |
| Port destination | `80` ou `443` |

> Cibler une IP unique est indispensable : `LAN net` comme destination aurait annulé la règle de blocage de l'exercice 2.

![Règle d'autorisation WLAN vers le serveur web](screenshots/04-regle-serveur-web.png)

### Exercice 4 — Autoriser l'accès aux imprimantes depuis le WLAN

Création d'un alias regroupant les IP des deux imprimantes, plutôt que deux règles dupliquées :

| Alias | Valeur |
|---|---|
| Nom | `Imprimantes_LAN` |
| Type | `Host(s)` |
| Contenu | `192.168.10.50`, `192.168.10.51` |

Puis une règle unique `Pass` sur l'interface WLAN ciblant cet alias en destination (protocole TCP, port `any`).

![Alias Imprimantes_LAN et règle associée](screenshots/05-alias-imprimantes.png)

**Résultat du volet 2** : segmentation réseau fonctionnelle — WLAN isolé du LAN par défaut, avec exceptions ciblées et documentées (serveur web, imprimantes).

---

## Volet 3 — Sauvegarde, restauration et accès SSH

### Sauvegarder la configuration

La configuration complète d'OPNsense (interfaces, règles, comptes, services) est centralisée dans un unique fichier XML. Procédure : `Système → Configuration → Sauvegardes`, options `Ne pas sauvegarder les données RRD` (cochée) et `Chiffrer ce fichier de configuration` (disponible), puis **Télécharger la configuration**.

![Page de sauvegarde et bouton de téléchargement](screenshots/01-page-sauvegarde.png)

### Restaurer la configuration

Depuis la section **Restauration** : sélection du fichier `.xml`, zones à restaurer `Tout (recommandé)`, options par défaut (redémarrage automatique, exclusion des paramètres console, purge de l'historique local).

> Le pare-feu redémarre automatiquement après restauration ; il faut patienter 2 à 3 minutes avant que l'interface web soit de nouveau accessible.

### Activer et utiliser le SSH

Dans `Système → Paramètres → Administration`, section **Secure Shell (SSH)** :

| Champ | Valeur |
|---|---|
| Serveur shell sécurisé | Activé |
| Connexion root | Autorisée |
| Méthode d'authentification | Mot de passe autorisé |
| Port SSH | `22` |
| Interfaces d'écoute | `Tout (recommandé)` |

![Paramètres du service Secure Shell activés](screenshots/03-parametres-ssh.png)

Connexion depuis un poste client via **PuTTY** (protocole SSH, port 22, adresse `192.168.6.254`), authentification en `root`. La bannière d'accueil OPNsense confirme l'accès et affiche l'état des interfaces réseau.

![Connexion SSH établie avec succès](screenshots/04-connexion-ssh-reussie.png)

**Résultat du volet 3** : mécanisme de sauvegarde/restauration opérationnel et accès distant en ligne de commande fonctionnel.

---

## Compétences mobilisées

- Administration système en ligne de commande (console série, SSH)
- Installation et configuration d'un système FreeBSD/OPNsense
- Configuration de règles de pare-feu et segmentation réseau (LAN/WLAN)
- Gestion de sauvegardes système (export/import de configuration)
- Utilisation d'outils d'administration à distance (PuTTY, Rufus)

---

## Captures d'écran à préparer

Dossier `screenshots/` à créer dans le repo, avec ces noms de fichiers exacts (les balises `![...]` sont déjà placées au bon endroit dans le texte ci-dessus) :

| Fichier | Contenu |
|---|---|
| `01-rufus-gravure.png` | Fenêtre Rufus pendant la gravure de la clé USB |
| `06-config-ip-lan.png` | Terminal console confirmant les IP WAN/LAN |
| `08-dashboard-final.png` | Tableau de bord OPNsense après connexion |
| `02-liste-regles-lan.png` | Liste des règles LAN (DNS, HTTP, HTTPS) |
| `03-regle-blocage-wlan-lan.png` | Formulaire de la règle de blocage WLAN → LAN |
| `04-regle-serveur-web.png` | Formulaire de la règle vers le serveur web |
| `05-alias-imprimantes.png` | Formulaire de l'alias `Imprimantes_LAN` |
| `01-page-sauvegarde.png` | Page de sauvegarde OPNsense |
| `03-parametres-ssh.png` | Section SSH des paramètres d'administration |
| `04-connexion-ssh-reussie.png` | Terminal PuTTY connecté en SSH |

*Si tu veux limiter le nombre de captures, garde en priorité : `01-rufus-gravure`, `06-config-ip-lan`, `03-regle-blocage-wlan-lan`, et `04-connexion-ssh-reussie` — une par volet, chacune racontant l'essentiel.*
