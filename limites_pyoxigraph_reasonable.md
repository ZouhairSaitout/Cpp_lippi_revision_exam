# pyoxigraph + reasonable : limites et contournements

## Gain mesuré

Données DNS synthétiques, même ontologie, mêmes inférences obtenues des deux côtés :

| Triplets | rdflib + owlrl | pyoxigraph + reasonable (+ contournements) |
|---|---|---|
| 10 k | 7,3 s | 0,7 s |
| 100 k | 83 s | 9,2 s |
| 500 k | 428 s | 66 s |

Le parsing et la sérialisation deviennent négligeables : à 500 k triplets, on passe de 60 s à 2 s.

## Limites de reasonable

| Limite | Conséquence | Contournement |
|---|---|---|
| Pas de chaînes de propriétés (`prp-spo2`) | `owl:propertyChainAxiom` ignoré | Générer un `INSERT` SPARQL par chaîne et boucler reasonable + inserts jusqu'au point fixe |
| `owl:FunctionalProperty` (`prp-fp`) quadratique | Mémoire saturée (6 Go à ~60 k triplets) | Retirer les déclarations `FunctionalProperty` avant reasonable et appliquer la règle en SPARQL (`?x :p ?y1, ?y2 FILTER(?y1 != ?y2)` → `owl:sameAs`) |
| Pas de `owl:AllDisjointClasses` (`cax-adc`) | Incohérences non détectées | Développer en paires `owl:disjointWith` (`cax-dw` est supporté), ou requête SPARQL de vérification |
| Pas de sémantique des datatypes | Pas de disjonction `xsd:boolean` / … | Fichier statique d'axiomes, à charger seulement si GraphDB les produit |
| Pas de `eq-ref` | Pas de `x owl:sameAs x` | Rien à faire : ces triplets n'apportent aucune information |
| Non implémentés : `eq-diff*`, `prp-key`, `cls-maxc*`, `cls-oo`, `prp-npa*` | Selon les constructions utilisées | Inventorier les constructions OWL présentes dans l'ontologie (requête SPARQL) avant de décider |
| Nœuds vides renommés à chaque aller-retour | Doublons à chaque itération, store qui grossit sans fin | Ne réinjecter que les triplets inférés sans nœud vide |
| Entrée par graphe rdflib ou fichier (`.ttl`/`.n3`), sortie en termes rdflib | Coût de conversion (~20 % du temps d'un tour) | Passer par un fichier N-Triples, limiter les tours, tester le mode incrémental (`update_graph`) |

## Limites de pyoxigraph (par rapport à rdflib)

| Limite | Conséquence | Contournement |
|---|---|---|
| Parser strict (RDF standard uniquement) | Refuse les fichiers écrits après owlrl : sujets littéraux, ex. `"7.5"^^xsd:float a rdfs:Datatype` | Régénérer les fichiers avec la nouvelle chaîne, ou filtrer ces triplets |
| Pas de raisonneur | — | reasonable (voir ci-dessus) |
| Store en quads | `dump` en N-Triples/Turtle exige `from_graph=DefaultGraph()` | À préciser une fois dans libontology |
| `"x"` = `"x"^^xsd:string` (RDF 1.1) | Différences apparentes avec d'anciens fichiers rdflib, qui distingue les deux | Normaliser les littéraux avant toute comparaison ancien/nouveau |
| Comparaison de graphes avec nœuds vides | Une comparaison par ensembles de triplets échoue (labels arbitraires) | Isomorphisme (`rdflib.compare`), ou comparer les résultats des requêtes |
| Style de sérialisation (`[ ]` ou `_:b0`) | Cosmétique uniquement | — |

## Validation avant adoption

- Inventaire des constructions OWL utilisées dans l'ontologie interne.
- Mêmes requêtes métier sur GraphDB et sur la nouvelle chaîne : les résultats doivent être identiques.
- Diff de triplets après normalisation (sans `sameAs` réflexifs, sans triplets sur le vocabulaire `rdf:`/`owl:`/`xsd:`, nœuds vides traités par isomorphisme), regroupé par prédicat.
