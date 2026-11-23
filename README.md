# OTTO – Multi-Objective Recommender System

Système de recommandation pour le jeu de données e-commerce **OTTO** (Kaggle) : à partir du début d'une session de navigation, prédire les produits qui seront **cliqués**, **ajoutés au panier** et **commandés**.

**Score Kaggle (soumission tardive) : 0,52728 (privé) / 0,52592 (public)**
Pour référence, le 1er du classement final obtient 0,60503.

---

## 1. Problème

Les sessions de test sont coupées au milieu. Pour chaque session, on propose jusqu'à **20 produits** pour chacun des trois types d'action (`clicks`, `carts`, `orders`).

- **Métrique** : Recall@20 pondéré, avec les poids 0,10 (clics), 0,30 (paniers) et 0,60 (commandes).
- **Type de problème** : recommandation *session-based* pure. Il n'y a pas d'identifiant utilisateur, seulement les événements de la session en cours.
- **Difficulté** : plus d'un million de produits, sans titre ni image (seulement des identifiants `aid`), et des sessions souvent très courtes.

## 2. Données

Les données ne sont **pas** dans ce dépôt (environ 12 Go). Elles se téléchargent sur la page Kaggle de la compétition.

| Fichier | Contenu |
|---|---|
| `train.jsonl` | sessions complètes (`session`, `events` avec `aid`, `ts`, `type`) |
| `test.jsonl` | 1 671 803 sessions tronquées |
| `sample_submission.csv` | format attendu : `session_type,labels` (5 015 409 lignes) |

Constats sur un échantillon de 100 000 sessions :

- 5 227 653 événements : 91,3 % de clics, 7,0 % de paniers, 1,8 % de commandes.
- Longueur des sessions : médiane de 19 événements, maximum de 495. Un quart des sessions a 6 événements ou moins.
- 663 079 produits distincts dans cet échantillon.

## 3. Méthode

La solution finale est une **heuristique de candidats**, sans modèle appris.

1. **Matrice de co-visitation** : pour chaque produit A, on compte combien de fois chaque produit B apparaît juste après lui dans une session.
2. **Produits déjà vus** : les produits de la session, classés du plus récent au plus ancien (selon leur *dernière* apparition).
3. **Voisins pondérés** : les 3 derniers produits de la session reçoivent les poids 1, 2 et 3. On additionne leurs voisins de la matrice, pondérés, et on garde les meilleurs.
4. **Complément** : si la liste a moins de 20 produits, on la complète avec les produits les plus populaires.
5. **Logique par type d'action** : pour `carts` et `orders`, les produits déjà mis au panier dans la session passent en tête de liste.

Ordre final de la liste : (produits du panier, pour carts et orders), produits déjà vus, voisins de co-visitation, produits populaires.

## 4. Résultats

### Validation locale

On prend les 100 000 premières sessions du train : 80 000 pour apprendre (matrice, popularité) et 20 000 pour valider. Chaque session de validation est coupée à un point aléatoire : le début est connu, la fin est la réponse cachée.

| Méthode | Score de validation |
|---|---|
| Popularité (mêmes 20 produits pour tous) | 0,0103 |
| Co-visitation simple (dernier produit) | 0,0947 |
| Co-visitation pondérée (3 derniers produits, poids 1-2-3) | 0,1218 |
| Produits déjà vus (ordre de récence) + co-visitation | 0,2460 |
| + produits du panier en tête pour `carts` et `orders` | **0,2597** |
| Re-ranking LightGBM (6 variables) | 0,2204 |

Observations :

- La co-visitation apporte un très grand gain par rapport à la popularité.
- Classer les produits déjà vus par leur **dernière** apparition plutôt que par la première a donné environ +0,018.
- Le re-ranking LightGBM n'a **pas** battu l'heuristique de récence (0,2204 contre 0,2460). Les variables testées étaient trop peu nombreuses, et l'entraînement portait sur un petit échantillon.

### Soumission Kaggle

Pour la soumission finale, la matrice est construite sur les 100 000 sessions, puis le fichier est généré sur `test.jsonl`.

| | Score |
|---|---|
| Privé | **0,52728** |
| Public | 0,52592 |
| Meilleur score du classement final (référence) | 0,60503 |

![Soumission Kaggle](results/kaggle_submission.png)

Le score de soumission est bien supérieur au score de validation. La validation est donc **pessimiste** (coupure aléatoire, seulement 80 000 sessions d'apprentissage) : elle sert à comparer les méthodes entre elles, pas à prédire le score réel.

## 5. Limites et pistes d'amélioration

- La matrice de co-visitation n'utilise qu'environ 0,8 % des sessions d'entraînement. Sur les 50 000 premières sessions de test, **13,7 %** ont un dernier produit absent de la matrice et reçoivent donc seulement des produits populaires.
- Pistes : construire la matrice sur beaucoup plus de sessions (lecture par morceaux), ajouter d'autres matrices (clic vers clic, panier ou commande vers panier ou commande), et enrichir le re-ranking avec plus de variables.

## 6. Structure du dépôt

```
otto-recommender/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── otto-recommender-covisitation.ipynb   # exploration, validation, méthodes, soumission
└── results/
    └── kaggle_submission.png                  # capture de l'onglet Submissions
```

## 7. Reproduire

1. Ouvrir un notebook Kaggle et ajouter la compétition *OTTO – Multi-Objective Recommender System* en entrée (accepter d'abord les règles de la compétition).
2. Importer `notebooks/otto-recommender-covisitation.ipynb`.
3. Vérifier le chemin des données avec `os.walk('/kaggle/input')`. Dans nos essais : `/kaggle/input/competitions/otto-recommender-system/`.
4. Lancer les cellules dans l'ordre. La génération du fichier de soumission prend quelques minutes.
5. En local, installer les dépendances avec `pip install -r requirements.txt` et adapter le chemin des données. Les 12 Go de données dépassent la mémoire d'un ordinateur portable standard, donc un notebook Kaggle est recommandé.

## Auteure

Salma Brika – Master MLAIM, FSDM Fès (Université Sidi Mohamed Ben Abdellah)
GitHub : [brikasalma](https://github.com/brikasalma) · Kaggle : `salmabrika`

## Références

- Compétition Kaggle : *OTTO – Multi-Objective Recommender System* (`kaggle.com/competitions/otto-recommender-system`).
