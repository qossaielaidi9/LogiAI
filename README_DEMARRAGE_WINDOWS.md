# Atelier 4 — LogiAI — Guide Windows PowerShell

Ce projet est basé sur le ZIP de l'Atelier 4 fourni avec l'énoncé de la Pr. Soukaina Gouraguine (2026-2027). Le code de l'Atelier 3 nécessaire est déjà inclus : **pas besoin de refaire l'Atelier 3**.

> **Démarrez dans ce dossier**, à la racine où se trouvent `requirements.txt`, `logiai/`, `web/`, `tests/` et `livraison.json`. N'exécutez pas les commandes dans `Downloads` si vous êtes ailleurs.

## 1. Préparation une seule fois

1. Installez Python 3.11 ou plus récent et assurez-vous que `python --version` fonctionne ; sinon utilisez `py -3.11` en remplaçant `python` dans les commandes. Ouvrez ce dossier dans VS Code, puis ouvrez un terminal **PowerShell**.
2. Tapez, **dans cet ordre** :

```powershell
python --version
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
notepad .env
```

Dans Notepad, remplacez les deux valeurs d'exemple par **deux clés différentes**, longues, de votre choix (lettres, chiffres, tirets), puis sauvegardez. Ne publiez jamais le contenu de `.env` ; il est exclu par `.gitignore`.

3. Générer les données, entraîner le modèle puis tester :

```powershell
.\.venv\Scripts\python.exe -m logiai.dataset
.\.venv\Scripts\python.exe -m logiai.train
.\.venv\Scripts\python.exe -m pytest -v
```

**Attendu sur le projet corrigé : 54 tests réussis** (le nombre exact peut changer si vous ajoutez des tests). Si `python -m venv` ou l'installation échoue, envoyez le message d'erreur complet.

## 2. Lancer l'API — Terminal PowerShell 1

```powershell
.\.venv\Scripts\python.exe -m uvicorn logiai.api.main:app --reload
```

**Laissez ce terminal ouvert.** Ouvrez dans un navigateur :

- http://localhost:8000/health — doit indiquer `model_loaded: true` et `auth_configured: true`.
- http://localhost:8000/docs — Swagger FastAPI.

## 3. Lancer la page — Terminal PowerShell 2

Dans un **deuxième** terminal, placé dans le même dossier :

```powershell
.\.venv\Scripts\python.exe -m http.server 5500 -d web
```

Ouvrez http://localhost:5500, entrez la **clé utilisateur** sauvegardée dans `.env`, puis cliquez sur **Évaluer**. Avec les valeurs proposées (120 km, 450 kg, départ 17 h, pluie 8 mm, trafic 7.5, quatre arrêts, camion), la page devrait afficher la probabilité et l'alerte éventuelle.

## 4. Tester les codes HTTP dans un troisième terminal

N'arrêtez pas les deux premiers terminaux. Dans le troisième :

```powershell
# Santé publique
curl.exe -i http://localhost:8000/health

# Sans clé : 401
curl.exe -i -X POST http://localhost:8000/v1/predict -H "Content-Type: application/json" -d "@livraison.json"

# Clé utilisateur : 200 (remplacez MA_CLE_UTILISATEUR par la vôtre)
curl.exe -i -X POST http://localhost:8000/v1/predict -H "Content-Type: application/json" -H "X-API-Key: MA_CLE_UTILISATEUR" -d "@livraison.json"

# Clé utilisateur sur la route admin : 403
curl.exe -i http://localhost:8000/v1/model-info -H "X-API-Key: MA_CLE_UTILISATEUR"

# Clé administrateur sur la route admin : 200
curl.exe -i http://localhost:8000/v1/model-info -H "X-API-Key: MA_CLE_ADMIN"
```

Pour garder les clés hors des commandes enregistrées dans le terminal, vous pouvez faire les essais avec le formulaire Swagger `/docs` et copier les captures **en masquant vos clés**.

## 5. Expérience CORS demandée

- À http://localhost:5500, les requêtes sont autorisées depuis le navigateur (avec clé valide). Ouvrez F12 → **Network** et consultez l'appel POST ainsi que la requête préliminaire OPTIONS.
- Dans le terminal 2, pressez `Ctrl+C`, puis lancez :

```powershell
.\.venv\Scripts\python.exe -m http.server 5501 -d web
```

- Ouvrez http://localhost:5501, retapez la clé utilisateur et cliquez sur **Évaluer**. La page devrait signaler une erreur réseau/CORS ; F12 → **Console** montre la raison précise. `curl.exe` n'est pas soumis à CORS et fonctionne toujours avec la clé valide.
- Arrêtez le serveur 5501 (`Ctrl+C`), puis relancez-le sur le port 5500 si nécessaire.

## 6. Qualité, compte rendu, GitHub

```powershell
.\.venv\Scripts\python.exe -m pip install ruff
.\.venv\Scripts\python.exe -m ruff check .
.\.venv\Scripts\python.exe -m pytest -v
```

N'exécutez `ruff check . --fix` **qu'après avoir terminé le code**. Complétez `COMPTE_RENDU.md` avec **vos observations locales**, les deux bugs, les essais CORS, les preuves nécessaires, vos nom/groupe et le lien réel de la pull request. Le document `docs/adr/003-securite.md` contient les justifications des deux décisions demandées.

Si ce dossier est un dépôt Git déjà configuré, branchez-vous sur `main`, créez `tp-securite` et poussez. Si ce n'est **pas encore** un dépôt Git, il faut d'abord créer et connecter votre dépôt GitHub, puis :

```powershell
git switch -c tp-securite
git status
git check-ignore .env
git add .
git commit -m "TP securite : tests, cle API, page web"
git push -u origin tp-securite
```

Créez la pull request et faites-la relire/fusionner, puis mettez le tag demandé par le professeur. **Attention : l'en-tête de l'énoncé demande `tp4` tandis que la section Rendre mentionne `tp2`. Demandez quel nom de tag utiliser avant de pousser.**

## Important avant l'Atelier 5

Ne remplacez **pas** la page de l'Atelier 4 par celle de l'Atelier 5 avant d'avoir réalisé les tests CORS. À l'Atelier 5, l'URL sera changée de `http://localhost:8000/v1/predict` vers `/api/v1/predict` avec Nginx et Docker Compose.

Cette archive **ne contient pas un fichier `.env` réel** (pour la sécurité) ni les fichiers entraînés du modèle. Les commandes de l'étape 1 les génèrent localement. Les résultats navigateur/GitHub ne peuvent être validés qu'après exécution sur votre ordinateur.
