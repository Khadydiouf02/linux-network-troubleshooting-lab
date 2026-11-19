# Linux & Network Troubleshooting Lab

**Summary (EN):** Ubuntu 24.04 virtual machine lab where I practiced network diagnostics
(TCP/IP, DNS, ping, dig, ss) and system log analysis, then diagnosed and documented
three simulated incidents.

**Résumé (FR) :** Laboratoire personnel réalisé en novembre 2026 dans le cadre de ma
préparation à un stage en informatique. Objectif : comprendre comment une machine Linux
communique sur le réseau et savoir diagnostiquer des pannes simples de façon méthodique.

## Environnement

| Élément | Détail |
|---|---|
| Hyperviseur | VMware sur [Windows 11] |
| Système invité | Ubuntu 24.04 LTS Desktop |
| Ressources de la VM | 4 Go de RAM, 2 processeurs, disque de 25 Go |
| Réseau | Mode NAT (adresse 10.0.2.15/24, passerelle 10.0.2.2) |
| Outils | ip, ping, dig, nslookup, resolvectl, ss, journalctl, systemctl, nmcli, ufw, nginx, curl |

## 1. Diagnostics de base

### Adresse IP et interface
```bash
ip a
```
Interface `[ens33]`, adresse IPv4  obtenue par DHCP, adresse MAC `[08:00:27:xx:xx:xx]`.

![ip a](screenshots/01-ip-a.png)

### Passerelle par défaut
```bash
ip route
```
La route par défaut passe par  (passerelle NAT de VMware).

![ip route](screenshots/02-ip-route.png)

