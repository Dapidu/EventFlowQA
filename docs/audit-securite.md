# Rapport d'audit de sécurité — EventFlow

| | |
|---|---|
| **Date** | 23/11/2026 |
| **Périmètre** | Dépôt `Dapidu/EventFlowQA` (fork de `BstormBe/EventFlow`), branche `mission-lundi` : historique Git, `qa/` (tests et utilitaires), dépendances de `qa/`, `docker-compose.yml`, pipeline `.github/workflows/ci.yml` |
| **Outils** | gitleaks 8.30.1 · pip-audit 2.10.1 · bandit 1.9.4 · zizmor 1.33.1 |
| **Preuves brutes** | [`docs/mission-lundi/preuves/`](mission-lundi/preuves/) — analyse détaillée dans le [rapport de mission, section B1](mission-lundi/rapport-mission.md#b1--la-fouille-du-dépôt-audit-outillé) |

> Le secret découvert n'a été ni lu, ni testé, ni copié : toutes les sorties gitleaks sont produites avec `--redact`. Les valeurs des mots de passe ne sont pas reproduites dans ce rapport.

**Échelle de gravité**

- **Critique** : exploitable maintenant par n'importe qui, compromission d'un compte ou d'un système externe.
- **Haute** : exploitable avec peu de conditions, donne des droits ou des identifiants.
- **Moyenne** : demande des conditions particulières, ou fragilise la fiabilité des tests et de la livraison.
- **Basse** : mauvaise pratique à risque limité dans le contexte actuel.

---

## 1. Constats

16 constats, des plus graves aux moins graves.

| # | Constat | Où | Outil | Gravité | Preuve | Correction proposée |
|---|---|---|---|---|---|---|
| 1 | Secret d'API Xray (`CLIENT_SECRET`) présent dans l'historique Git d'un dépôt **public** | `qa/utils/xray_export.py:11`, ajouté dans `cc42802`, fichier supprimé dans `0a07c5c` | gitleaks | **Critique** | `gitleaks git --redact -v .` : `leaks found: 1`, `RuleID: generic-api-key` ; `gitleaks dir` : `no leaks found` ; `git cat-file -t cc42802:qa/utils/xray_export.py` → `blob` (toujours lisible) | Réponse à incident (section 2) : révoquer le secret chez Xray, puis purger l'historique et ajouter un contrôle gitleaks avant commit et en CI |
| 2 | Injection de commande dans le workflow : la saisie `seed_bugs` est insérée telle quelle dans un script shell | `.github/workflows/ci.yml:23` | zizmor | **Haute** | `error[template-injection]` … `may expand into attacker-controllable code`, confiance High | Passer la valeur par une variable d'environnement : `env: SEED_BUGS_INPUT: ${{ github.event.inputs.seed_bugs }}` puis `echo "Bugs du fil rouge : $SEED_BUGS_INPUT"` ; restreindre l'entrée à `type: boolean` |
| 3 | `qa/.env` versionné : mots de passe des comptes admin, organisateur, client et URL de base de données avec identifiants | `qa/.env` (suivi depuis `daac602`) ; `.gitignore` n'exclut que `backend/.env` et `frontend/.env` | revue manuelle (non détecté par gitleaks) | **Haute** | `git ls-files --error-unmatch qa/.env` → suivi ; variables `ADMIN_PASSWORD`, `ORGA_PASSWORD`, `CLIENT_PASSWORD`, `DB_URL` ; `gitleaks dir` : `no leaks found` | `git rm --cached qa/.env`, ajouter `qa/.env` au `.gitignore`, fournir un `qa/.env.example` sans valeurs, injecter les valeurs en CI via les *secrets* GitHub ; changer les mots de passe s'ils servent ailleurs qu'en local |
| 4 | Secret de signature JWT en clair : quiconque le connaît peut fabriquer un jeton valide, y compris administrateur | `docker-compose.yml:22` (`JWT_SECRET`), valeur par défaut aussi dans `backend/app/config.py:6` | revue manuelle (non détecté par gitleaks) | **Haute** | Ligne 22 de `docker-compose.yml` ; le backend signe les jetons avec cette valeur (`jwt_secret` dans `config.py`) | Lire `JWT_SECRET` depuis l'environnement sans valeur par défaut (échec au démarrage s'il manque), valeur aléatoire longue par environnement, stockée hors dépôt |
| 5 | Vérification des certificats TLS désactivée dans le client API des tests | `qa/utils/api_client.py:22` (`verify=False`) et `:11` (`session.verify = False`) | bandit B501 | **Haute** | `Issue: [B501:request_with_no_cert_validation]`, `Severity: High` | Supprimer `verify=False` ; pour un certificat auto-signé de recette, pointer `verify=` vers le certificat de l'autorité (`REQUESTS_CA_BUNDLE`) |
| 6 | Actions GitHub référencées par une étiquette mobile (`@v7`) et non par un hash | `.github/workflows/ci.yml:19` (`checkout`), `:26` (`setup-python`), `:63` (`upload-artifact`) | zizmor | **Haute** | 3 × `error[unpinned-uses]` … `action is not pinned to a hash`, confiance High | Épingler chaque action sur le SHA complet du commit (`actions/checkout@<sha> # v7`), mises à jour via Dependabot |
| 7 | Job CI sans bloc `permissions` : le `GITHUB_TOKEN` reçoit les droits par défaut | `.github/workflows/ci.yml:14` (job `tests`) | zizmor | **Moyenne** | `warning[excessive-permissions]` … `default permissions used due to no permissions: block` | Ajouter en tête du workflow `permissions: contents: read` |
| 8 | `requests` 2.31.0 a des vulnérabilités connues | `qa/requirements.txt:2` | pip-audit | **Moyenne** | `pip-audit -r qa/requirements.txt` : 3 avis distincts (6 lignes) — `PYSEC-2026-1873`, `-1872`, `-2275` | Épingler une version corrigée (≥ 2.33.0). **Appliqué** : commit `75ca618`, `No known vulnerabilities found` |
| 9 | 12 dépendances sur 13 sans version : installations non reproductibles, nouvelle version piégée intégrée automatiquement | `qa/requirements.txt` (toutes les lignes sauf `requests`) | revue manuelle | **Moyenne** | `pytest`, `jsonschema`, `SQLAlchemy`, `psycopg2-binary`, `python-dotenv`, `PyJWT`, `pytest-html`, `pytest-playwright`, `pytest-rerunfailures`, `selenium`, `allure-pytest`, `pytest-xdist` sans `==` | Fichier verrouillé avec versions exactes et hashes (`pip-compile --generate-hashes`), `pip-audit` en CI, mises à jour via Dependabot |
| 10 | Mot de passe de la base de données en clair | `docker-compose.yml:6` (`POSTGRES_PASSWORD`) et `:21` (`DATABASE_URL`) | revue manuelle (non détecté par gitleaks) | **Moyenne** | Lignes 6 et 21 de `docker-compose.yml` ; port 5432 publié sur la machine hôte (`ports: "5432:5432"`) | Variables `${POSTGRES_PASSWORD}` lues depuis un `.env` non versionné ; ne pas publier le port 5432 hors développement |
| 11 | Test qui ne peut jamais échouer : `try … except: pass` avale aussi l'`AssertionError` | `qa/tests/api/test_promo.py:37` (`test_disable_promo`) | bandit B110 | **Moyenne** | `Issue: [B110:try_except_pass]` ; lecture du code : l'`assert` est dans le `try` | Retirer le `try/except` : laisser l'erreur faire échouer le test |
| 12 | Requêtes HTTP sans délai d'attente : une API qui ne répond plus bloque le test et le job CI | `qa/tests/api/test_promo.py:7,14,22,33,35,42,44`, `qa/utils/api_client.py:19` | bandit B113 | **Basse** | 8 × `Issue: [B113:request_without_timeout]` | Ajouter `timeout=10` à chaque appel (ou un délai par défaut dans `ApiClient`) ; `timeout-minutes` sur le job CI |
| 13 | Identifiants des comptes admin et client écrits en dur dans un test, en double de `qa/.env` | `qa/tests/api/test_promo.py:7` et `:42` | bandit B105 | **Basse** | `Issue: [B105:hardcoded_password_string]` sur ces deux lignes | Utiliser `utils.config` et les fixtures existantes (`admin_api`, `client_api`) |
| 14 | Requête SQL construite par f-string | `qa/utils/db.py:29` (`… WHERE o.id = {order_id}`) | bandit B608 | **Basse** | `Issue: [B608:hardcoded_sql_expressions]` ; `order_id` vient de l'API (entier), d'où le risque faible | Paramètre lié : `WHERE o.id = :order_id`, comme le fait déjà `execute()` |
| 15 | Le checkout conserve le jeton GitHub dans `.git/config` | `.github/workflows/ci.yml:19` | zizmor | **Basse** | `help[artipacked]` … `does not set persist-credentials: false`, confiance Low (seul `qa/reports/` est publié en artefact) | `with: persist-credentials: false` sur `actions/checkout` |
| 16 | Fonction de test jamais exécutée (nom sans préfixe `test_`) | `qa/tests/api/test_promo.py:48` (`code_nul`) | revue manuelle | **Basse** | pytest ne collecte que les fonctions `test_*` | La supprimer ou la renommer avec une vraie assertion |

