# Compte rendu : TP API LogiAI

**Nom:** SIHADI Aya **Nom Binome :** ABDELMOUNIM Youssef **Groupe :** G4

## 1. Environnement et premiers tests
- Sortie de `python -m pytest` au départ (collez-la) :
**Reponse:**
PS C:\Users\lenovo\Desktop\TPs_5IAD\DEV_DEP_G4\FastAPI\starter> python -m pytest
=================== test session starts ====================
platform win32 -- Python 3.14.0, pytest-9.2.0, pluggy-1.7.0
rootdir: C:\Users\lenovo\Desktop\TPs_5IAD\DEV_DEP_G4\FastAPI\starter
plugins: anyio-4.13.0
collected 15 items                                                                                             

tests\test_api.py ...............                                                                        [100%]

============================================== warnings summary ===============================================
..\..\..\..\..\AppData\Roaming\Python\Python314\site-packages\anyio\pytest_plugin.py:14
  C:\Users\lenovo\AppData\Roaming\Python\Python314\site-packages\anyio\pytest_plugin.py:14: PytestRemovedIn10Warning: _pytest.python.CallSpec2 has been renamed to CallSpec.
  The CallSpec2 alias will be removed in pytest 10.
  Update imports to use CallSpec instead.
  See https://docs.pytest.org/en/stable/deprecations.html#callspec2-renamed
    from _pytest.python import CallSpec2

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================================== 15 passed, 1 warning in 8.38s ========================================



- Pourquoi certains tests qui attendent un 422 passent-ils déjà, alors que rien n'est écrit ?
**Reponse:**
Au début, le schéma Pydantic était presque vide et refusait les
champs inconnus. Les requêtes invalides étaient donc rejetées en `422`, mais
pour la mauvaise raison : elles n'étaient pas réellement contrôlées par les
bornes ou par le type du champ. Le test d'une livraison valide qui doit renvoyer
`200` permet de détecter ce problème. Un test vert ne signifie pas toujours que
la fonctionnalité est correctement implémentée ; il faut aussi vérifier la
raison de son succès.



## 2. Mon schéma
- Mes bornes et leur justification physique (une ligne par variable) :

| Variable | Min | Max | Pourquoi |
|---|---|---|---|
| distance_km | 1 | 1000 | |
| poids_kg | | | |
| heure_depart | | | |
| pluie_mm | | | |
| trafic_index | | | |
| nb_arrets | | | |

**Reponse:**

- Mes bornes et leur justification physique :

| Variable | Min | Max | Pourquoi |
|---|---:|---:|---|
| distance_km | 1 | 1000 | Une distance minimale de 1 km est réaliste et 1000 km couvre un long trajet. |
| poids_kg | 0.1 | 3000 | Le poids doit être positif et 3000 kg couvre une charge lourde réaliste. |
| heure_depart | 0 | 23 | Une heure de départ doit correspondre à une heure valide de la journée. |
| pluie_mm | 0 | 100 | La pluie ne peut pas être négative ; 100 mm représente une très forte pluie. |
| trafic_index | 0 | 10 | 0 signifie trafic fluide et 10 trafic totalement saturé. |
| nb_arrets | 0 | 12 | Une livraison peut ne comporter aucun arrêt et 12 couvre une tournée importante. |
| type_vehicule | moto, fourgon, camion | moto, fourgon, camion | Seuls les types prévus par le modèle sont acceptés. |


->Les bornes sont définies dans `logiai/api/schemas.py` avec
`Field(ge=..., le=...)`. Elles servent à valider les entrées, à documenter
automatiquement l'API dans `/docs` et à retourner une erreur `422` lorsqu'une
valeur est impossible.


## 3. Ma fonction predict()
(collez le code)

**Reponse:**

def predict(self, features: dict) -> dict:
        """features : un dictionnaire avec les 7 variables du contrat."""
        df = pd.DataFrame([features], columns=FEATURES)
        proba = float(self.model.predict_proba(df)[0, 1])
        return {
            "probabilite_retard": round(proba, 4),
            "alerte": proba >= self.threshold,
            "seuil": self.threshold,
            "model_version": self.version,
        }

## 4. Quatre requêtes
| Requête | Code obtenu | Pourquoi |
|---|---|---|
| livraison normale | | |
| distance_km = -5 | | |
| type_vehicule = "avion" | | |
| modèle absent (supprimez models/ et relancez) | | |

**Reponse:**


| Requête | Code obtenu | Pourquoi |
|---|---:|---|
| Livraison normale | 200 | Les sept champs sont présents et respectent le schéma ; la prédiction est calculée. |
| `distance_km = -5` | 422 | La distance est inférieure à la borne minimale `1 km`. |
| `type_vehicule = "avion"` | 422 | `avion` ne fait pas partie des valeurs autorisées par `Literal`. |
| Modèle absent | 503 | L'API peut démarrer, mais le predictor n'est pas disponible pour effectuer une prédiction. |

-> `422` correspond à une requête qui ne respecte pas le contrat
d'entrée. `503` correspond à un service momentanément indisponible, ici parce
que le modèle n'est pas chargé.


## 5. Questions de compréhension
1. Pourquoi charge-t-on le modèle au démarrage et pas à chaque requête ?
->Le modèle est chargé une seule fois au démarrage et reste en mémoire. Le
   charger à chaque requête serait lent, gaspillerait des ressources et
   augmenterait le temps de réponse. Le chargement au démarrage permet de
   réutiliser le même modèle pour toutes les requêtes.

2. Qui écrit le contrôle « pluie <= 100 », et où est-il écrit ?
-> C'est Pydantic, utilisé par FastAPI, qui applique ce contrôle à partir de la
   déclaration `pluie_mm: float = Field(ge=0, le=100)` dans
   `logiai/api/schemas.py`, avant l'appel de la fonction `predict()`.

3. Différence entre 422 et 503 ?
->`422` signifie que les données envoyées par le client sont invalides ou ne
   respectent pas le schéma. `503` signifie que le service est temporairement
   indisponible, par exemple lorsque le modèle de prédiction est absent.

4. Pourquoi une route `/v1/predict` plutôt que `/predict` ?
->Le préfixe `/v1` indique la version du contrat de l'API. Il sera possible de
   créer une version `/v2` plus tard tout en conservant `/v1` pour les anciens
   clients et sans les casser.


5. Que se passerait-il si la distance était déclarée `int` ?
->  Les distances décimales, comme `120.5`, seraient refusées avec une erreur
   `422`. Une distance est une mesure qui peut contenir des décimales ; elle
   doit donc être déclarée `float`.



## 6. Décision : modèle absent
Répondre 503 ou refuser de démarrer ? Qu'est-ce que chaque option fait perdre ?

->**Réponse :** Je choisis l'option A : démarrer l'API et répondre `503` sur
`/v1/predict` lorsque le modèle est absent. Cette option permet à `/health` de
répondre et permet au superviseur de constater que `model_loaded` vaut `false`,
ce qui facilite le diagnostic et le redéploiement. En revanche, les prédictions
ne sont pas disponibles tant que le modèle n'est pas restauré.

L'option B, qui consiste à refuser de démarrer, empêche toute requête et rend le
service indisponible immédiatement. Elle peut être préférable en production si
une API sans modèle ne doit jamais être exposée, mais elle empêche aussi
`/health` de signaler clairement le problème.

## 7. Lien de la pull request

[Pull request #1](https://github.com/sihadi/logiai-api/pull/1)





1