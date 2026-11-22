# Requêtes SQL types : fiche de révision

Base Jeu de rôle (TP), syntaxe H2.

```
CLASSE     (idclasse, libelle, pointsparniveau, armurepossible)
PERSONNAGE (idpersonnage, idclasse#, nom, race, niveau, datecreation)
OBJET      (numobjet, libelle, categorie, prix)
POSSEDER   (idpersonnage#, numobjet#, quantite, dateacquisition)
CONNAITRE  (idpersonnage#, idsort#, dateconnu)
SORT       (idsort, nom, niveau, action)
COMPOSER   (idsort#, numingredient#, quantite)
INGREDIENT (numingredient, libelle, prix)
```

---

## 0. Repérer le type de requête dans l'énoncé

| Dans l'énoncé… | Outil |
|---|---|
| « différents » | `DISTINCT` |
| « …de Gandalf », 2+ tables | `INNER JOIN` |
| « même … que X » | auto-jointure |
| « y compris ceux qui n'ont pas » | `LEFT OUTER JOIN` |
| « aucun », « ne … pas » | `NOT IN` / `NOT EXISTS` / `EXCEPT` |
| « à la fois A et B » | `INTERSECT` ou 2× `IN` |
| « A ou B » (2 sources) | `UNION` |
| « plus que la moyenne » | sous-requête `AVG` |
| « le plus … », « maximal » | `= (SELECT MAX…)` |
| « par … », « pour chaque … » | `GROUP BY` |
| « au moins N … » | `HAVING COUNT(*) >= N` |
| « TOUS les … » | **division** |

---

## 1. Le squelette

```sql
SELECT   [DISTINCT] colonnes | expressions | *
FROM     tables
WHERE    condition sur les LIGNES
GROUP BY colonnes de regroupement
HAVING   condition sur les GROUPES
ORDER BY col [ASC|DESC], … ;
```

---

## 2. Une seule table

**Valeurs distinctes + tri**
```sql
SELECT DISTINCT race
FROM personnage
ORDER BY race;
```

**Champ calculé + alias + IN**
```sql
SELECT libelle, prix, prix * 1.15 AS prix_majore
FROM objet
WHERE categorie IN ('Arme', 'Armure');
```

**BETWEEN + différent + tri multiple**
```sql
SELECT nom, niveau
FROM personnage
WHERE race <> 'Humain'
  AND niveau BETWEEN 5 AND 10      -- bornes incluses
ORDER BY niveau DESC, nom;         -- DESC ne vaut que pour niveau
```

**LIKE** (`%` = suite quelconque, `_` = 1 caractère)
```sql
SELECT nom
FROM sort
WHERE nom LIKE 'Soins%'
   OR LOWER(action) LIKE '%d_g_ts%';   -- dégâts / dégats
```

**NULL**
```sql
SELECT idpersonnage
FROM personnage
WHERE nom IS NULL;                 -- jamais = NULL
```

**Dates**
```sql
SELECT idpersonnage, idsort, dateconnu
FROM connaitre
WHERE YEAR(dateconnu) = 2008
  AND MONTH(dateconnu) BETWEEN 6 AND 9;
-- aussi : CURRENT_DATE, DATEDIFF('DAY', d1, d2)
```

---

## 3. Jointures

**2 tables**
```sql
SELECT p.nom, c.libelle
FROM personnage p INNER JOIN classe c
     ON (p.idclasse = c.idclasse);

-- Syntaxe 1, équivalente
SELECT p.nom, c.libelle
FROM personnage p, classe c
WHERE p.idclasse = c.idclasse;
```