**Synthèse :** 1 critique · 5 hautes · 5 moyennes · 5 basses. Les trois premières priorités : révoquer le secret Xray (#1), corriger l'injection du workflow (#2), sortir `qa/.env` et les secrets de configuration du dépôt (#3, #4).

---

## 2. Réponse à incident : secret Xray dans l'historique

**Faits :** un secret d'API (`CLIENT_SECRET`, règle `generic-api-key`) a été commité le 20/10/2026 (`cc42802`) dans `qa/utils/xray_export.py`, puis le fichier a été supprimé (`0a07c5c`). Le dépôt est public et forké par chaque stagiaire. **Exposition : depuis le 20/10/2026, soit plus d'un mois.**

### Réponses aux questions

- **Supprimer le fichier suffit-il ? Non.** C'est exactement ce qui a été fait (`0a07c5c`) : le fichier a disparu de la version actuelle, mais chaque commit est une photo complète du dépôt. Le commit `cc42802` contient toujours le secret, lisible par n'importe qui sur GitHub.
- **Réécrire l'historique suffit-il ? Non.** `git filter-repo` retire le secret de *notre* dépôt, mais pas des copies déjà faites : les forks de tous les stagiaires, chaque clone sur un poste, les caches de GitHub (vues de commit, références de pull requests), les artefacts CI et d'éventuels robots qui aspirent les dépôts publics à la recherche de secrets. Après un mois d'exposition publique, il faut supposer que le secret a été copié.
- **Qu'est-ce qui protège vraiment ? La révocation.** Un secret révoqué ne donne plus accès à rien, quel que soit le nombre de copies qui circulent. C'est la seule action qui supprime le risque ; tout le reste réduit l'exposition ou prévient la récidive.
- **Comment éviter que ça recommence ?** Des barrières automatiques à plusieurs niveaux (avant le commit, à la réception sur GitHub, en CI) et un emplacement prévu pour les secrets, hors du code (voir action 6).

### Actions, dans l'ordre

| # | Action | Pourquoi à ce moment-là |
|---|---|---|
| 1 | **Ne pas utiliser ni diffuser le secret ; ouvrir un incident** et prévenir le propriétaire du compte Xray (formateur, responsable sécurité). | Tester le secret serait un accès non autorisé et brouillerait l'analyse des journaux. Seul le propriétaire du compte peut révoquer : il doit être alerté en premier. |
| 2 | **Révoquer le secret et en générer un nouveau** dans l'administration Xray. | C'est la seule action qui ferme réellement l'accès. Elle passe avant tout nettoyage : chaque minute de retard laisse un secret public valide. Réécrire l'historique d'abord ne protégerait rien (copies déjà faites). |
| 3 | **Analyser les journaux d'accès Xray depuis le 20/10/2026** : connexions, exports ou modifications inhabituels. | Une fois la porte fermée, on vérifie si quelqu'un est passé pendant le mois d'exposition. Si oui, l'incident devient une fuite de données avérée à traiter comme telle. |
| 4 | **Ranger le nouveau secret hors du code** : *secret* GitHub Actions (`Settings > Secrets and variables > Actions`) lu par variable d'environnement, jamais écrit dans un fichier versionné. | Le service qui utilisait l'ancien secret doit continuer à fonctionner, sans recréer la même faille. |
| 5 | **Purger l'historique** : `git filter-repo --path qa/utils/xray_export.py --invert-paths` sur une copie fraîche, `git push --force` sur toutes les branches, demander au support GitHub de purger les vues en cache, et prévenir les détenteurs de forks et de clones de repartir d'un clone propre. | Après la révocation, le secret ne vaut plus rien : la purge est de l'hygiène (éviter les alertes à répétition et qu'un futur secret ne suive le même chemin). Elle réécrit l'historique de tout le monde : à coordonner, pas à faire dans l'urgence. |
| 6 | **Empêcher la récidive** : `gitleaks git --redact` en hook pre-commit et comme étape bloquante de la CI (avec le `.gitleaks.toml` du dépôt) ; activer le *secret scanning* et la *push protection* de GitHub ; ajouter `.env` (tous dossiers) au `.gitignore` avec des fichiers `.env.example` ; revue de code attentive aux constantes nommées `SECRET`, `TOKEN`, `KEY`. | Une barrière avant le commit (poste), une à la réception (GitHub), une en CI (pull request) : si l'une est contournée, les autres arrêtent la fuite. |
| 7 | **Clore l'incident par une fiche RCA** (comme `docs/rca/B4.md`) : chronologie, durée d'exposition, cause racine, actions. | Garder une trace et vérifier que les actions de prévention ont bien été mises en place. |

