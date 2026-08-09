# OPNsense_Riverbed
![OPNsense](https://img.shields.io/badge/OPNsense-Firewall-D94F00?logo=opnsense&logoColor=white)
![Type](https://img.shields.io/badge/Type-Scolaire-blue)

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
