# Révision C++ : syntaxe essentielle

## Classe et constructeurs

```cpp
class Rat {
    int num, denom;                        // privé par défaut dans une class (public dans une struct)
public:
    class DivZero {};                      // classe exception (vide : son type suffit)
    Rat() : num(0), denom(1) {}            // constructeur par défaut
    Rat(int n) : num(n), denom(1) {}       // conversion : permet Rat r = 3;
    Rat(int p, int q);                     // déclaré ici, défini dans le .cpp
    int get_num() const { return num; }    // const : ne modifie pas l'objet
    int get_denom() const { return denom; }
    Rat& operator+=(const Rat& r);
    friend ostream& operator<<(ostream& os, const Rat& r);
};
```
- Un constructeur n'a **pas de type de retour** et porte le nom de la classe.
- La liste d'initialisation est `: num(0), denom(1)`.
- Dès qu'on écrit un constructeur, le constructeur par défaut n'est plus généré.

## Définitions dans le .cpp

```cpp
int pgcd(int a, int b) { while (b != 0) { int r = a % b; a = b; b = r; } return a; }

Rat::Rat(int p, int q) {                   // Rat:: car on est hors de la classe
    if (q == 0) throw DivZero();
    if (q < 0) { p = -p; q = -q; }
    int d = pgcd(abs(p), q);
    num = p / d;  denom = q / d;
}

Rat& Rat::operator+=(const Rat& r) {
    *this = Rat(num * r.denom + r.num * denom, denom * r.denom);   // le constructeur simplifie
    return *this;                                                   // toujours *this
}

Rat operator+(Rat a, const Rat& b) { return a += b; }   // hors classe, a par valeur

bool operator==(const Rat& a, const Rat& b) { return a.get_num() * b.get_denom() == b.get_num() * a.get_denom(); }
bool operator!=(const Rat& a, const Rat& b) { return !(a == b); }   // à partir de ==

ostream& operator<<(ostream& os, const Rat& r) {   // pas de Rat::, pas de friend ici
    os << r.num << "/" << r.denom;
    return os;
}
```

| | `+=` | `+` |
|---|---|---|
| Modifie la gauche ? | oui | non |
| Type de retour | `Rat&` | `Rat` |
| Où | membre (`Rat::`) | hors classe |

## Classe avec des pointeurs : le trio

```cpp
List(const List& l);                // constructeur de copie : recrée les cellules
List& operator=(const List& l);     // if (this != &l) { delete head; recopier; } return *this;
~List();                            // delete head;
```

## Héritage

```cpp
class Pixel : public Point {
    string color;
public:
    Pixel(int x, int y, string c) : Point(x, y), color(c) {}   // la base d'abord
};
```
- Sans appel explicite, c'est le constructeur **par défaut** de la base qui est appelé.
- **Construction** : base, puis membres, puis corps. **Destruction** : l'inverse, automatiquement.

## Virtuel, abstrait, override

```cpp
class Figure {
public:
    virtual double aire() const = 0;   // virtuelle pure, donc Figure est abstraite (non instanciable)
    virtual ~Figure() {}               // destructeur virtuel
};

class Carre : public Figure {
    double c;
public:
    Carre(double c) : c(c) {}
    double aire() const override { return c * c; }   // signature identique, const compris
};
```
- **virtual** : c'est le type **dynamique** (l'objet réel) qui décide, à travers un pointeur ou une référence.
- **non virtual** : c'est le type **statique** (le type déclaré) qui décide. On parle de **masquage**.
- **override** n'est possible que sur une fonction virtuelle de la base. Il fait vérifier la signature par le compilateur.
- **Destructeur virtuel** : nécessaire pour que `delete p` avec `Figure *p = new Carre(2);` appelle `~Carre`, puis `~Figure`.
- Une classe dérivée qui ne redéfinit pas toutes les fonctions virtuelles pures reste **abstraite**.

## Exceptions

```cpp
if (head == 0) throw Empty();              // lever (objet construit avec le constructeur par défaut)

try { l.get_first(); }
catch (List::Empty) { cout << "vide"; }    // List:: hors de la classe
catch (Rat::DivZero) { cout << "div 0"; }  // testés dans l'ordre
```
- `throw` quitte immédiatement la fonction. Le programme reprend **après** le `catch` qui correspond.
- Les objets locaux sont détruits en remontant.
- Si aucun `catch` ne correspond, le programme s'arrête (`terminate`).
