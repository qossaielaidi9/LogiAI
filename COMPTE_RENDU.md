# COMPTE_RENDU — Atelier 4 : Tester, sécuriser et appeler l'API LogiAI

**Module :** Développement et déploiement d’applications intelligentes  
**Enseignante :** Pr. Soukaina Gouraguine  
**Année universitaire :** 2026–2027  
**Noms :** Salma EL mansouri & jihane tahrane
**Groupe :** 4  

> **Statut des observations :** les codes ci-dessous ont été vérifiés sur le projet corrigé au moyen de `pytest` et `TestClient` dans l'environnement de préparation. Les manipulations dans le navigateur, l'état Git et le lien de pull request doivent être confirmés sur votre ordinateur avant la remise. Ne pas confondre « prévu » et « observé ».

## 1. Activité 1 — Casser son API

| Cas | Requête (corps ou URL) | Prévu | Vérifié avec TestClient |
|---|---|---:|---:|
| Champ manquant | `POST /v1/predict`, sans `poids_kg` | 422 | 422 |
| Valeur absurde | `poids_kg = 99999` | 422 | 422 |
| Texte à la place d'un nombre | `distance_km = "loin"` | 422 | 422 |
| Champ en trop | Champ `couleur = "rouge"` | 422 | 422 |
| Mauvaise route | `POST /v1/prediction` | 404 | 404 |

Ces tests se trouvent dans `tests/test_casser.py` avec `@pytest.mark.parametrize`. L'API refuse les valeurs hors schéma grâce à Pydantic et les routes inexistantes avec 404. Aucun des cinq cas ne doit être accepté après correction.

## 2. Activité 2 — Les deux bugs des bornes

| Bug du starter | Test qui le révèle | Avant la correction | Correction |
|---|---|---|---|
| `heure_depart` acceptait 24 | `heure_depart = 24` | 200, alors qu'une heure 24 n'est pas admise | `Field(ge=0, le=23)` dans `logiai/api/schemas.py` ; désormais 422 |
| `nb_arrets` n'avait pas de minimum | `nb_arrets = -1` | 200, bien que le nombre d'arrêts ne puisse pas être négatif | `Field(ge=0, le=15)` ; désormais 422 |

Pour chaque nombre, les tests vérifient les deux côtés des bornes, ainsi que les entiers requis. Pour `type_vehicule`, les valeurs `moto`, `fourgon`, `camion` sont acceptées et `avion` est refusé.

**Résultat de la suite de tests exécutée sur la version corrigée :**

```text
54 passed in 1.10s
```

**À reproduire dans PowerShell :** `python -m pytest -v` après installation et entraînement.

## 3. Sprint 1 — Sécuriser l'API

### `verify_api_key()`

Le code réalisé figure dans `logiai/api/security.py` :

```python
def verify_api_key(x_api_key: str | None = Header(default=None)) -> str:
    if x_api_key is None or not x_api_key:
        raise HTTPException(
            status_code=401,
            detail="Clé API absente",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    if _egal(x_api_key, config.ADMIN_KEY):
        return "admin"
    if _egal(x_api_key, config.API_KEY):
        return "user"
    raise HTTPException(
        status_code=401,
        detail="Clé API invalide",
        headers={"WWW-Authenticate": "ApiKey"},
    )
```

`require_admin()` renvoie une erreur 403 pour un utilisateur authentifié sans les droits administrateur. `_egal` utilise `secrets.compare_digest` et renvoie faux si une clé n'est pas configurée. `/health` n'exige aucune clé.

| Requête | Code vérifié en test |
|---|---:|
| `POST /v1/predict` sans clé | 401 |
| `POST /v1/predict` avec fausse clé | 401 |
| `POST /v1/predict` avec clé utilisateur | 200 |
| `GET /v1/model-info` avec clé utilisateur | 403 |
| `GET /v1/model-info` avec clé administrateur | 200 |
| `GET /health` sans clé | 200 |

**Secrets :** `.env` est ignoré par `.gitignore` ; seul `.env.example` est conservé dans le dépôt. Les tests utilisent des clés fictives définies dans `tests/conftest.py`, indépendamment des clés locales. Les codes de la table doivent également être vérifiés avec les clés choisies dans le fichier `.env` de votre PC.

**Preuve à ajouter avant le rendu :** résultat de `git status --short` montrant que `.env` n'est pas suivi par Git, sans jamais révéler son contenu.

## 4. Sprint 2 — Le client web et CORS

La page `web/index.html` fait l'appel suivant :

```javascript
const reponse = await fetch(API_URL, {
  method: "POST",
  headers: { "Content-Type": "application/json", "X-API-Key": cle },
  body: JSON.stringify(livraison),
});
const corps = await reponse.json();
return { status: reponse.status, corps };
```

Le bouton est désactivé pendant l'appel et réactivé dans `finally`. Les erreurs Pydantic 422 sont rendues lisibles ; en cas d'échec réseau ou CORS, la page indique une impossibilité de joindre l'API.

