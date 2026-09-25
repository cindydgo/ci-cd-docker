# 📝 Pipeline CI/CD – Documentation et Checklist

## 1. Objectif

Cette pipeline CI/CD automatisée permet de :

* Exécuter les tests de l’application (`check_app.sh`).
* Construire une image Docker à partir du dossier `app/`.
* Pousser l’image Docker sur Docker Hub (ou GHCR).
* Envoyer une notification Google Chat en cas de succès ou d’échec.

---

## 2. Prérequis

* Compte GitHub avec un dépôt configuré.
* Docker installé localement pour tests.
* Secrets GitHub configurés :

  * `DOCKER_USERNAME` → Nom d’utilisateur Docker Hub.
  * `DOCKER_PASSWORD` → Mot de passe Docker Hub.
  * `GOOGLE_CHAT_WEBHOOK_URL` → URL du webhook Google Chat (Incoming Webhook).

---

## 3. Checklist de validation

1. **Tests automatiques**

   * Tous les tests doivent passer via `app/check_app.sh`.
   * Vérifier que le workflow échoue si un test échoue.

2. **Image Docker**

   * Construire localement avec :

     ```bash
     docker build -t myapp:dev ./app
     docker run -p 8080:80 myapp:dev
     ```
   * Vérifier que l’application fonctionne localement.

3. **Push Docker**

   * Vérifier que l’image est bien poussée sur Docker Hub / GHCR :

     ```bash
     docker push <DOCKER_USERNAME>/myapp:dev
     ```
   * Utilisation correcte des tags pour différencier `dev` (branche main) et `prod` (tag Git).

4. **Notification Google Chat**

   * Vérifier que la notification est envoyée à chaque exécution.
   * Tester avec un message simple si la notification échoue (`curl` direct).

5. **Sécurité des secrets**

   * Aucun secret ou mot de passe en clair dans le dépôt ou le workflow.
   * Tous les secrets doivent passer par GitHub Actions.

6. **Historique des workflows**

   * Consulter l’onglet **Actions** sur GitHub pour suivre l’exécution.
   * Vérifier que les jobs se déclenchent correctement sur push et tags.

7. **Documentation interne**

   * Indiquer la procédure pour ajouter/modifier les secrets.
   * Indiquer comment tester la pipeline localement.
   * Ajouter des informations sur les erreurs fréquentes (Docker, Google Chat).

---

## 4. Commandes utiles

```bash
# Lancer les tests localement
cd app
chmod +x check_app.sh
./check_app.sh

# Construire l'image Docker
docker build -t myapp:dev ./app

# Lancer le conteneur
docker run -p 8080:80 myapp:dev

# Pousser l'image Docker
docker push <DOCKER_USERNAME>/myapp:dev

# Tester la notification Google Chat
curl -X POST -H "Content-Type: application/json" \
  -d '{"text": "Test notification"}' \
  "<GOOGLE_CHAT_WEBHOOK_URL>"
```

---

## 5. Bonnes pratiques

* Toujours tester localement avant de pousser.
* Utiliser des branches pour les tests et tags pour la production.
* Vérifier les logs de GitHub Actions pour diagnostiquer rapidement les échecs.
* Ne jamais exposer de secrets dans le dépôt.

---

# COMMENT ÇA MARCHE ?

# Déclenchement du pipeline

Le workflow est déclenché lorsqu'un `push` est effectué sur `main` ou lorsqu'un tag commençant par `v` est poussé.

```yaml
on:
  push:
    branches: [main]
    tags:
      - 'v*'
```

### Exemple avec `main`

```bash
git add .
git commit -m "..."
git push origin main
```

Le pipeline est alors automatiquement lancé.

### Exemple avec un tag

```bash
git tag v1.0.0
git push origin v1.0.0
```

Le pipeline est également déclenché.

---

# Environnement d'exécution

Le pipeline utilise :

```yaml
runs-on: ubuntu-latest
```

GitHub Actions exécute donc toutes les étapes sur un environnement Ubuntu.

Cela permet notamment d'utiliser les commandes Linux présentes dans le workflow.

---

# 1. Checkout du dépôt

La première étape récupère le contenu du dépôt :

```yaml
- name: Checkout repository
  uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

L'action `checkout` permet au runner GitHub Actions d'accéder aux fichiers du projet.

La version de l'action est référencée par un commit précis afin de conserver une version déterminée de l'action utilisée.

---

# 2. Exécution des tests

Le pipeline se place dans le dossier `app` :

```bash
cd app
```

Puis rend le script exécutable :

```bash
chmod +x check_app.sh
```

Enfin, il lance les vérifications :

```bash
./check_app.sh
```

Le script `check_app.sh` permet de vérifier le fonctionnement attendu de l'application avant de construire l'image Docker.

### Pourquoi `chmod` ?

`chmod +x` donne au script le droit d'exécution sous Linux.

Cette commande est exécutée sur le runner **Ubuntu de GitHub Actions**.

Elle n'est donc pas nécessaire dans PowerShell Windows pour le fonctionnement du pipeline GitHub.

---

# 3. Connexion à Docker Hub

Le pipeline utilise `docker/login-action` pour se connecter à Docker Hub :

```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

