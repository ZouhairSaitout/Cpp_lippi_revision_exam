# Révision C++ : prédire les sorties

## Ex 1 : copie, affectation, destructeur

```cpp
class A {
public:
    int id;
    A(int i = 0) : id(i)        { cout << "C" << id << " "; }
    A(const A& a) : id(a.id+10) { cout << "K" << id << " "; }
    A& operator=(const A& a)    { id = a.id + 100; cout << "=" << id << " "; return *this; }
    ~A()                        { cout << "D" << id << " "; }
};
void f(A a)        { cout << "f "; }
void g(const A& a) { cout << "g "; }

int main() {
    A x(1);  A y = x;  A z;  z = y;  f(x);  g(x);  cout << endl;
}
```

**Sortie :** `C1 K11 C0 =111 K11 f D11 g` puis `D111 D11 D1`

- `A y = x;` est une **construction**, donc le constructeur de copie est appelé (pas `=`).
- `z = y;` : `z` existe déjà, donc c'est l'**affectation**.
- `f(A a)` est un passage **par valeur** : copie à l'entrée (`K11`), destruction à la sortie (`D11`).
- `g(const A&)` est un passage **par référence** : aucune copie, aucune destruction.
- En fin de bloc, la destruction se fait dans l'**ordre inverse** de la construction : z, puis y, puis x.

---

## Ex 2 : héritage avec une donnée membre

```cpp
class A {
public:
    A()      { cout << "A "; }
    A(int n) { cout << "A" << n << " "; }
    ~A()     { cout << "~A "; }
};
class B : public A {
    A membre;
public:
    B()                          { cout << "B "; }
    B(int n) : A(n), membre(n+1) { cout << "B" << n << " "; }
    ~B()                         { cout << "~B "; }
};
int main() { B b1; cout << "| "; B b2(5); cout << "| "; }
```

**Sortie :** `A A B | A5 A6 B5 | ~B ~A ~A ~B ~A ~A`

- **Construction :** la base, puis les membres (dans l'ordre de déclaration), puis le corps du constructeur.
- **Destruction :** l'ordre exactement inverse. Le corps de `~B`, puis `~A` du membre, puis `~A` de la base.
- Les destructeurs du membre et de la base sont appelés **automatiquement**, sans les écrire.
- Sans liste d'initialisation, c'est le constructeur **par défaut** de la base qui est appelé.
- `b2` est détruit avant `b1` : le dernier construit est le premier détruit.

---

## Ex 3 : les déclarations piégées

```cpp
class R {
public:
    R()             { cout << "0 "; }
    R(int k)        { cout << "1 "; }
    R(int p, int q) { cout << "2 "; }
};
int main() {
    R a;        // 0
    R b();      // rien : c'est la DÉCLARATION d'une fonction qui renvoie un R
    R c(3);     // 1
    R d = 4;    // 1 : conversion int → R avec R(int)
    R e(2, 3);  // 2
    R t[3];     // 0 0 0 : tableau de 3 R, chacun construit avec R()
}
```

- `R t[3] = {7, 8};` affiche `1 1 0`.
- Sans `R()`, `R t[3];` **ne compile pas**. Dès qu'on écrit un constructeur, le compilateur ne génère plus celui par défaut.

---

## Ex 4 : List (`list_simple`)

```cpp
List l;
l.append(3); l.prepend(7); l.append(1); l.prepend(5);
OUT(l);                      // l : 5 7 3 1
OUT(l.get_middle_value());   // l.get_middle_value() : 3
OUT(l.get_first());          // l.get_first() : 5
OUT(l);                      // l : 7 3 1
l.mirror();                  // (rien)
OUT(l);                      // l : 1 3 7
l.max() = 0;                 // (rien)
OUT(l);                      // l : 1 3 0
```

- `OUT(expr)` affiche `expr : valeur`. Une instruction qui n'est pas dans `OUT` n'affiche **rien**.
- `get_first` **retire la tête** et renvoie sa valeur.
- `get_middle_value` sur une longueur paire renvoie le **2ᵉ** des deux milieux.
- `mirror` **inverse** l'ordre des cellules, il ne trie pas.
- `max()` non-const renvoie `int&`, une référence vers la case du max. `= 0` modifie donc la liste.

---

## Ex 5 : partage de cellules (aucun constructeur de copie)

```cpp
List l1; l1.append(1); l1.append(2);
List l2 = l1;      // copie des POINTEURS head/tail : cellules partagées
l2.append(3);
OUT(l1);           // l1 : 1 2 3
l1.prepend(0);
OUT(l2);           // l2 : 1 2 3
```

- `append` modifie `tail->next`, qui est une **cellule partagée**, donc `l1` voit le 3.
- `prepend` ne modifie que `l1.head`, un champ propre à `l1`, donc `l2` ne voit pas le 0.
- **Avec `list-copy`**, les cellules sont recopiées et les listes sont indépendantes : `l1 : 1 2` et `l2 : 1 2 3`.
- **Sans constructeur de copie**, appeler deux fois `sum(List l)` fait que `get_first` supprime les cellules de `l1`. On obtient de la mémoire libérée, puis un double `delete` : plantage.
- En conception : partage = **agrégation** (losange vide), pas de partage = **composition** (losange plein).

---

## Ex 6 : virtual

```cpp
class Animal {
public:
    void nom()         { cout << "Animal "; }
    virtual void cri() { cout << "... "; }
};
class Chat : public Animal {
public:
    void nom() { cout << "Chat "; }
    void cri() { cout << "Miaou "; }
};
int main() {
    Chat c;  Animal *p = &c;  Animal a = c;
    p->nom(); p->cri();   // Animal Miaou
    a.nom();  a.cri();    // Animal ...
    c.nom();  c.cri();    // Chat Miaou
}
```

- Pour une fonction **non virtuelle**, c'est le **type statique** qui décide (le type déclaré du pointeur).
- Pour une fonction **virtuelle**, c'est le **type dynamique** qui décide (l'objet réellement pointé).
- `virtual` n'agit qu'à travers un **pointeur** ou une **référence**.
- `Animal a = c;` provoque du **slicing** : la partie `Chat` est coupée, `a` est un vrai `Animal`.

---

## Les règles à retenir

1. Le constructeur de copie sert à **créer** un objet, `operator=` à modifier un objet qui **existe déjà**.
2. Un passage par valeur fait une copie à l'entrée et une destruction à la sortie. Un passage par `const &` ne fait rien de tout ça.
3. Construction : base, puis membres, puis corps. Destruction : l'ordre inverse, automatiquement.
4. Les objets d'un bloc sont détruits dans l'ordre inverse de leur création.
5. `R b();` est une fonction, pas un objet.
6. Pour une fonction virtuelle, c'est l'objet réel qui compte, mais seulement via un pointeur ou une référence.
