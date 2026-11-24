# TAVARDT France : Centre de Durcissement et Audit WordPress Upgrade Race Condition

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-5b3cf5.svg)](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Vulnerability](https://img.shields.io/badge/Vulnerability-Race%20Condition-red.svg)](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Dépôt officiel et référence technique pour l'atténuation de la vulnérabilité de **Race Condition lors de la Mise à Jour WordPress et Exécution de Code à Distance (RCE)**.

***

## 1. Vue d'Ensemble de la Vulnérabilité

Pendant la mise à jour du cœur, des thèmes ou des extensions, WordPress télécharge et extrait temporairement des archives ZIP dans `wp-content/upgrade/` avant leur validation et transfert final.

Si le serveur web permet l'interprétation de fichiers PHP dans les répertoires de contenu, un attaquant peut appeler directement les fichiers fraîchement extraits pendant la fenêtre temporelle de concurrence, ouvrant la voie à une Exécution de Code à Distance (RCE) avec les droits du serveur web.

### Indicateurs de Risque Majeurs :
* Répertoire `wp-content/upgrade/` accessible publiquement via HTTP
* Exécution FastCGI/PHP autorisée dans uploads et upgrade
* Absence de la directive `WP_TEMP_DIR` en dehors du DocumentRoot

***

## 2. Audit Déterministe via CLI

Les navigateurs web appliquant des restrictions de sécurité strictes (CORS et bac à sable), utilisez l'utilitaire officiel en Python directement depuis votre terminal ou chaîne CI/CD :

```bash
# 1. Télécharger l'outil officiel d'audit
curl -sO https://raw.githubusercontent.com/TAVARDT-France/wordpress-upgrade-race-checker-fr/main/audit_wp_race.py

# 2. Exécuter l'audit direct contre votre domaine
python audit_wp_race.py https://votre-domaine.fr
```

### Paramètres Disponibles :
* Cible en argument positionnel ou `-u https://votre-domaine.fr`
* `-t <secondes>` : Délai d'expiration en secondes (par défaut : 5)
* `-j` : Format JSON pour intégration dans les pipelines automatisés

***

## 3. Blueprints d'Atténuation

### A. Nginx (`nginx.conf`)
```nginx
location ~* /wp-content/upgrade/ {
    deny all;
    return 403;
}

location ~* ^/wp-content/(uploads|upgrade|languages|mu-plugins)/.*\.php$ {
    deny all;
    return 403;
}
```

### B. Apache (`.htaccess`)
```apache
<IfModule mod_authz_core.c>
    <FilesMatch "\.(php|php[0-9]|phtml|pl|py|jsp|asp|sh|cgi)$">
        Require all denied
    </FilesMatch>
</IfModule>
Options -Indexes
```

### C. WordPress (`wp-config.php`)
```php
define('WP_TEMP_DIR', dirname(__DIR__) . '/tmp/');
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', true);
```

### D. Cloudflare WAF Rule
```cloudflare
(http.request.uri.path contains "/wp-content/upgrade/") or
(http.request.uri.path contains "/wp-content/uploads/" and http.request.uri.path contains ".php")
```

***

## 4. Consultation en Ligne

Consultez le guide interactif en production :  
[https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)

***

&copy; 2026 TAVARDT France. Tous droits réservés.
