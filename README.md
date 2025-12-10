![cloud-automation-and-database.webp](img/cloud-automation-and-database.webp)

# Guide complet : Installer n8n gratuitement avec Render & Supabase

Ce tutoriel vous guidera dans l'installation de n8n, un puissant outil d'automatisation open-source, en utilisant uniquement des services gratuits. Nous allons combiner Render.com pour héberger l'application n8n et Supabase pour la base de données PostgreSQL. Cette configuration vous permettra d'avoir une instance n8n fonctionnelle sans dépenser un centime ni frais mensuels.

Avant de commencer, comprenez que n8n a besoin d'une base de données pour stocker vos workflows, vos identifiants et l'historique d'exécution. C'est pourquoi nous utilisons PostgreSQL plutôt que le stockage SQLite par défaut, qui serait effacé à chaque redémarrage sur Render.

**Limitations du plan gratuit :**
*   **Render** offre 750 heures gratuites par mois avec 0.1 CPU et 512 MB (RAM). Si vous laissez votre instance n8n tourner en permanence, cela représente environ 31 jours (744 heures), ce qui est juste suffisant.
*   **Render** se met en veille après 15 minutes d'inactivité (délai de ~30 à 60s au redémarrage).
*   **Supabase** limite votre base de données à 500 Mo de stockage et 2 Go de bande passante par mois sur le plan gratuit. Pour un usage personnel de n8n, ces limites sont généralement plus que suffisantes, à moins que vous ne stockiez d'énormes quantités de données dans vos workflows.
*   **Supabase** met la base de données se met en pause après 1 semaine d'inactivité (il faut se connecter au dashboard Supabase pour la réactiver).

---

## Prérequis

Vous aurez besoin de créer deux comptes gratuits. Prenez quelques minutes pour vous inscrire si ce n'est pas déjà fait :