### Connectivité
```bash
ping -c 4 127.0.0.1    # pile réseau locale
ping -c 4 ####     # passerelle
ping -c 4 8.8.8.8      # Internet par adresse IP
ping -c 4 google.com   # Internet par nom (DNS)
```
[Résultat : 0 % de perte, latence moyenne d'environ XX ms.]

![ping](screenshots/03-ping.png)

### Résolution DNS
```bash
dig google.com
resolvectl status
```
[Statut NOERROR, réponse en XX ms. Le résolveur local 127.0.0.53 relaie vers ####]

![dig](screenshots/04-dig.png)

### Ports en écoute
```bash
sudo ss -tulpn
```
| Port | Protocole | Service | Exposé au réseau ? |
|---|---|---|---|
| 53 | UDP/TCP | systemd-resolved | Non (127.0.0.53) |
| 631 | TCP | cupsd (impression) | Non (127.0.0.1) |
| 80 | TCP | nginx | Oui (0.0.0.0) |

![ss](screenshots/06-ss.png)

### Journaux système
```bash
journalctl -b -p warning
journalctl -b | grep -i dhcp
```
[Ce que j'ai observé : par ex. l'obtention du bail DHCP pour l'adresse ip]

![journalctl](screenshots/07-journalctl.png)

## 2. Problèmes simulés

Méthode suivie : interface → adresse IP → passerelle → Internet par IP → DNS → service.

### Problème 1 : interface réseau désactivée
- **Symptômes :** `ping 8.8.8.8` renvoie « Network is unreachable ».
- **Diagnostic :** `ip a` montre l'interface en `state DOWN`, sans adresse IPv4 ; `ip route` est vide.
- **Cause :** interface désactivée (`ip link set ens33 down`).
- **Solution :** `sudo ip link set enp0s3 up` puis vérification avec `ip a` et `ping`.
- **Leçon :** toujours vérifier la couche la plus basse en premier.

![Problème 1](screenshots/10-pb1-symptome.png)

### Problème 2 : DNS mal configuré
- **Symptômes :** `ping 8.8.8.8` fonctionne mais `ping google.com` échoue
  (« Temporary failure in name resolution »).
- **Diagnostic :** `dig google.com` expire ; `dig @1.1.1.1 google.com` fonctionne ;
  `resolvectl status` montre un serveur DNS incorrect (192.0.2.53).
- **Cause :** serveur DNS manuel invalide dans la configuration NetworkManager.
- **Solution :** retour au DNS fourni par DHCP avec `nmcli`, puis vérification.
- **Leçon :** interroger un autre serveur DNS permet d'isoler rapidement la cause.

![Problème 2](screenshots/13-pb2-diagnostic.png)

### Problème 3A : le service web ne démarre plus
- **Symptômes :** `curl http://localhost` renvoie « Couldn't connect to server ».
- **Diagnostic :** `systemctl status nginx` indique `failed` ; `journalctl -u nginx`
  signale « unknown directive » dans `/etc/nginx/conf.d/erreur.conf` ; `nginx -t` échoue.
- **Solution :** suppression du fichier fautif, `nginx -t`, puis `systemctl start nginx`.
- **Leçon :** les journaux donnent souvent la cause exacte ; tester la configuration avant de redémarrer.

![Problème 3A](screenshots/15-pb3a-failed.png)

### Problème 3B : port 80 bloqué par le pare-feu
- **Symptômes :** le site répond dans la VM (`curl localhost`) mais pas depuis l'hôte
  (redirection de port 8080 → 80).
- **Diagnostic :** `ss` montre nginx en écoute sur 0.0.0.0:80 ; `ufw status` : refus par défaut,
  aucune règle pour le port 80 ; journal du noyau : `[UFW BLOCK] ... DPT=80`.
- **Solution :** `sudo ufw allow 80/tcp`.
- **Leçon :** « marche localement mais pas à distance » oriente vers un pare-feu.

![Problème 3B](screenshots/18-pb3b-ufw-block.png)

## 3. Ce que j'ai appris
- [Ex. : lire une table de routage et identifier la passerelle.]
- [Ex. : distinguer un problème de connectivité d'un problème DNS.]
- [Ex. : utiliser journalctl pour trouver la cause d'un service en échec.]

## 4. Difficultés rencontrées
- [Ex. : écran noir au premier démarrage, résolu en augmentant la mémoire vidéo à 128 Mo.]

## 5. Prochaines étapes
- Laboratoire de dépannage Windows (décembre 2026).

---
*Laboratoire personnel réalisé sur ma propre machine, à des fins d'apprentissage.*


## 5. Screenshots

### Network Diagnostics

![IP Configuration](linux-network-troubleshooting-labscreenshots/01-ip-a.png)

![IP Route](linux-network-troubleshooting-labscreenshots/02-ip-route.png)

![Ping Test](linux-network-troubleshooting-labscreenshots/03-ping.png)

![DNS Test](linux-network-troubleshooting-labscreenshots/04-dig.png)

![Port Verification](linux-network-troubleshooting-labscreenshots/06-ss.png)

### System Logs and Web Server

![Journalctl](linux-network-troubleshooting-labscreenshots/07-journalctl.png)

![Journalctl 2](linux-network-troubleshooting-labscreenshots/07-journalctl2.png.png)

![Journalctl 3](linux-network-troubleshooting-labscreenshots/07-journalctl3.png.png)

![Nginx Test](linux-network-troubleshooting-labscreenshots/08-nginx-ok.png)

### Troubleshooting Scenarios

![PB1 Symptom](linux-network-troubleshooting-labscreenshots/10-pb1-symptome.png)

![PB1 Symptom 2](linux-network-troubleshooting-labscreenshots/10-pb1-2-symptome.png)

![PB2 Symptom](linux-network-troubleshooting-labscreenshots/12-pb2-symptome.png)

![PB2 Diagnostic](linux-network-troubleshooting-labscreenshots/13-pb2-diagnostic.png)

![PB2 Resolution](linux-network-troubleshooting-labscreenshots/14-pb2-resolu.png)

![PB3A Failed](linux-network-troubleshooting-labscreenshots/15-pb3a-failed.png)

![PB3A Failed 2](linux-network-troubleshooting-labscreenshots/15-pb3a-2-failed.png)

![PB3A Failed 3](linux-network-troubleshooting-labscreenshots/15-pb3a-3-failed.png)

![PB3B Failed](linux-network-troubleshooting-labscreenshots/15-pb3b-failed.png)

![PB3B Failed 2](linux-network-troubleshooting-labscreenshots/15-pb3b-failed_1.png)

![Browser Error](linux-network-troubleshooting-labscreenshots/17-pb3b-navigateur-erreur.png)

![UFW Block](linux-network-troubleshooting-labscreenshots/18-pb3b-ufw-block.png)

![PB3B Resolution](linux-network-troubleshooting-labscreenshots/19-pb3b-resolu.png)




