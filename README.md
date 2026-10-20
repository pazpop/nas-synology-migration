# Migrer les données d'un NAS Synology

[English version](README.en.md)

Copier ses données hors d'un Synology avant de le vendre ou de le remplacer.
Deux copies vérifiées : une sur un disque externe (USB), une sur un disque libéré dans le NAS.
Toutes deux en **ext4 simple** : elles se lisent sur n'importe quel Linux, sans DSM, sans mdadm.

Essayé sur un DS1621+ avec DSM 7. Les commandes se tapent en SSH, sur le NAS.

> **Règle d'or : on n'efface rien tant que deux copies ne sont pas vérifiées.**
>
> **Avant de commencer**, si possible : une sauvegarde de plus, dans le cloud (Hyper Backup vers Synology C2,
> une Storage Box, un service S3…). Elle protège aussi d'une erreur de manipulation pendant la migration.

## 1. Libérer un disque

Faire le ménage, pour que les données tiennent sur un seul volume :

- Désinstaller les applications qui ne servent plus (**Centre de paquets**).
- Supprimer les dossiers partagés inutiles (**Panneau de configuration → Dossier partagé**).
- Rassembler les autres sur un seul volume : **Dossier partagé → Modifier → Emplacement**.
- Puis, dans **Gestionnaire de stockage** : supprimer le volume et le groupe de stockage du disque à libérer.
- Ne jamais toucher au groupe qui porte les données.

## 2. Brancher le disque externe

- Sur un port USB **à l'arrière**. Ceux de la façade peuvent tourner en USB 2 : environ 30 Mo/s au lieu de 300.
- Vérifier la vitesse : `SuperSpeed` = USB 3, `high-speed` = USB 2.

  ```sh
  dmesg | grep -i -E 'usb.*(high-speed|superspeed)' | tail -3
  ```

- DSM le monte sous `/volumeUSB1/usbshare`. Voir son format :

  ```sh
  mount | grep volumeUSB
  ```

  ext4 de préférence. NTFS et exFAT ne gardent ni les propriétaires, ni les droits, ni les liens physiques.

- Le formater en ext4 (cela l'efface) : **Panneau de configuration → Périphériques externes**, choisir le disque,
  **Formater**, **Disque entier**, **ext4**. DSM y fait une partition ext4 simple, lisible sur tout Linux.

## 3. Activer SSH

- DSM : **Panneau de configuration → Terminal et SNMP → Activer le service SSH**.
- Depuis le PC :

  ```sh
  ssh compte_admin@adresse_du_nas
  sudo su -                         # devenir root, avec le mot de passe du compte
  ```

- Lancer les copies dans `screen` ou `tmux`, s'ils existent, ou avec `nohup`.
  Une coupure SSH arrêterait la copie. Relancer la même commande la reprend.

## 4. Copier vers le disque externe

Choisir les dossiers à garder : `ls -la /volume1`.
Les dossiers `@…` et `#…` sont internes à DSM : ne pas les copier.

```sh
rsync -aH --info=progress2 --exclude='@eaDir' --exclude='#recycle' --exclude='#snapshot' \
  /volume1/homes/mon_compte /volumeUSB1/usbshare/
```

Sans `/` après le dossier source, rsync crée `mon_compte/` dans la destination.

Vérifier. Rien ne doit s'afficher :

```sh
rsync -aHc --dry-run --itemize-changes --exclude='@eaDir' --exclude='#recycle' --exclude='#snapshot' \
  /volume1/homes/mon_compte /volumeUSB1/usbshare/
```

Disque en NTFS ou exFAT : remplacer `-aHc` par `-rc`. On compare alors le contenu seulement.

## 5. Préparer le disque libéré

DSM met une copie de son système (`md0`) et de son swap (`md1`) sur **chaque** disque.
Il faut l'en retirer avant de le repartitionner. Sinon, DSM voudra le « réparer ».

**Identifier le disque.** Les disques s'appellent `/dev/sata1`, `/dev/sata2`…

```sh
cat /proc/mdstat
smartctl -i /dev/sataX | grep -i -E 'serial|capacity'
```

- Les données sont dans `md2` (ou plus) : leurs disques sont interdits.
- Le disque libéré n'apparaît que dans `md0` et `md1`.
- Noter son numéro de série. Le relire avant chaque commande qui écrit.

**Le sortir du système et du swap de DSM.** Ils restent sur les autres disques.

```sh
mdadm /dev/md0 --fail /dev/sataXp1 --remove /dev/sataXp1
mdadm /dev/md1 --fail /dev/sataXp2 --remove /dev/sataXp2
cat /proc/mdstat                                   # sataX n'y est plus ; les données sont intactes
```

**Effacer ses signatures.** DSM n'a pas `wipefs` ; `mdadm` suffit.

```sh
mdadm --zero-superblock /dev/sataXp1 /dev/sataXp2
mdadm --examine /dev/sataXp1 /dev/sataXp2          # No md superblock detected
```

**Une partition, en ext4.** Seule étape sans retour : relire le numéro de série.

```sh
parted -s /dev/sataX mklabel gpt mkpart secours ext4 0% 100%
mkfs.ext4 -L secours /dev/sataX1
mkdir -p /mnt/secours && mount /dev/sataX1 /mnt/secours
df -h /mnt/secours
```

DSM affichera ce disque « non initialisé », et `md0`, `md1` « dégradés ».
C'est normal : ne cliquer ni sur **Réparer** ni sur **Créer**.

## 6. Copier vers ce disque

Les mêmes commandes qu'à l'étape 4, vers `/mnt/secours/` : la copie, puis la vérification.

Ordre de grandeur sur un DS1621+ : 126 Mo/s de disque à disque, 31 Mo/s vers un SSD branché en USB 2.

## 7. Avant d'éteindre le NAS

- `umount /mnt/secours`.
- Supprimer le cache SSD, s'il y en a un (**Gestionnaire de stockage → Cache SSD**).
- Exporter les machines virtuelles à garder (**Virtual Machine Manager → Exporter**).
- Dissocier le NAS du compte Synology, et de QuickConnect.
- Ne jamais remettre le disque de copie dans un Synology : DSM pourrait proposer de l'initialiser, donc de l'effacer.
- Pas besoin de réinitialiser le NAS : il suffit de retirer tous les disques. DSM et ses réglages sont sur les
  disques (la partition système `md0`), pas dans le NAS, et ne s'installent pas sur les NVMe M.2 ([Synology :
  installer DSM](https://kb.synology.com/fr-fr/DSM/tutorial/How_to_install_DSM)). Sans disque, le NAS apparaît
  « Non installé », comme neuf. La réinitialisation depuis DSM, elle, efface les disques présents.

## Licence

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.fr) — pazpop