1.  Un compte [Render.com](https://render.com/) :
    Render propose un tier gratuit qui inclut 750 heures de calcul par mois, parfait pour héberger n8n. L'inscription peut également se faire via GitHub.
2.  Un compte [Supabase.com](https://supabase.com/) :
    Supabase offre une base de données PostgreSQL gratuite avec 500 Mo de stockage, ce qui est largement suffisant pour commencer avec n8n. Vous pouvez vous inscrire avec votre compte GitHub pour simplifier le processus.
3.  Un générateur de mot de passe (ou terminal) pour créer une clé de chiffrement.

---

## Étape 1 : Créer votre base de données PostgreSQL sur Supabase

n8n a besoin d'une base de données pour sauvegarder vos workflows, votre historique et vos identifiants. Nous allons utiliser PostgreSQL via Supabase.

1.  Connectez-vous à **Supabase** et cliquez sur **"New Project"**.
2.  Choisissez votre organisation, donnez un **Name** (ex: `n8n-db`) et définissez un **Database Password** (⚠️ **Notez-le bien**, vous ne pourrez plus le voir).
3.  Choisissez la région la plus proche de vous (ex: `Frankfurt` ou `London`).
4.  Cliquez sur **"Create new project"**.
5.  Attendez quelques minutes que la base de données soit prête.
6.  Une fois prête, allez dans votre projet `n8n-db` et dans la barre supérieur cliquez sur le bouton **Connect**.
![Supabase-connect.png](img/supabase-connect.png)
7.  Vous trouverez dans cette modale une section appelée `Connection string`. C'est exactement ce dont nous avons besoin. Cherchez la chaîne de connexion intitulée **URI** ou **Direct connection**. Vous verrez quelque chose qui ressemble à ceci :
![Supabase-uri.png](img/supabase-uri.png)
```
postgresql://postgres:[YOUR-PASSWORD]@db.xxxxxxxxxxxxx.supabase.co:5432/postgres
```
Cliquez sur le bouton pour copier cette URL. Notez bien que dans cette chaîne, il y a un placeholder `[YOUR-PASSWORD]` que vous devrez remplacer par le mot de passe que vous avez créé à l'étape précédente. Gardez cette URL modifiée à portée de main, elle sera essentielle pour la prochaine étape.

Si vous cliquez sur **View parameters**, vous devriez voir quelquechose comme cela :

    *   **Host** (ex: `db.xxxxxxxx.supabase.co`)
    *   **User** (généralement `postgres`)
    *   **Port** (généralement `5432`)
    *   **Database Name** (généralement `postgres`)

---

## Étape 2 : Créer votre Web Service sur Render

Nous allons maintenant déployer l'image Docker officielle de n8n sur Render.

Vous avez deux options pour déployer n8n. La méthode **Blueprint (recommandée)** est plus simple et automatise la configuration des variables.

### Option A : Méthode Simplifiée (Blueprint)

Cette méthode utilise un fichier de configuration "Infrastructure as Code" pour dire à Render exactement comment configurer le serveur.

1. Utilisez le fichier nommé `render-n8n-supabase.yaml` dans le dépôt GitHub ou en cliquez directement sur le bouton suivant :

[![Deployer n8n sur Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy)

*(Si le bouton ne fonctionne pas directement, allez sur Render, faites "New > Blueprint", et sélectionnez le dépôt GitHub).*

2. **Remplir les informations** : Render va détecter le fichier et vous demander de remplir les 3 valeurs manquantes à l'écran :

- DB_POSTGRESDB_HOST : L'hôte récupéré à l'étape 1 sur Supabase.

- DB_POSTGRESDB_PASSWORD : Le mot de passe de votre base de données.

- WEBHOOK_URL : Pour l'instant, mettez n'importe quoi (ex: TBD), nous le corrigerons après.

Cliquez sur Apply. Render va démarrer le déploiement. Passez directement à l'étape 4.

### Option B : Méthode Manuelle (Avancée)

1.  Connectez-vous à **Render**.
2.  Cliquez sur le bouton **"New +"** et sélectionnez **"Web Service"**.
3.  Dans la section "Deploy an existing image...", cochez **"Image URL"**.
4.  Dans le champ "Image URL", collez ceci :
    ```text
    docker.io/n8nio/n8n:latest
    ```
5.  Cliquez sur **"Next"**.
6.  Configurez les détails du service :
    *   **Name :** `mon-n8n` (ou ce que vous voulez). Ce nom fera partie de votre URL, donc choisissez quelque chose de simple et mémorable. Render générera automatiquement une URL comme `https://mon-n8n.onrender.com`.
    *   **Region :** Essayez de choisir la même région que Supabase (ex: Frankfurt).
    *   **Instance Type :** Sélectionnez **Free**.
7.  Cliquez sur le bouton **"Advanced"** pour préparer l'ajout des variables d'environnement (nous le ferons à l'étape suivante).

---

## Étape 3 : Configurer les Variables d'Environnement

C'est l'étape la plus critique. Sans cela, n8n ne saura pas comment parler à Supabase ni comment sécuriser vos données.

Dans la section **Environment Variables** sur Render, cliquez sur **"Add Environment Variable"** et ajoutez les clés/valeurs suivantes :

### A. Configuration de base
| Key                  | Value               | Description                             |
| :------------------- | :------------------ | :-------------------------------------- |
| `N8N_PORT`           | `5678`              | Render écoute sur ce port par défaut    |
| `N8N_PROTOCOL`       | `https`             | Pour avoir une connexion sécurisée      |
| `GENERIC_TIMEZONE`   | `Europe/Paris`      | Pour avoir la bonne heure dans vos logs |
| `N8N_ENCRYPTION_KEY` | *(voir ci-dessous)* | **TRÈS IMPORTANT**                      |

> **⚠️ Générer la clé de chiffrement (N8N_ENCRYPTION_KEY) :**
> Générez une chaîne aléatoire complexe (alphanumérique).
> *Exemple :* `v0treCleSecreteSuperComplexe12345` avec au moins 32 caractères. Cette clé sert à chiffrer les données sensibles dans votre base de données.
> **Sauvegardez cette clé quelque part en sécurité.** Si vous perdez cette clé et que votre instance redémarre, vous perdrez l'accès à tous vos identifiants (Google Drive, Slack, etc.) enregistrés dans n8n.

### B. Connexion à Supabase (PostgreSQL)
Utilisez les infos notées à l'Étape 1.

| Key                      | Value                                                    |
| :----------------------- | :------------------------------------------------------- |
| `DB_TYPE`                | `postgresdb`                                             |
| `DB_POSTGRESDB_HOST`     | *Votre Host Supabase* (ex: `db.RefDuProjet.supabase.co`) |
| `DB_POSTGRESDB_PORT`     | `5432`                                                   |
| `DB_POSTGRESDB_DATABASE` | `postgres`                                               |
| `DB_POSTGRESDB_USER`     | `postgres`                                               |
| `DB_POSTGRESDB_PASSWORD` | *Le mot de passe créé à l'étape 1*                       |
| `DB_POSTGRESDB_SCHEMA`   | `public`                                                 |

### C. Configuration Webhook (Optionnel mais recommandé)
Pour que les webhooks fonctionnent correctement, n8n doit connaître sa propre URL. Comme nous n'avons pas encore l'URL finale, vous pouvez revenir ici après le déploiement, ou deviner l'URL qui sera souvent : `https://<nom-de-votre-service>.onrender.com`.

| Key           | Value                                     |
| :------------ | :---------------------------------------- |
| `WEBHOOK_URL` | `https://votre-nom-service.onrender.com/` |

(n'oubliez pas le `/` à la fin, ex: https://votre-nom-service.onrender.com/)

Voici un récapitulatif complet des variables à configurer :

```
N8N_PORT=5678
N8N_PROTOCOL=https
GENERIC_TIMEZONE=Europe/Paris
N8N_ENCRYPTION_KEY=votre-clé-aléatoire-de-32-caractères
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=db.xxxxxxxxxxxxx.supabase.co
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=postgres
DB_POSTGRESDB_USER=postgres
DB_POSTGRESDB_PASSWORD=votre-mot-de-passe-supabase
DB_POSTGRESDB_SCHEMA=public
WEBHOOK_URL=https://votre-nom-service.onrender.com
```

---

## Étape 4 : Déploiement

1.  Une fois toutes les variables ajoutées, cliquez sur **"Create Web Service"** en bas de page.
2.  Render va commencer à télécharger l'image Docker et lancer n8n.
3.  Regardez les **Logs**. Le premier démarrage peut prendre 3 à 5 minutes car n8n doit créer les tables dans votre base de données Supabase.
4.  Attendez de voir le message : `Editor is now accessible via: http://localhost:5678/` (dans les logs Render, cela signifie que c'est prêt).
5.  Si vous êtes passé par l'option d'installation A, rendez-vous sur **Render** dans la section environnement pour renseigner la variable `WEBHOOK_URL` avec votre URL de déploiement finale.

---

## Étape 5 : Premier Lancement

1.  En haut à gauche de votre dashboard Render, cliquez sur le lien de votre application (ex: `https://mon-n8n.onrender.com`).
2.  **Patience :** Le premier chargement peut être long.
3.  Une page s'affiche pour **créer votre compte propriétaire** (Owner account).
4.  Remplissez vos infos (Email, Nom, Mot de passe).
5.  C'est terminé ! Vous êtes sur votre interface n8n.

---

## Étape 6 : Vérifier que tout fonctionne

Pour vous assurer que votre installation est correctement configurée, créez un workflow de test simple. Dans l'interface n8n, créez un nouveau workflow et ajoutez un nœud "Schedule Trigger" qui s'exécute toutes les heures. Ajoutez ensuite un nœud "Set" pour créer une donnée simple.

Activez le workflow et vérifiez dans l'onglet "Executions" que les exécutions sont bien enregistrées. Si vous voyez l'historique s'accumuler, cela signifie que la connexion à PostgreSQL fonctionne parfaitement et que vos données sont persistantes.

---

## Conseils de sécurité et bonnes pratiques

Maintenant que votre instance fonctionne, pensez à sécuriser correctement votre installation. Ne partagez jamais votre URL n8n publiquement, car même avec authentification, c'est votre porte d'entrée vers vos automatisations.

Utilisez des mots de passe forts et uniques pour votre compte n8n. Pensez à activer l'authentification à deux facteurs si n8n le propose dans les versions futures.

Faites régulièrement des exports de vos workflows importants. Vous pouvez le faire depuis l'interface n8n en téléchargeant vos workflows au format JSON (ou mieux, en automatisant la sauvegarde automatique de vos workflows avec n8n vers un espace de stockage externe). C'est une bonne sauvegarde en cas de problème avec votre/vos instance(s).

---

## Astuces et Maintenance

### 1 - Le problème du "Cold Start" (Démarrage à froid)
Comme vous êtes sur le plan gratuit de Render, si personne n'utilise n8n pendant 15 minutes, le serveur s'éteint.
*   **Conséquence :** Si un webhook est appelé alors que le serveur est éteint, il mettra environ 30 à 60 secondes à répondre le temps que Render redémarre.
*   **Solution :** Pour éviter cela, vous pouvez utiliser un service gratuit comme [UptimeRobot](https://uptimerobot.com/?rid=7e7eefb05d096f) pour "pinger" votre URL n8n toutes les 5 minutes, ce qui gardera le service actif.

### 2 - Mise à jour de n8n
Pour mettre à jour n8n vers une version plus récente :
1.  Allez sur le dashboard Render.
2.  Allez dans votre Web Service.
![n8n.png](img/n8n.png)
3.  Cliquez sur **"Manual Deploy"** > **"Deploy latest reference"**.
Render va re-télécharger l'image `latest` et redémarrer.

### 3 - Sauvegarde de la clé
Je le répète : **Sauvegardez votre `N8N_ENCRYPTION_KEY`**. Sur Render gratuit, le disque n'est pas persistant. Si n8n redémarre, il va chercher cette clé dans les variables d'environnement pour décrypter vos accès dans la base de données Supabase. Si elle change ou manque, vos workflows seront corrompus.

### 4 - Supabase "Paused"
Si vous n'utilisez pas votre n8n pendant une semaine, Supabase mettra la base de données en pause. n8n ne fonctionnera plus (Erreur de connexion DB).
*   **Solution :** Connectez-vous simplement à votre compte Supabase pour réactiver le projet ("Restore Project").

### 5 - n8n ne démarre pas
Si n8n ne démarre pas, la première chose à vérifier est les logs dans Render. Ils vous indiqueront exactement où se situe le problème. L'erreur la plus fréquente concerne les informations de connexion à la base de données. Vérifiez que vous avez bien remplacé le placeholder du mot de passe dans votre chaîne de connexion et que toutes vos variables d'environnement sont correctement définies.

### 6 - Erreur du type **"Connection refused"**
Si vous voyez une erreur du type "Connection refused" ou "ECONNREFUSED", cela signifie généralement que les paramètres de connexion PostgreSQL sont incorrects. Retournez dans Supabase pour vérifier votre chaîne de connexion et assurez-vous que le host, le port et les identifiants correspondent exactement.

---

## This is the end
Félicitations, vous avez maintenant une instance n8n entièrement fonctionnelle et gratuite. Vous pouvez commencer à créer vos premiers workflows d'automatisation, connecter vos applications préférées et laisser n8n gérer vos tâches répétitives.

Cette configuration gratuite est parfaite pour apprendre, tester et automatiser vos tâches personnelles. Si vos besoins évoluent vers un usage professionnel intensif, vous pourrez toujours migrer vers des plans payants offrant plus de ressources et une meilleure disponibilité.