| Scénario sur la page | Résultat prévu ; à confirmer sur le PC |
|---|---|
| Livraison normale et vraie clé | Probabilité de retard en pourcentage avec alerte éventuelle |
| `pluie_mm = 500` et vraie clé | Erreur de validation 422 ; attention, le navigateur peut bloquer la soumission selon les contraintes HTML |
| Fausse clé | Message « Clé API absente ou invalide » (401) |
| API arrêtée | Message « API injoignable ou appel bloqué par CORS » |

### Expérience CORS

- **Page depuis `http://localhost:5500`** : origine prévue comme autorisée dans `logiai/config.py`; le navigateur peut appeler l'API avec l'en-tête `X-API-Key`.
- **Page depuis `http://localhost:5501`** : origine non autorisée ; la requête de pré-vérification OPTIONS n'obtient pas l'autorisation CORS et le navigateur bloque la lecture de la réponse.
- **Pourquoi `curl.exe` continue de fonctionner ?** CORS est une restriction appliquée par les navigateurs, pas une vérification d'identité côté serveur. L'API exige toujours une vraie clé.
- **Où se trouve la clé ?** Dans F12 → Network → requête `/v1/predict` → Request Headers → `X-API-Key`. Elle ne doit pas être écrite en dur dans le JavaScript.

**À vérifier sur Windows :** insérer ici la capture F12 (en masquant la clé), le message exact de la console à 5501 et la réponse observée par la page. Les scénarios ci-dessus sont des attentes déduites de la configuration, pas des captures de navigateur réalisées.

## 5. Cinq questions de compréhension

**1. Différence entre 401, 403 et 422 ?**  
401 : clé absente ou erronée ; 403 : identité connue mais accès interdit (utilisateur sur une route admin) ; 422 : corps JSON reçu mais données invalides. L'authentification intervient avant la validation ; une pluie de 500 mm sans clé conduit donc d'abord à 401.

**2. Pourquoi la clé n'est-elle ni dans le code ni dans Git ?**  
Parce que toute personne ayant accès au dépôt ou à son historique pourrait la récupérer. `.env` reste local et est exclu de Git. Il ne faut pas inclure les clés dans les captures d'écran.

**3. Pourquoi `compare_digest` plutôt que `==` ?**  
`secrets.compare_digest` réduit les différences de temps d'exécution liées au contenu des clés, ce qui diminue le risque d'inférence par mesure des temps de comparaison.

**4. Pourquoi `curl.exe` fonctionne-t-il alors que la page du port 5501 est bloquée ?**  
Le navigateur fait respecter CORS ; `curl.exe` n'applique pas cette restriction. Avec la bonne clé, curl peut appeler directement l'API.

**5. Pourquoi la validation du navigateur ne remplace-t-elle pas celle de l'API ?**  
N'importe quel client peut envoyer directement du JSON, sans passer par la page. Seule la validation côté serveur garantit que les règles métier et les types sont appliqués à toutes les requêtes.

## 6. Décision et livraison

**ADR 003 :** voir `docs/adr/003-securite.md` pour les choix des en-têtes d'authentification et des origines CORS autorisées.

**Branche :** `tp-securite`  
**Pull request :** [insérer le vrai lien GitHub]  
**Commentaire du relecteur :** [à ajouter]  
**Tag visé :** `tp4` (l'en-tête de l'énoncé indique `tp4`, mais la section finale mentionne `tp2` ; vérifier l'exigence du professeur).

**Checklist avant remise :** reproduire `pytest` et `ruff check .`, confirmer `git status`, réaliser les captures Swagger et CORS, créer puis faire fusionner la pull request et poser le tag correct. Ne pas inventer de résultats non constatés.

## 7. Résultats expérimentaux sur Windows — 11/11/2026

### Vérification de la qualité
- Pytest : 54 tests réussis, 1 avertissement Starlette.
- Ruff check : All checks passed!
- Ruff format : 19 files already formatted.

### Tests de sécurité
| Requête | Code obtenu |
|---|---|
| Prédiction sans clé | 401 |
| Prédiction avec fausse clé | 401 |
| Informations du modèle avec clé utilisateur | 403 |
| Informations du modèle avec clé administrateur | 200 |
| Prédiction avec clé utilisateur valide | 200 |
| Healthcheck public | 200 |

### Validation des données
Une livraison avec pluie_mm = 500 a renvoyé HTTP 422.
La limite maximale autorisée est 100 mm.

### Prédiction de livraison
Données : 120 km, 450 kg, départ 17 h, pluie 8 mm,
trafic 7.5, 4 arrêts, véhicule camion.

Probabilité de retard : 73,2 %.
Seuil : 40 %.
Alerte : livraison à risque.

### Expérience CORS
Sur localhost:5500 : prédiction réussie.
Sur localhost:5501 : appel bloqué par CORS.
La console du navigateur confirme l'absence
de l'autorisation Access-Control-Allow-Origin.

La page affiche :
"API injoignable ou appel bloqué par CORS."

Conclusion : les origines CORS sont correctement limitées.
L'authentification reste nécessaire indépendamment de CORS.

