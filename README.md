# Classification du trafic réseau d'un campus — UNSW-NB15

Analyse exploratoire, prétraitement et modélisation binaire (trafic normal / attaque)
sur le jeu de données public **UNSW-NB15**.

Le projet suit trois étapes : comprendre les données (`01_eda`), construire des jeux
d'entraînement et de test propres (`02_preprocessing`), puis entraîner et comparer
quatre modèles (`03_models`).

## Contenu du dépôt

```
├── notebooks/
│   ├── 01_eda.ipynb            # analyse exploratoire
│   ├── 02_preprocessing.ipynb  # nettoyage, features, folds, pipelines
│   └── 03_models.ipynb         # modèles, seuils, SHAP, analyse d'erreurs
├── requirements.txt
└── README.md
```

Les données, les artefacts (`data/processed`, `artifacts/`) et le rapport Word/PDF ne
sont pas versionnés : ils se régénèrent avec les notebooks.

## Données

Télécharger les deux fichiers CSV du jeu UNSW-NB15 :

- `UNSW_NB15_training-set.csv`
- `UNSW_NB15_testing-set.csv`

Source : <https://research.unsw.edu.au/projects/unsw-nb15-dataset>

Puis les placer dans `data/` :

```
data/
├── UNSW_NB15_training-set.csv
└── UNSW_NB15_testing-set.csv
```

## Installation

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate

pip install -r requirements.txt
```

## Exécution

Lancer Jupyter depuis la racine du projet (ou depuis `notebooks/`, les chemins gèrent
les deux cas) :

```bash
jupyter lab
```

Puis exécuter les notebooks dans l'ordre :

1. **`01_eda.ipynb`** — constats, distributions, doublons, corrélations.
2. **`02_preprocessing.ipynb`** — écrit `data/processed/*.parquet` et
   `artifacts/pipeline_*.joblib`.
3. **`03_models.ipynb`** — lit ces fichiers, entraîne les modèles et produit les
   tableaux et figures de la partie modélisation.

## Méthode

- **Cible** : `label` (0 = normal, 1 = attaque). `attack_cat` est exclue des variables
  d'entrée, car c'est la version détaillée de la cible (fuite directe).
- **Doublons** : les lignes identiques sont repérées par hachage, les folds sont
  calculés sur les flux uniques puis propagés aux copies. Un flux et ses doublons
  restent donc dans le même fold.
- **Deux branches de features** :
  - *arbres* : 62 variables, valeurs brutes + one-hot ;
  - *linéaire* : 49 variables, `log1p` sur les colonnes asymétriques, standardisation
    et suppression des colonnes corrélées à plus de 0,95 (Spearman).
- **Modèles** : régression logistique, MLP, forêt aléatoire, XGBoost.
- **Seuil** : choisi sur les probabilités out-of-fold du train, jamais sur le test.
- **Graine** : `random_state = 42` partout.

## Résultats (test officiel, seuil 0,5)

| Modèle | F1 (validation croisée) | F1 test | PR-AUC test | FNR | FPR |
|---|---|---|---|---|---|
| XGBoost | 0,9636 | **0,9175** | **0,9883** | 3,48 % | 17,01 % |
| Forêt aléatoire | **0,9645** | 0,9107 | 0,9864 | 2,48 % | 20,41 % |
| MLP | 0,9615 | 0,8816 | 0,9837 | 1,85 % | 30,02 % |
| Régression logistique | 0,9503 | 0,8675 | 0,9664 | 7,46 % | 25,50 % |

Les Fuzzers restent la famille la plus difficile à détecter (rappel 0,767 pour
XGBoost), tandis que Generic, DoS et Reconnaissance sont quasiment parfaits.

