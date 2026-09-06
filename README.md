# Lab CCNA — VTP & DTP : synchronisation de VLANs et négociation de trunks


## README GitHub

### 1. Contexte / Objectif

Dans un réseau avec plusieurs switchs, ajouter ou modifier un VLAN à la main sur chaque switch un par un devient vite ingérable et source d'erreurs à mesure que le réseau grandit. VTP (VLAN Trunking Protocol) permet de centraliser cette gestion : un switch "serveur" propage automatiquement les VLANs aux autres. En parallèle, DTP (Dynamic Trunking Protocol) gère la négociation automatique entre deux switchs pour savoir si un lien entre eux doit devenir un trunk (capable de transporter plusieurs VLANs) ou rester un simple port d'accès.

### 2. Topologie

3 switchs interconnectés, 9 PC répartis dessus, 4 VLANs différents configurés sur le réseau.

![[Pasted image 20260906132706.png]]

### 3. Ce que j'ai fait

- Configuré les 3 switchs chacun dans un mode VTP différent : serveur, client, et transparent, pour observer leur comportement respectif.
- Testé l'ajout d'un VLAN depuis chaque mode et observé la propagation (ou non) vers les autres switchs.
- Configuré progressivement les liens entre switchs en mode DTP "trunk administratif", puis désactivé DTP une fois les trunks établis.
- Assigné les ports connectés aux PC aux bons VLANs et vérifié la communication entre machines d'un même VLAN.

### 4. Décisions & points d'attention

- **VTP client** : comme attendu, impossible de créer ou modifier un VLAN localement — le switch client dépend entièrement du serveur pour toute mise à jour.
- **VTP transparent** : c'est le comportement le plus contre-intuitif du lab. Un switch en mode transparent ne synchronise jamais sa propre base de VLANs avec les autres, mais il **relaie** quand même les messages VTP qu'il reçoit vers les switchs suivants dans la topologie. Il agit donc comme un simple relais pour les autres, tout en restant indépendant lui-même. C'est une nuance qu'on comprend vraiment seulement en le testant, pas en le lisant.
- **VTP Serveur** : le mode le plus important du protocole — toute VLAN créée par le serveur VTP se synchronisera avec les autres clients ou serveurs. Plusieurs points de vigilance si plusieurs serveurs VTP se trouvent dans le réseau : celui avec la "Configuration Revision" la plus haute sera le serveur de référence, et donc tous les autres se synchroniseront dessus. De plus, VTP propose un système de domaine ; par défaut, aucun n'est défini, mais si vous en configurez un et qu'un client ne possède pas le même, la synchronisation ne se fera pas.
- **DTP** : j'ai choisi de configurer les trunks manuellement (mode "trunk administratif") plutôt que de laisser la négociation automatique se faire par défaut, puis j'ai désactivé DTP une fois les trunks établis. Dans un vrai réseau, désactiver DTP après coup est une bonne pratique de sécurité : ça évite qu'un appareil branché par erreur (ou avec de mauvaises intentions) puisse négocier automatiquement un trunk et accéder à tous les VLANs du réseau.

### 5. Vérification

Test de communication entre PC d'un même VLAN (réussi) et vérification de l'absence de communication entre VLANs différents (comportement attendu, pas de routage inter-VLAN configuré dans ce lab). Vérification de la table VLAN sur chaque switch selon son mode VTP pour confirmer la synchronisation (ou son absence).

![[Pasted image 20260906135653.png]]
*`show vtp status` sur les 3 switchs : SW1 (serveur) et SW3 (client) partagent la même Configuration Revision (6) et le même nombre de VLANs (8), preuve de la synchronisation. SW2 (transparent) reste à la révision 0 avec 9 VLANs — sa propre base locale, non synchronisée.*

J'ai également vérifié, sur les interfaces entre les switchs, que le mode administratif était bien en trunk et que DTP était désactivé.

![[Pasted image 20260906135445.png]]
*Capture 2 — `show interface switchport` : Administrative et Operational Mode confirmés en "trunk" sur les liens entre switchs, avec "Negotiation of Trunking: Off", confirmant la désactivation de DTP après établissement manuel des trunks.*

### 6. Ce que j'ai appris

au-delà du fonctionnement de ces deux protocoles, ce sont surtout les risques de leur utilisation qui m'ont marqué, malgré leur côté pratique et le gain de temps qu'ils apportent.
Pour DTP, un attaquant peut connecter son PC à un port en mode dynamique et envoyer des trames DTP pour se faire passer pour un switch, ce qui créera un lien trunk. Avec cet accès trunk, il pourra communiquer avec tous les appareils connectés sur les VLANs (VLAN hopping). 
Pour VTP, les risques ne viennent pas de l'extérieur mais de l'intérieur : le plus gros problème, ce sont les erreurs de manipulation — le fait de pouvoir influencer tout le réseau avec de petites manipulations est trop risqué. Le meilleur exemple est la remise en service d'un ancien switch : si ce dernier est connecté au réseau sans avoir été vérifié, il peut avoir sa "Configuration Revision" plus haute que le serveur VTP actuel, et donc remplacer tous les VLANs sur le réseau, ce qui créera une coupure générale et une grande perte de temps.