Les identifiants ne sont pas directement écrits dans le workflow.

Ils sont stockés dans les **Secrets GitHub**.

---

# 4. Construction de l'image Docker

L'image est construite à partir du `Dockerfile` présent dans le dossier `app` :

```bash
docker build -t $IMAGE_NAME:$TAG ./app
```

Le nom de l'image est construit à partir du nom d'utilisateur Docker Hub :

```bash
IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/myapp"
```

Par défaut, le tag utilisé est :

```text
dev
```

L'image produite est donc :

```text
mon-utilisateur/myapp:dev
```

---

# 5. Gestion des versions avec les tags Git

Le workflow vérifie si la référence Git correspond à un tag :

```bash
if [[ $GITHUB_REF == refs/tags/* ]]; then
  TAG="${GITHUB_REF#refs/tags/}"
fi
```

Si le workflow est déclenché avec :

```bash
git tag v1.0.0
git push origin v1.0.0
```

alors :

```text
GITHUB_REF = refs/tags/v1.0.0
```

Le pipeline récupère :

```text
TAG = v1.0.0
```

L'image Docker devient alors :

```text
mon-utilisateur/myapp:v1.0.0
```

Cela permet d'associer une image Docker à une version précise du projet.

---

# 6. Publication de l'image sur Docker Hub

Après sa construction, l'image est envoyée sur Docker Hub :

```bash
docker push $IMAGE_NAME:$TAG
```

Selon le déclenchement du pipeline, l'image peut donc être publiée avec :

```text
mon-utilisateur/myapp:dev
```

ou :

```text
mon-utilisateur/myapp:v1.0.0
```

---

# 7. Notification Google Chat

Une dernière étape envoie une notification dans Google Chat :

```yaml
- name: Send Google Chat Notification
  if: always()
```

L'utilisation de :

```yaml
if: always()
```

permet d'exécuter cette étape même lorsqu'une étape précédente du job a échoué.

Le pipeline récupère :

* le statut du job ;
* l'auteur du dernier commit ;
* le message du commit ;
* la référence Git utilisée.

Ces informations sont intégrées dans un message JSON.

Le message est envoyé au webhook Google Chat avec `curl`.

---

# Gestion des secrets

Les informations sensibles sont stockées dans :

**GitHub → Settings → Secrets and variables → Actions**

Les trois secrets utilisés sont :

| Secret                    | Utilisation                      |
| ------------------------- | -------------------------------- |
| `DOCKER_USERNAME`         | Nom d'utilisateur Docker Hub     |
| `DOCKER_PASSWORD`         | Token ou mot de passe Docker Hub |
| `GOOGLE_CHAT_WEBHOOK_URL` | URL du webhook Google Chat       |

Le workflow récupère les valeurs avec :

```yaml
${{ secrets.DOCKER_USERNAME }}
```

```yaml
${{ secrets.DOCKER_PASSWORD }}
```

```yaml
${{ secrets.GOOGLE_CHAT_WEBHOOK_URL }}
```

Les valeurs réelles ne doivent pas être écrites dans le code source ou dans le README.

---

# Tester le webhook Google Chat sous Windows

La commande `curl` utilisée dans GitHub Actions est une commande exécutée dans l'environnement Linux du runner.

Pour effectuer un test depuis PowerShell Windows, on peut utiliser :

```powershell
Invoke-RestMethod `
  -Uri "URL_DU_WEBHOOK" `
  -Method Post `
  -ContentType "application/json" `
  -Body '{"text":"Test notification"}'
```

L'URL réelle du webhook ne doit pas être publiée dans le dépôt.

---

# Schéma du pipeline

```text
                    git push
                       │
                       ▼
                ┌─────────────┐
                │ GitHub      │
                │ Actions     │
                └──────┬──────┘
                       │
                       ▼
                Checkout du code
                       │
                       ▼
                     Tests
                       │
                       ▼
                Login Docker Hub
                       │
                       ▼
                 Build Docker
                       │
                       ▼
                Push Docker Hub
                       │
                       ▼
              Notification Google
                    Chat
```

---

# Résultat

Le pipeline automatise le cycle de livraison de l'application.

Un `push` sur `main` permet de vérifier l'application, construire son image Docker et la publier sur Docker Hub.

L'utilisation des tags Git permet également de publier des versions identifiables de l'image Docker.

Enfin, Google Chat permet de recevoir automatiquement le résultat de l'exécution du pipeline.

Le projet met ainsi en œuvre une chaîne CI/CD basée sur :

* **GitHub Actions** pour l'automatisation ;
* **Docker** pour la conteneurisation ;
* **Docker Hub** pour le stockage des images ;
* **Google Chat** pour les notifications.
