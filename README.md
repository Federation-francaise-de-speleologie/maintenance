# Page de maintenance FFS

Page statique affichée pendant les périodes de maintenance du site de la Fédération Française de
Spéléologie. Hébergée sur **GitHub Pages**, indépendamment de l'infrastructure applicative, afin de rester
disponible même si les serveurs applicatifs (Apache/PHP/DB) sont hors service.

- Page : [index.html](index.html)
- Logo : [assets/ffs-logo.svg](assets/ffs-logo.svg) (logo officiel FFS, blanc/vert sur fond sombre)
- URL une fois GitHub Pages activé : `https://federation-francaise-de-speleologie.github.io/maintenance/`

## Mettre un site Apache en maintenance (proxy vers cette page)

Objectif : faire servir **tout le trafic** du site par la page de maintenance hébergée sur GitHub Pages, sans
rediriger le navigateur (l'URL affichée reste celle du site, ex. `https://monsite.fr/`) et sans toucher au
code applicatif ni à la configuration existante (on pourra tout désactiver d'un coup une fois la maintenance
terminée).

On utilise `mod_proxy` (reverse proxy) plutôt qu'une redirection `mod_rewrite` classique (`R=permanent`) :
Apache va chercher la page sur GitHub Pages côté serveur et la renvoie telle quelle au visiteur, qui ne voit
jamais l'URL `github.io` ni de saut de page.

⚠️ `ProxyPass / https://.../maintenance/` seul ne suffit pas : Apache **concatène** le chemin demandé à
l'URL cible, donc `/test/` serait proxifié vers `.../maintenance/test/`, qui n'existe pas sur GitHub Pages
(404). Il faut forcer *toutes* les requêtes vers la même URL fixe avec `mod_rewrite` et le drapeau `[P]`
(proxy) — voir les exemples ci-dessous.

### Prérequis — activer les modules Apache nécessaires

```bash
a2enmod proxy proxy_http ssl
apachectl configtest && systemctl reload apache2
```

⚠️ Sur certains builds, la directive `ProxySSLVerify` n'existe pas dans `mod_proxy` (erreur `Invalid
command 'ProxySSLVerify'` au `configtest` même avec `proxy`/`proxy_http` chargés) — dans ce cas, retirez-la
simplement, la vérification du certificat backend est déjà activée par défaut. En revanche
`SSLProxyEngine On` (fournie par `mod_ssl`) est indispensable dès que la cible du `ProxyPass` est en
`https://` : sans elle, Apache renvoie une erreur 500 (`AH01961: failed to enable ssl support`).

### Option recommandée — `.htaccess` à la racine du site

Cette méthode ne nécessite pas de recharger Apache : on dépose (ou renomme) un fichier à la racine du
webroot, ce qui permet à n'importe qui ayant accès au serveur (SSH/FTP) de l'activer ou de le désactiver très
rapidement. Elle suppose que `mod_proxy`/`mod_proxy_http` sont déjà chargés globalement (prérequis
ci-dessus) — `ProxyPass` n'est pas autorisé dans un contexte `Directory`/`.htaccess` avant Apache 2.4.24+
mais fonctionne dans un contexte serveur/virtualhost ; si le `.htaccess` ne suffit pas sur votre version
d'Apache, utilisez directement l'alternative VirtualHost ci-dessous.

1. Sauvegarder le `.htaccess` existant :

   ```bash
   mv /var/www/monsite/.htaccess /var/www/monsite/.htaccess.bak
   ```

2. Créer un `.htaccess` de maintenance à la racine du webroot :

   ```apacheconf
   # --- MAINTENANCE : proxy vers la page de maintenance GitHub Pages ---
   # Pour désactiver la maintenance, supprimer/commenter les lignes ci-dessous
   # (le contenu applicatif normal redevient actif tel quel — rien d'autre à changer).
   SSLProxyEngine On
   ProxyPreserveHost Off

   # Sert le contenu de GitHub Pages sous l'URL du site, sans changer l'URL
   # visible par le visiteur (pas de redirection) et quel que soit le chemin demandé
   # (RewriteRule ^ ... [P] force TOUJOURS la même URL cible, contrairement à
   # ProxyPass qui concatènerait le chemin d'origine et casserait sur /xxx/).
   RewriteEngine On
   RewriteRule ^ https://federation-francaise-de-speleologie.github.io/maintenance/ [P,L]
   ProxyPassReverse / https://federation-francaise-de-speleologie.github.io/maintenance/
   # --- / MAINTENANCE ---
   ```

3. Tester :

   ```bash
   curl -I https://monsite.fr/n-importe-quelle-page
   # doit repondre 200 (contenu de la page de maintenance), sans en-tete Location
   ```

4. Pour désactiver la maintenance, il suffit de restaurer l'ancien fichier :

   ```bash
   mv /var/www/monsite/.htaccess.bak /var/www/monsite/.htaccess
   ```

### Alternative — directement dans le VirtualHost

Méthode la plus fiable, indépendante de la version d'Apache/de `AllowOverride` :

```apacheconf
<VirtualHost *:80>
    ServerName monsite.fr

    # --- MAINTENANCE : proxy vers la page de maintenance GitHub Pages ---
    # Pour désactiver la maintenance, supprimer/commenter les lignes ci-dessous
    # (le contenu applicatif normal, servi via DocumentRoot, redevient actif tel
    # quel — rien d'autre à changer).
    SSLProxyEngine On
    ProxyPreserveHost Off
    RewriteEngine On
    RewriteRule ^ https://federation-francaise-de-speleologie.github.io/maintenance/ [P,L]
    ProxyPassReverse / https://federation-francaise-de-speleologie.github.io/maintenance/
    # --- / MAINTENANCE ---
</VirtualHost>
```

Si le site tourne en HTTPS (`<VirtualHost *:443>` avec `SSLEngine on`), ajoutez ces directives dans **ce**
bloc (celui réellement servi), pas dans un éventuel bloc `:80` qui ne fait que rediriger vers `https` — le
reste de la configuration (`DocumentRoot`, `Directory`, logs...) peut rester tel quel, il devient simplement
inutilisé tant que le proxy est actif.

Puis recharger Apache :

```bash
apachectl configtest && systemctl reload apache2
```

Pour désactiver : commenter/supprimer les directives `Proxy*` et recharger à nouveau.

### Pourquoi un proxy plutôt qu'une redirection ou un fichier `maintenance.html` local

- L'URL du site ne change pas pour le visiteur : pas de redirection visible, pas de dépendance à ce que le
  navigateur suive un `Location`, meilleure expérience (favoris, partages de liens inchangés).
- La page reste servie même si le serveur applicatif (PHP, base de données...) est complètement down —
  c'est justement le scénario le plus probable en cas de maintenance lourde.
- Un seul endroit à mettre à jour (ce repo) pour tous les sites/environnements FFS qui pointeraient vers la
  même page de maintenance.
- Pas de dépendance à `mod_php`/à l'application pour afficher la page.

## Développement local

Le fichier `index.html` est autonome (pas de build, pas de dépendances) : ouvrir directement dans un
navigateur, ou servir le dossier avec n'importe quel serveur statique (`php -S localhost:8000`, `npx serve`,
etc.).
