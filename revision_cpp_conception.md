# Révision C++ : conception objet

## Agrégation et composition

Les deux sont des relations **contenant / contenu**. La différence porte sur **qui possède les parties, et qui les détruit**.

| | **Agrégation** ◇ (losange vide) | **Composition** ◆ (losange plein) |
|---|---|---|
| Partage | possible | impossible (un seul propriétaire) |
| Destruction du tout | les parties **survivent** | les parties sont **détruites avec** |
| Exemple | playlist et chansons | maison et chambres |
| Tes listes | `list_simple` | `list-copy` |

Le losange se dessine **du côté du contenant** (`List`).

---

## En mémoire

Un objet `List` ne contient que 2 pointeurs (`head`, `tail`), sur la **pile**. Les cellules sont créées avec `new`, donc elles sont sur le **tas**.

**`list_simple` (agrégation)**
```
l1.head ─┐
l2.head ─┴──→ [1|•]──→[2|•]──→[3|0]      (après List l2 = l1;)
```
- La copie ne copie que les pointeurs, donc les cellules sont **partagées**.
- Il n'y a pas de destructeur : les cellules restent sur le tas, c'est une **fuite mémoire**.
- Un destructeur avec `delete` provoquerait un **double delete**, puisque les deux listes supprimeraient les mêmes cellules.

**`list-copy` (composition)**
```
l1.head ──→ [1|•]──→[2|•]──→[3|0]
l2.head ──→ [1|•]──→[2|•]──→[3|0]        (nouvelles cellules)
```
- Le **constructeur de copie** recrée les cellules avec `append`.
- **`operator=`** vérifie `this != &l`, fait `delete head`, puis recopie.
- Le **destructeur** fait `delete head`, et `~Cell` fait `delete next`, ce qui supprime toute la chaîne.
- `get_first` doit faire `p->next = 0` avant `delete p`, sinon toute la suite est détruite.

**Passer de l'agrégation à la composition** = écrire le destructeur, le constructeur de copie et `operator=`.

---

## Diagrammes de classes

```
list_simple (agrégation)

        0..*     début    0..1
 List  ◇──────────────────────>  Cell        0..1
       ◇──────────────────────>  (val:int) ─────┐
        0..*     fin      0..1       ◇  0..*     │ next
                                     └───────────┘

list-copy (composition) : même dessin, avec ◆ et 0..1 côté List et à la source de next
```

| Multiplicité | `list_simple` | `list-copy` | Pourquoi |
|---|---|---|---|
| côté Cell (`début`, `fin`, `next`) | 0..1 | 0..1 | au plus une tête, une fin, une suivante (0 si vide ou si c'est la dernière) |
| côté List | **0..\*** | **0..1** | avec partage, une cellule peut être la tête de plusieurs listes |
| source de `next` | **0..\*** | **0..1** | avec partage, plusieurs cellules peuvent pointer sur la même |

**La différence entre les deux diagrammes** : le losange vide devient plein, et la multiplicité côté `List` passe de 0..\* à **0..1**. En composition, une partie appartient à un seul tout.

`next` est une **association réflexive** : elle relie `Cell` à `Cell`.

---

## Diagramme d'objets (liste 11, 12, 13)

```
l.head                          l.tail
  ↓                               ↓
[11 | •]──→[12 | •]──→[13 | 0]
```
On dessine des **objets** (des valeurs concrètes), et pas des classes. Le dernier `next` vaut `0` (`nullptr`).

---

## Autres notations à connaître

- **Héritage** : une flèche à **triangle vide**, de la dérivée vers la base (`Pixel ──▷ Point`). Elle se lit « est un ».
- **Classe abstraite** : une classe avec au moins une fonction **virtuelle pure** (`= 0`). On ne peut pas l'instancier.
  - `Cell` n'est **pas** abstraite : elle n'a aucune fonction virtuelle pure.
- **Héritage private** (`Pile : private vector<int>`) : on réutilise l'implémentation sans exposer l'interface. Le `[]` devient inaccessible à l'utilisateur, pour ne pas casser le comportement LIFO. C'est pour la même raison que `s.append(12)` ne compile pas sur un `Stack`.
