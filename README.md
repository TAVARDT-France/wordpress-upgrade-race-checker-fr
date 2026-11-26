<p align="center">
  <a href="https://ag.tavardt.com/fr/">
    <img src="https://ag.tavardt.com/img/tavardt-banner.png" alt="TAVARDT" width="1024" />
  </a>
</p>

# Centre de Durcissement et Audit WordPress Upgrade Race Condition

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-5b3cf5.svg)](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Vulnerability](https://img.shields.io/badge/Vulnerability-Race%20Condition-red.svg)](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Spécification technique d'ingénierie défensive, modèles de configuration pour la production et utilitaire d'audit déterministe en ligne de commande pour la neutralisation des conditions de concurrence et l'Exécution de Code à Distance (RCE) lors des mises à jour de WordPress.

***

## Résumé Technique: La Condition de Concurrence lors des Mises à Jour

WordPress propulse plus de 40 pour cent des sites web mondiaux. Durant les phases de mise à niveau du cœur, des thèmes ou des extensions, la plateforme décompresse des archives ZIP dans un répertoire temporaire avant de transférer les fichiers vers leurs emplacements définitifs.

Par défaut, cette extraction s'opère dans le dossier `wp-content/upgrade/`, situé directement au sein de la racine accessible du serveur web (DocumentRoot).

Lorsque la configuration du serveur autorise l'interprétation de scripts PHP dans les dossiers de contenu, un tiers hostile peut exécuter des fichiers fraîchement décompressés pendant la fenêtre d'extraction:
* L'attaquant surveille ou provoque une phase de mise à jour sur le site visé.
* Les fichiers PHP temporaires deviennent exécutables avant l'application finale des droits d'accès.
* Des requêtes HTTP directes vers ces fichiers déclenchent une Exécution de Code à Distance (RCE) avec les privilèges du serveur web (`www-data` ou `nginx`).

Ce dépôt met à disposition des configurations vérifiées pour les environnements de production, des modèles d'isolation périmétrique et un utilitaire déterministe en Python (`audit_wp_race.py`) pour certifier la robustesse des défenses.

***

## Surfaces d'Attaque Critiques dans les Répertoires WordPress

### 1. L'Exposition Publique du Répertoire `wp-content/upgrade/`
La faille provient de l'emplacement d'un dossier de décompression transitoire dans une zone accessible via HTTP:
* **Fenêtre Temporelle de Risque:** Sur des serveurs mutualisés ou fortement sollicités, l'extraction d'archives volumineuses peut durer plusieurs secondes.
* **Vecteur d'Exploitation:** L'adversaire émet des rafales de requêtes HTTP concurrentes pour solliciter les fichiers avant leur déplacement ou suppression.
* **Directive Défensive:** Bloquer tout accès HTTP direct au dossier `wp-content/upgrade/` et déplacer le stockage temporaire hors du DocumentRoot.

### 2. Exécution Non Restreinte de Scripts PHP dans les Dossiers de Contenu
Les configurations par défaut redirigent couramment toute requête se terminant par `.php` vers FastCGI sans vérifier le dossier parent:
* **Fenêtre Temporelle de Risque:** Les répertoires de téléchargement requièrent des droits d'écriture pour recevoir médias et mises à jour.
* **Vecteur d'Exploitation:** La combinaison de droits d'écriture et de permissions d'exécution de scripts génère un risque immédiat d'exécution arbitraire.
* **Directive Défensive:** Interdire formellement l'exécution de scripts PHP dans les dossiers `uploads`, `upgrade`, `languages` et `mu-plugins` au niveau du serveur web.

### 3. Listage Automatique de Dossiers Actif (Directory Listing)
Les serveurs dont la configuration par défaut n'interdit pas l'indexation exposent l'arborescence des dossiers:
* **Fenêtre Temporelle de Risque:** Permet à des tiers d'observer en temps réel le nom des fichiers temporaires décompressés pendant une mise à jour.
* **Directive Défensive:** Désactiver l'indexation automatique des répertoires de manière globale sur les hôtes virtuels.

***

## Modèles de Durcissement pour la Production

Appliquez ces configurations éprouvées pour sécuriser vos serveurs web en production:

### 1. Configuration pour Nginx (`nginx.conf`)
Bloque l'accès au répertoire temporaire et désactive l'exécution de PHP dans les dossiers de contenu:

```nginx
# 1. Bloquer l acces direct au dossier de mise a niveau temporaire
location ~* /wp-content/upgrade/ {
    deny all;
    return 403;
}

# 2. Empecher l execution de PHP dans les dossiers de contenu statique
location ~* ^/wp-content/(uploads|upgrade|languages|mu-plugins)/.*\.php$ {
    deny all;
    return 403;
}
```

### 2. Configuration pour Apache (`.htaccess`)
Insérez cette directive dans les fichiers `.htaccess` situés dans `/wp-content/upgrade/` et `/wp-content/uploads/`:

```apache
# Interdire l execution de toute extension executable
<IfModule mod_authz_core.c>
    <FilesMatch "\.(php|php[0-9]|phtml|pl|py|jsp|asp|sh|cgi)$">
        Require all denied
    </FilesMatch>
</IfModule>

# Desactiver le listage des dossiers
Options -Indexes
```

### 3. Configuration dans WordPress (`wp-config.php`)
Déplace le stockage temporaire hors du DocumentRoot et désactive l'édition de fichiers:

```php
/* Deplacer le stockage temporaire de WordPress hors de la racine web */
define('WP_TEMP_DIR', dirname(__DIR__) . '/tmp/');

/* Restreindre l edition et l installation de fichiers depuis le tableau de bord */
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', true);
```

### 4. Règle Périmétrique Cloudflare WAF
Règle de bordure pour le rejet instantané des requêtes non autorisées:

```text
(http.request.uri.path contains "/wp-content/upgrade/") or
(http.request.uri.path contains "/wp-content/uploads/" and http.request.uri.path contains ".php")
Action: Block (Rejeter la connexion immediatement)
```

***

## Utilitaire Déterministe d'Audit CLI

Les navigateurs web imposent des mécanismes stricts d'isolation et de sécurité CORS qui empêchent l'analyse directe des sockets et la lecture complète des réponses HTTP. Pour permettre des audits techniques reproductibles et dépourvus de télémétrie, TAVARDT fournit un utilitaire en Python conçu pour les terminaux d'administration et les pipelines de validation continue.

### Fonctionnement de l'Analyse Déterministe

L'outil (`audit_wp_race.py`) applique un processus d'évaluation méthodique:
* **Validation d'URL & Connectivité:** Vérifie la syntaxe, normalise les protocoles et s'assure de la joignabilité de l'hôte.
* **Empreinte WordPress:** Valide la présence de la plateforme via les points d'accès de l'API REST, les fichiers statiques canoniques et les formulaires d'authentification.
* **Détection de Routeur SPA:** Distingue les codes d'erreur 403/404 réels des redirections génériques créées par les applications Single Page.
* **Sondage des Chemins Sensibles:** Analyse `/wp-content/upgrade/` et des fichiers tests PHP pour confirmer le rejet par le serveur.
* **Calcul de Score:** Attribue une note technique automatisée allant de 100 (Protégé) à 25 (Vulnérable).

### Mode d'Emploi

Téléchargez et lancez le programme directement contre le domaine WordPress cible:

```bash
# 1. Recuperer le script officiel depuis le depot
curl -sO https://raw.githubusercontent.com/TAVARDT-France/wordpress-upgrade-race-checker-fr/main/audit_wp_race.py

# 2. Executer l audit direct contre le domaine
python audit_wp_race.py https://votre-domaine.com

# 3. Exporter le resultat au format JSON
python audit_wp_race.py https://votre-domaine.com -j
```

***

## Foire Aux Questions Techniques (FAQ Architecture & Sécurité)

### 1. Quelle est l'origine de la condition de concurrence lors des mises à jour de WordPress?
Durant la mise à niveau du cœur, des thèmes ou des extensions, WordPress extrait les archives ZIP dans le répertoire temporaire `wp-content/upgrade/` avant de déplacer les fichiers finaux. Si la configuration du serveur permet l'exécution de scripts dans les dossiers de contenu, un tiers hostile peut solliciter les fichiers PHP pendant cette fenêtre d'extraction, obtenant une Exécution de Code à Distance avec les privilèges du serveur web.

### 2. Pour quelle raison le blocage HTTP du dossier wp-content/upgrade est-il indispensable?
Le dossier `wp-content/upgrade/` est prévu exclusivement pour les processus internes locaux du serveur PHP. Les clients externes sur le web n'ont aucune raison opérationnelle légitime d'accéder à ce chemin, et autoriser ces requêtes expose des fichiers transitoires à des tentatives d'exploitation automatisées.

### 3. Comment la relocalisation de la constante WP_TEMP_DIR neutralise-t-elle ce danger?
Le fait de définir `WP_TEMP_DIR` en dehors du DocumentRoot (par exemple dans `/var/www/tmp/` au lieu de `/wp-content/upgrade/`) rend les fichiers décompressés physiquement inaccessibles par voie web. Même si un attaquant devine le nom des fichiers temporaires, le serveur web ne peut ni les délivrer ni les exécuter car ils résident hors de la racine publique.

### 4. De quelle manière Nginx et Apache bloquent-ils l'exécution de scripts dans les dossiers de contenu?
Nginx recourt à des règles de localisation basées sur des expressions régulières renvoyant le code HTTP 403 pour toute requête ciblant des fichiers `.php` dans les dossiers de contenu. Apache applique des fichiers `.htaccess` combinant la directive `<FilesMatch>` avec `Options -Indexes` afin de proscrire tant l'exécution de scripts que l'affichage du contenu des dossiers.

### 5. L'utilitaire d'audit en Python engendre-t-il des ralentissements sur le site évalué?
Non. L'outil agit comme un scanner passif qui émet de simples requêtes HTTP standard GET et HEAD afin de contrôler les codes de réponse. Il vérifie si les chemins retournent HTTP 403 ou 404 plutôt qu'un état HTTP 200 ouvert, évaluant la posture périmétrique sans altérer les fichiers ni les bases de données.

***

## Cadres de Conformité et Normes de Sécurité

Architecture de sécurité conforme au Règlement Général sur la Protection des Données (RGPD, Règlement UE 2016/679), aux recommandations de la CNIL et aux guides de sécurisation de serveurs web de l'ANSSI.

***

## 4. Consultation en Ligne

Consultez le guide interactif en production :  
[https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)

***

## Liens Utiles

* **Portail Entreprise:** https://ag.tavardt.com/fr/
* **Suite de Diagnostic:** https://tools.tavardt.com/fr/
* **Contact Technique:** https://ag.tavardt.com/fr/contact/

***

© 2026 TAVARDT. Tous droits réservés.
