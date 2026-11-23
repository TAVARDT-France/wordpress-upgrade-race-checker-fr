# Auditeur de Race Condition lors de la Mise à Jour WordPress & RCE | TAVARDT France

> Audit de Résilience d'Entreprise contre les Failles de Race Condition lors des Mises à Jour WordPress, l'Exposition des Dossiers Transitoires et l'Exécution de Code à Distance (RCE)

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Active-brightgreen)](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Security: CIS WordPress](https://img.shields.io/badge/Security-CIS%20WordPress%20%7C%20OWASP-red.svg)](https://owasp.org/)

## Vue d'Ensemble de la Vulnérabilité et Architecture Défensive

Le processus standard de mise à jour des composants dans WordPress (cœur, thèmes et extensions) orchestré par `class-wp-upgrader.php` télécharge les archives compressées au format ZIP et les décompresse dans le dossier transitoire `wp-content/upgrade/` au moyen de la fonction `unzip_file()`.

Durant l'intervalle temporel entre l'extraction des fichiers sur le disque et le déplacement atomique vers les répertoires finaux, une fenêtre de concurrence critique (Race Condition) s'ouvre dans la racine web publique. Si le serveur web ne dispose pas de règles strictes de restriction périmétrique ou de filtrage WAF, toute requête externe anonyme peut accéder directement aux fichiers extraits dans `wp-content/upgrade/`. Si l'interpréteur PHP (FastCGI/PHP-FPM) autorise la compilation de scripts dans ce dossier, des attaquants peuvent exécuter du code malveillant arbitraire (Remote Code Execution) avec les privilèges du serveur web.

Ce dépôt met à disposition un outil web interactif de diagnostic périmétrique et un utilitaire CLI en Python sans dépendance externe pour les audits automatisés.

## Application Web Interactive

Accédez à la version hébergée directement sur GitHub Pages :
[https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/](https://tavardt-france.github.io/wordpress-upgrade-race-checker-fr/)

Fonctionnalités de l'application web :
* Vérification en temps réel de la connectivité et mesure de la latence réseau.
* Détection d'exposition du dossier transitoire `wp-content/upgrade/` et de l'indexation de répertoire.
* Matrice comparative d'architecture (Topologie vulnérable vs Serveur sécurisé).
* Générateur automatisé de configurations défensives pour Nginx, Apache (.htaccess), wp-config.php et Cloudflare WAF.
* Interface réactive conçue pour terminaux mobiles et ordinateurs de bureau.

## Utilisation du Scanner CLI en Python

Le script `wordpress_upgrade_checker.py` ne requiert aucune bibliothèque tierce et fonctionne sous tout environnement Python 3.8+ :

```bash
# Lancer un audit direct sur un domaine
python wordpress_upgrade_checker.py https://site-entreprise.fr

# Lancer un audit au format JSON pour les pipelines CI/CD et DevSecOps
python wordpress_upgrade_checker.py https://site-entreprise.fr --json

# Afficher les blocs de configuration pour la remédiation et le durcissement
python wordpress_upgrade_checker.py --remediate
```

## Contrôles Architecturaux et Mesures de Durcissement

Pour neutraliser définitivement la surface d'attaque liée aux conditions de concurrence et au RCE sous WordPress, appliquez une stratégie de défense en profondeur sur 4 niveaux :

1. **Isolation des Fichiers Temporaires (wp-config.php) :**
   Configurez la constante `WP_TEMP_DIR` pour pointer vers un dossier externe hors de la racine publique `DocumentRoot` :
   ```php
   define('WP_TEMP_DIR', dirname(__DIR__) . '/tmp/');
   define('DISALLOW_FILE_MODS', true);
   define('DISALLOW_FILE_EDIT', true);
   ```

2. **Blocage Périmétrique sous Nginx (nginx.conf) :**
   Rejetez les requêtes HTTP ciblant le dossier d'upgrade et interdisez la compilation des scripts `.php` dans les répertoires de contenu :
   ```nginx
   location ~* /wp-content/upgrade/ {
       deny all;
       return 403;
   }
   location ~* ^/wp-content/(uploads|upgrade)/.*\.php$ {
       deny all;
       return 403;
   }
   ```

3. **Protection sous Apache (.htaccess) :**
   Définissez des règles strictes dans `wp-content/upgrade/.htaccess` :
   ```apache
   <FilesMatch "\.(php|phtml|php[0-9]|sh|cgi)$">
       Require all denied
   </FilesMatch>
   Options -Indexes
   ```

4. **Règle Personnalisée Cloudflare WAF :**
   Créez une règle de blocage rejetant toute requête contenant `(http.request.uri.path contains "/wp-content/upgrade/")`.

## Portails Officiels

* **Portail d'Entreprise TAVARDT France :** [https://ag.tavardt.com/fr/](https://ag.tavardt.com/fr/)
* **Services de Cybersécurité :** [https://ag.tavardt.com/fr/cybersecurite/](https://ag.tavardt.com/fr/cybersecurite/)
* **Hub d'Outils Défensifs :** [https://tools.tavardt.com/fr/](https://tools.tavardt.com/fr/)
* **Organisation GitHub :** [https://github.com/TAVARDT-France](https://github.com/TAVARDT-France)

## Licence

Distribué sous licence MIT. Consultez le fichier `LICENSE` pour plus de précisions.