---

## 3. Faux positifs acceptés

| Constat ignoré | Nombre | Où | Pourquoi on l'accepte | Comment on l'ignore proprement |
|---|---|---|---|---|
| bandit **B101** `assert_used` | 91 | Tous les fichiers `qa/tests/**`, `qa/conftest.py:12`, `qa/utils/factories.py:29,36` | pytest repose sur `assert` : c'est ainsi qu'un test vérifie un résultat. Le risque signalé (assertions supprimées par `python -O`) ne concerne que du code de production qui compterait sur `assert` pour un contrôle de sécurité ; `qa/` n'est jamais livré en production. | Désactiver B101 **uniquement pour `qa/`** : `bandit -r qa -x qa/.venv -s B101`, ou `skips: ['B101']` dans un fichier de configuration bandit propre à `qa/`. Ne pas l'ignorer pour `backend/`. |
| bandit **B105 / B107** mot de passe en dur, **comptes jetables** | 5 | `qa/tests/api/test_auth.py:37,41`, `qa/tests/performance/locustfile.py:58,59`, `qa/utils/factories.py:33` | Ce mot de passe sert à créer un compte neuf et unique pendant le test (e-mail aléatoire `@example.com`), qui ne donne accès à rien d'autre. Ce n'est pas un secret. | Commentaire `# nosec B105` (ou `B107`) **sur la ligne**, avec la raison : `# nosec B105 - compte jetable cree par le test`. Ne pas désactiver B105 globalement. |
| pip-audit : **lignes en double** | 3 | sortie de `pip-audit -r qa/requirements.txt` | Ce n'est pas un faux positif sur la vulnérabilité, mais un doublon d'affichage : 6 lignes pour 3 avis réels. On compte les identifiants distincts, pas les lignes. | Rien à ignorer : la correction (#8) fait disparaître les six lignes. |

**Ce qui n'est pas accepté comme faux positif :**

- **B105 sur `test_promo.py:7` et `:42`** : identifiants des vrais comptes de démonstration écrits en dur (constat #13).
- **`artipacked`** malgré sa confiance « Low » : la correction coûte une ligne (#15).
- **L'absence d'alerte gitleaks sur `qa/.env` et `docker-compose.yml`** : un faux négatif, pas un feu vert (#3, #4, #10). Un scan vert ne prouve pas l'absence de secret.
