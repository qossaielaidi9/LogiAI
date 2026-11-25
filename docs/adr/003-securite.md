# ADR 003 — Sécurité de l’API LogiAI

- **Statut :** accepté, Atelier 4
- **Contexte :** la prédiction doit être protégée, tandis que la page web locale (port 5500) doit pouvoir appeler l’API (port 8000).

## Décision 1 — Transmettre la clé dans l’en-tête `X-API-Key`

**Choix.** Transmettre la clé dans l’en-tête HTTP `X-API-Key`, jamais dans l’URL, et ne pas la conserver dans le code ni dans Git. Les clés utilisateur et administrateur résident dans `.env`, ignoré par Git.

**Pourquoi pas l’URL ?** Les URL apparaissent dans l’historique de navigation et dans les journaux, ce qui augmente le risque de divulgation. Le corps JSON sert aux données métier ; mélanger les clés avec les données de livraison brouillerait le contrat et ne convient pas notamment aux requêtes GET. L’en-tête est standard pour une information d’authentification, même s’il reste visible aux utilisateurs de leur propre navigateur (F12).

## Décision 2 — Autoriser une liste précise d’origines CORS

**Choix.** Autoriser uniquement `http://localhost:5500` et `http://127.0.0.1:5500` en local, avec GET, POST, OPTIONS et les en-têtes `Content-Type` et `X-API-Key`.

**Pourquoi pas `*` ?** Un joker laisserait n’importe quel site lire des réponses depuis un navigateur disposant des informations nécessaires. La liste limitée répond au besoin de ce TP et réduit la surface d’exposition. CORS n’est pas une authentification : les outils hors navigateur comme curl ne suivent pas cette politique, donc la clé API reste obligatoire.

## Conséquences

- `GET /health` reste public.
- `POST /v1/predict` requiert une clé utilisateur ou administrateur.
- `GET /v1/model-info` requiert une clé administrateur ; un utilisateur authentifié reçoit 403.
- L’Atelier 5 pourra servir la page et relayer `/api/` via Nginx sur la même origine.
