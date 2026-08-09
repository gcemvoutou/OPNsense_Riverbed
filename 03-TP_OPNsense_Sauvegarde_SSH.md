![OPNsense](https://img.shields.io/badge/OPNsense-Firewall-D94F00?logo=opnsense&logoColor=white)
![Type](https://img.shields.io/badge/Type-Scolaire-blue)

# Sauvegarde, restauration et accès distant SSH sur OPNsense

## Contexte

Ce projet s'inscrit dans la continuité des travaux précédents sur le pare-feu OPNsense (installation, puis configuration des règles de filtrage). L'objectif est ici double : sécuriser les manipulations futures grâce à un mécanisme de sauvegarde/restauration, et mettre en place un accès distant en ligne de commande via SSH.

## Objectifs

- Sauvegarder la configuration du système avant toute modification importante
- Savoir la restaurer en cas d'erreur
- Mettre en place un accès distant sécurisé en SSH pour administrer le pare-feu en ligne de commande

## 1. Sauvegarder la configuration d'OPNsense

L'ensemble de la configuration d'OPNsense (interfaces, règles de pare-feu, comptes, services, etc.) est centralisé dans un unique fichier au format **XML**, ce qui rend la sauvegarde particulièrement simple et rapide.

Procédure : `Système → Configuration → Sauvegardes`, section **Téléchargement** :
- Option `Ne pas sauvegarder les données RRD` : cochée (évite d'alourdir le fichier avec des données de statistiques non essentielles)
- Option `Chiffrer ce fichier de configuration` : disponible pour protéger le fichier par mot de passe si nécessaire
- Clic sur **Télécharger la configuration**

Le fichier obtenu (`config-<nom-du-pare-feu>-<date>.xml`) décrit chaque paramètre du pare-feu sous forme de balises et doit être conservé précieusement.

<!-- 📸 Capture à insérer ici : page de sauvegarde OPNsense avec le bouton de téléchargement -->
![Page de sauvegarde et bouton de téléchargement de la configuration](screenshots/01-page-sauvegarde.png)

## 2. Restaurer la configuration

En cas d'erreur de manipulation ou de besoin de revenir à un état antérieur, il est possible de réinjecter une sauvegarde précédemment téléchargée, depuis le même écran, section **Restauration** :
- Restaurer les zones : `Tout (recommandé)`
- Sélection du fichier `.xml` via **Choisir un fichier**
- Options cochées par défaut : `Redémarrer après une restauration réalisée avec succès`, `Exclure les paramètres de la console de l'importation`, `Effacer (complet) l'historique de la configuration locale`
- Clic sur **Restaurer la configuration**

> **Attention** : le pare-feu redémarre automatiquement pour appliquer la configuration restaurée. Il faut patienter 2 à 3 minutes avant que l'interface web soit de nouveau accessible.

## 3. Activer et accéder à OPNsense en SSH

Par défaut, l'accès SSH est désactivé sur OPNsense pour des raisons de sécurité.

### 3.1 Activation du SSH

Dans `Système → Paramètres → Administration`, section **Secure Shell (SSH)** :

| Champ | Valeur |
|---|---|
| Serveur shell sécurisé | Activé |
| Connexion root | Autorisée (indispensable pour se connecter avec le compte `root`) |
| Méthode d'authentification | Connexions avec mot de passe autorisées |
| Port SSH | `22` |
| Interfaces d'écoute | `Tout (recommandé)` |

<!-- 📸 Capture à insérer ici : section SSH des paramètres d'administration (options cochées) -->
![Paramètres du service Secure Shell activés](screenshots/03-parametres-ssh.png)

### 3.2 Connexion depuis un poste client (PuTTY)

Depuis un poste connecté au réseau LAN, connexion via **PuTTY** en protocole SSH sur le port 22, à l'adresse `192.168.6.254`. Après acceptation de l'empreinte du serveur (première connexion) et authentification avec le compte `root`, la bannière d'accueil d'OPNsense confirme l'accès, accompagnée d'un récapitulatif de l'état des interfaces réseau (LAN, WAN, WLAN) et des empreintes de clés SSH du serveur.

<!-- 📸 Capture à insérer ici : terminal PuTTY connecté en SSH avec la bannière OPNsense -->
![Connexion SSH établie avec succès sur OPNsense](screenshots/04-connexion-ssh-reussie.png)

## Résultat

- Un mécanisme de sauvegarde/restauration opérationnel, garantissant un retour arrière rapide en cas d'erreur de configuration
- Un accès distant en ligne de commande fonctionnel via SSH, permettant l'administration du pare-feu sans passer par l'interface graphique

## Compétences mobilisées

- ![Sauvegarde](https://img.shields.io/badge/Sauvegarde-XML_Export%2FImport-blue?style=flat-square) Gestion de sauvegardes système (export/import de configuration XML)
- ![Sécurité](https://img.shields.io/badge/Sécurité-SSH_Distant-green?style=flat-square) Administration à distance sécurisée en SSH
- ![Services](https://img.shields.io/badge/Services-Accès_Distant-orange?style=flat-square) Configuration de services d'accès distant (PuTTY, authentification)