**3 tables (avec la table d'association au milieu)**
```sql
SELECT s.nom, c.dateconnu
FROM (personnage p
      INNER JOIN connaitre c ON (p.idpersonnage = c.idpersonnage))
      INNER JOIN sort s      ON (c.idsort = s.idsort)
WHERE p.nom = 'Gandalf';
```
> Rappel : n tables → n−1 conditions de jointure.

**Auto-jointure (« même race qu'Aragorn »)**
```sql
SELECT p2.nom
FROM personnage p1 INNER JOIN personnage p2
     ON (p1.race = p2.race)
WHERE p1.nom = 'Aragorn'
  AND p2.nom <> 'Aragorn';
```

**Jointure externe (garder tout le monde)**
```sql
SELECT p.nom, c.idsort
FROM personnage p LEFT OUTER JOIN connaitre c
     ON (p.idpersonnage = c.idpersonnage);
-- Boromir, Gimli → idsort = NULL
```

**Anti-jointure (ceux qui n'ont RIEN)**
```sql
SELECT p.idpersonnage
FROM personnage p LEFT OUTER JOIN connaitre c
     ON (p.idpersonnage = c.idpersonnage)
WHERE c.idsort IS NULL;            -- BOR, GIM
```

**Jointure naturelle** (sur toutes les colonnes de même nom ⚠️)
```sql
SELECT nom, dateconnu
FROM personnage NATURAL JOIN connaitre;
```

---

## 4. Sous-requêtes

**Une seule valeur → `=`, `<`, `>`**
```sql
SELECT libelle, prix
FROM objet
WHERE prix > (SELECT AVG(prix) FROM objet);
```

**Le maximum**
```sql
SELECT nom, niveau
FROM sort
WHERE niveau = (SELECT MAX(niveau) FROM sort);
-- ❌ SELECT nom, MAX(niveau) FROM sort;
```

**Plusieurs valeurs → IN**
```sql
SELECT nom
FROM personnage
WHERE idpersonnage IN (
    SELECT po.idpersonnage
    FROM posseder po INNER JOIN objet o
         ON (po.numobjet = o.numobjet)
    WHERE o.categorie = 'Arme');
```

**EXISTS (corrélée : utilise la ligne courante p)**
```sql
SELECT p.nom
FROM personnage p
WHERE EXISTS (
    SELECT *
    FROM posseder po INNER JOIN objet o
         ON (po.numobjet = o.numobjet)
    WHERE o.categorie = 'Arme'
      AND po.idpersonnage = p.idpersonnage);
```

**« Aucun » → NOT IN / NOT EXISTS**
```sql
SELECT nom
FROM personnage
WHERE idpersonnage NOT IN (SELECT idpersonnage FROM posseder);

SELECT p.nom
FROM personnage p
WHERE NOT EXISTS (SELECT *
                  FROM posseder po
                  WHERE po.idpersonnage = p.idpersonnage);
-- Arwen, Saroumane
```

**Corrélée avec agrégat (« plus que la moyenne de SA race »)**
```sql
SELECT p.nom, p.race, p.niveau
FROM personnage p
WHERE p.niveau > (SELECT AVG(p2.niveau)
                  FROM personnage p2
                  WHERE p2.race = p.race);
```

---

## 5. Combinaisons

> Les deux SELECT doivent avoir le même nombre de colonnes, de types compatibles (uni-compatibles).
> Le `ORDER BY` s'écrit une seule fois, à la fin.

**UNION**
```sql
SELECT libelle, 'Objet' AS type FROM objet
UNION
SELECT libelle, 'Ingrédient' FROM ingredient
ORDER BY libelle;
```

**INTERSECT (≈ IN)**
```sql
SELECT idpersonnage FROM connaitre
INTERSECT
SELECT idpersonnage FROM posseder;
```

**EXCEPT (≈ NOT IN), attention R − S ≠ S − R**
```sql
SELECT idpersonnage FROM connaitre
EXCEPT
SELECT idpersonnage FROM posseder;     -- ARW, SAR
```

**« À la fois A et B »**
```sql
SELECT nom FROM personnage
WHERE idpersonnage IN (SELECT po.idpersonnage
                       FROM posseder po INNER JOIN objet o
                            ON (po.numobjet = o.numobjet)
                       WHERE o.libelle = 'Epée longue')
  AND idpersonnage IN (SELECT po.idpersonnage
                       FROM posseder po INNER JOIN objet o
                            ON (po.numobjet = o.numobjet)
                       WHERE o.libelle = 'Targe');
-- ❌ WHERE libelle = 'Epée longue' AND libelle = 'Targe' → 0 ligne
```

---

## 6. Agrégation

`COUNT` · `SUM` · `AVG` · `MIN` · `MAX`

**Sur toute la table (1 seul groupe)**
```sql
SELECT COUNT(*), AVG(niveau), MIN(niveau), MAX(niveau)
FROM personnage;
```

**GROUP BY**
```sql
SELECT c.libelle, COUNT(*) AS nb, AVG(p.niveau) AS moy
FROM classe c INNER JOIN personnage p
     ON (c.idclasse = p.idclasse)
GROUP BY c.libelle
ORDER BY nb DESC;
```
> Règle d'or : tout ce qui est dans le SELECT sans fonction d'agrégation doit être dans le GROUP BY.

**Compter avec les zéros (LEFT JOIN + COUNT(colonne))**
```sql
SELECT p.nom, COUNT(c.idsort) AS nb_sorts
FROM personnage p LEFT OUTER JOIN connaitre c
     ON (p.idpersonnage = c.idpersonnage)
GROUP BY p.idpersonnage, p.nom;
-- COUNT(c.idsort) → 0 ; COUNT(*) → 1 ❌
```

**Grouper par une expression**
```sql
SELECT YEAR(dateconnu) AS annee, COUNT(*)
FROM connaitre
GROUP BY YEAR(dateconnu)
ORDER BY annee;
```

**HAVING (filtre APRÈS le regroupement)**
```sql
SELECT race, COUNT(*), AVG(niveau)
FROM personnage
GROUP BY race
HAVING COUNT(*) >= 2 AND AVG(niveau) >= 8;
```

**WHERE + HAVING**
```sql
SELECT idpersonnage, COUNT(*)
FROM connaitre
WHERE dateconnu >= '2008-01-01'    -- filtre AVANT (lignes)
GROUP BY idpersonnage
HAVING COUNT(*) >= 3;              -- filtre APRÈS (groupes)
```

**SUM d'un produit**
```sql
SELECT p.nom, SUM(po.quantite * o.prix) AS valeur
FROM (personnage p
      INNER JOIN posseder po ON (p.idpersonnage = po.idpersonnage))
      INNER JOIN objet o     ON (po.numobjet = o.numobjet)
GROUP BY p.idpersonnage, p.nom
HAVING SUM(po.quantite * o.prix) > 100
ORDER BY valeur DESC;
```

**COUNT(DISTINCT …)**
```sql
SELECT p.race, COUNT(DISTINCT po.numobjet)
FROM personnage p INNER JOIN posseder po
     ON (p.idpersonnage = po.idpersonnage)
GROUP BY p.race;
```

**Le groupe le plus gros (pas de MAX(COUNT(*)) !)**
```sql
SELECT idclasse, COUNT(*)
FROM personnage
GROUP BY idclasse
HAVING COUNT(*) = (SELECT MAX(nb)
                   FROM (SELECT COUNT(*) AS nb
                         FROM personnage
                         GROUP BY idclasse) t);
```

---

## 7. Division : « TOUS les … »

Exemple : les personnages qui connaissent **tous** les sorts de niveau 1.

**Version 1 : double NOT EXISTS (celle du cours)**
> « Il n'existe pas de sort de niveau 1 tel qu'il n'existe pas de ligne connaitre pour ce perso et ce sort. »
```sql
SELECT p.nom
FROM personnage p
WHERE NOT EXISTS (
    SELECT *
    FROM sort s
    WHERE s.niveau = 1
      AND NOT EXISTS (
          SELECT *
          FROM connaitre c
          WHERE c.idsort = s.idsort
            AND c.idpersonnage = p.idpersonnage));
-- Gandalf
```

**Version 2 : comptage**
```sql
SELECT p.nom
FROM (personnage p
      INNER JOIN connaitre c ON (p.idpersonnage = c.idpersonnage))
      INNER JOIN sort s      ON (c.idsort = s.idsort)
WHERE s.niveau = 1
GROUP BY p.idpersonnage, p.nom
HAVING COUNT(DISTINCT s.idsort) =
       (SELECT COUNT(*) FROM sort WHERE niveau = 1);
```

Gabarit :
`dividende ÷ diviseur` → NOT EXISTS (diviseur … AND NOT EXISTS (dividende lié aux deux))

---

## 8. Vues

```sql
CREATE VIEW v_cout_sort AS
  SELECT s.idsort, s.nom, SUM(co.quantite * i.prix) AS cout
  FROM (sort s
        INNER JOIN composer co  ON (s.idsort = co.idsort))
        INNER JOIN ingredient i ON (co.numingredient = i.numingredient)
  GROUP BY s.idsort, s.nom;

-- s'utilise comme une table
SELECT nom, cout
FROM v_cout_sort
WHERE cout > (SELECT AVG(cout) FROM v_cout_sort);
```

```sql
CREATE VIEW v_humains AS
  SELECT idpersonnage, nom, niveau
  FROM personnage
  WHERE race = 'Humain'
  WITH CHECK OPTION;   -- toute ligne insérée doit vérifier race='Humain'
```
> Une vue est une table **virtuelle**, réévaluée à chaque utilisation.
> On peut mettre à jour à travers une vue si elle porte sur 1 seule table, sans champ calculé, et si les colonnes absentes de la vue acceptent NULL ou ont un DEFAULT.

---

## 9. LDD : structure

```sql
CREATE TABLE guilde (
  idguilde   VARCHAR(3)   CONSTRAINT pk_guilde PRIMARY KEY,
  nom        VARCHAR(30)  NOT NULL UNIQUE,
  ville      VARCHAR(30)  DEFAULT 'Minas Tirith',
  cotisation NUMERIC(6,2) CONSTRAINT ck_cot CHECK (cotisation >= 0)
);
```

**Clé primaire composée**
```sql
CREATE TABLE archive (
  idpersonnage    VARCHAR(3) NOT NULL,
  numobjet        INTEGER    NOT NULL,
  quantite        SMALLINT,
  dateacquisition DATE,
  CONSTRAINT pk_archive PRIMARY KEY (idpersonnage, numobjet)
);
```

**ALTER TABLE**
```sql
ALTER TABLE personnage ADD idguilde VARCHAR(3);
ALTER TABLE personnage DROP idguilde;
ALTER TABLE classe ADD CONSTRAINT pk_classe PRIMARY KEY (idclasse);
ALTER TABLE personnage
  ADD CONSTRAINT fk_personnage_classe
  FOREIGN KEY (idclasse) REFERENCES classe;
```

**DROP**
```sql
DROP TABLE guilde;           -- structure + données
DELETE FROM guilde;          -- données seulement
TRUNCATE TABLE guilde;       -- données seulement
```

> PRIMARY KEY = UNIQUE + NOT NULL

---

## 10. LMD : données

```sql
-- toutes les colonnes, dans l'ordre
INSERT INTO guilde VALUES ('ISE', 'Isengard', 'Orthanc', 50);

-- colonnes choisies (les autres : DEFAULT ou NULL)
INSERT INTO guilde (idguilde, nom, cotisation)
VALUES ('COM', 'Communauté', 15);

-- plusieurs lignes d'un coup
INSERT INTO archive
  SELECT * FROM posseder
  WHERE dateacquisition < '2007-12-31';

UPDATE objet
SET prix = prix * 1.10
WHERE categorie = 'Arme';     -- sans WHERE → TOUTES les lignes

DELETE FROM posseder
WHERE dateacquisition < '2007-12-31';
```

---

## 11. Algèbre relationnelle ↔ SQL

| AR | SQL |
|---|---|
| π A,B (R) | `SELECT A, B FROM R` |
| σ cond (R) | `SELECT * FROM R WHERE cond` |
| ρ A/newA (R) | `SELECT A AS newA FROM R` |
| R ∪ S | `UNION` |
| R ∩ S | `INTERSECT` |
| R − S | `EXCEPT` / `NOT IN` |
| R × S | `FROM R, S` |
| R ⋈cond S | `INNER JOIN … ON cond` |
| R ÷ S | double `NOT EXISTS` |

**Exemples**
```
π nom (σ libelle='Targe' (Personnage ⋈ Posseder ⋈ Objet))

Optimisé (σ le plus tôt possible) :
π nom (Personnage ⋈ Posseder ⋈ σ libelle='Targe' (Objet))

Personnages sans sort :
π idpersonnage (Personnage) − π idpersonnage (Connaitre)

Personnages connaissant tous les sorts de niveau 1 :
π idpersonnage,idsort (Connaitre) ÷ π idsort (σ niveau=1 (Sort))
```

> R ⋈cond S = σ cond (R × S)

---

## 12. Checklist anti-pièges ✅

- [ ] `IS NULL`, jamais `= NULL`
- [ ] `COUNT(*)` compte les lignes, `COUNT(col)` ignore les NULL
- [ ] Une condition sur un agrégat va dans `HAVING`, pas dans `WHERE`
- [ ] Tout ce qui n'est pas agrégé dans le SELECT est dans le `GROUP BY`
- [ ] `=` + sous-requête seulement si elle renvoie **1 valeur**, sinon `IN`
- [ ] n tables → n−1 conditions de jointure (sinon produit cartésien)
- [ ] Une colonne ambiguë (nom, libelle, niveau, quantite, prix) doit être préfixée par l'alias
- [ ] `DESC` porte sur **une** colonne
- [ ] LIKE : `%` et `_` en H2 (`*` et `?` sous Access)
- [ ] UNION/INTERSECT/EXCEPT : même nombre de colonnes, `ORDER BY` à la fin
- [ ] « Tous » → division ; « aucun » → NOT IN / NOT EXISTS
- [ ] UPDATE / DELETE sans WHERE → toute la table
- [ ] Chaînes entre `'apostrophes'`

Bon courage ! 🍀
