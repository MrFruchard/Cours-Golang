# Cours Ultra Complet de Go (Golang) - Partie 1 : Fondamentaux!

## Table des matières - Partie 1
1. Introduction et historique
2. Installation et environnement
3. Syntaxe de base
4. Types de données fondamentaux
5. Structures de contrôle
6. Collections et structures de données

## 1. Introduction et historique

### 1.1 Histoire de Go
- Créé en 2007 chez Google par Robert Griesemer, Rob Pike et Ken Thompson
- Premières versions publiques en 2009
- Version 1.0 sortie en 2012
- Évolution majeure avec modules Go en 1.11

### 1.2 Philosophie de Go
- Simplicité et clarté du code
- Gestion efficace de la concurrence
- Compilation rapide
- Support natif du networking
- Garbage collection automatique
- Rétrocompatibilité garantie

### 1.3 Cas d'usage
- Services web et microservices
- Applications cloud natives
- Outils en ligne de commande
- Systèmes distribués
- DevOps et automatisation
- Applications haute performance

## 2. Installation et environnement

### 2.1 Installation par système d'exploitation

#### Linux
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install golang-go

# CentOS/RHEL
sudo yum install golang

# Installation manuelle
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
```

#### Configuration de l'environnement Linux/MacOS
```bash
# Ajoutez ces lignes à ~/.bashrc ou ~/.zshrc
export GOPATH=$HOME/go
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
export GO111MODULE=on

# Rechargez le shell
source ~/.bashrc
```

#### Windows
```powershell
# Avec Chocolatey
choco install golang

# Configuration des variables d'environnement
setx GOPATH "%USERPROFILE%\go"
setx PATH "%PATH%;%USERPROFILE%\go\bin"
```

### 2.2 Vérification de l'installation
```bash
go version
go env
```

### 2.3 Configuration de l'IDE
Principales options recommandées :
- VSCode avec l'extension Go
- GoLand de JetBrains
- Vim/Neovim avec plugins Go

#### Configuration VSCode
```json
{
    "go.useLanguageServer": true,
    "go.lintTool": "golangci-lint",
    "go.formatTool": "goimports",
    "editor.formatOnSave": true,
    "[go]": {
        "editor.codeActionsOnSave": {
            "source.organizeImports": true
        }
    }
}
```

## 3. Syntaxe de base

### 3.1 Structure d'un programme Go
```go
// Déclaration du package
package main

// Imports
import (
    "fmt"
    "strings"
    "time"
)

// Constantes
const (
    Pi = 3.14159
    Version = "1.0.0"
)

// Variables globales
var (
    counter int
    prefix  string
)

// Structure principale
func main() {
    // Code principal
    fmt.Println("Hello, World!")
}
```

### 3.2 Déclaration de variables

```go
// Méthodes de déclaration
func exemplesDeclarations() {
    // Déclaration explicite
    var nombre int = 42
    var texte string = "Hello"
    
    // Inférence de type
    age := 25
    message := "Bonjour"
    
    // Déclaration multiple
    var x, y, z int = 1, 2, 3
    
    // Déclaration avec valeur zéro
    var (
        compteur int     // 0
        nom     string   // ""
        actif   bool    // false
        valeur  float64 // 0.0
    )
    
    // Conversion de types
    var i int = 42
    var f float64 = float64(i)
    var u uint = uint(f)
}
```

### 3.3 Types de données fondamentaux

```go
func exempleTypes() {
    // Nombres entiers
    var (
        a int8    = 127         // -128 à 127
        b int16   = 32767       // -32768 à 32767
        c int32   = 2147483647  // -2147483648 à 2147483647
        d int64   = 9223372036854775807
        
        ua uint8  = 255         // 0 à 255
        ub uint16 = 65535       // 0 à 65535
        uc uint32 = 4294967295  // 0 à 4294967295
        ud uint64 = 18446744073709551615
    )
    
    // Nombres à virgule flottante
    var (
        f32 float32 = 3.14159  // Précision simple
        f64 float64 = 3.14159265359  // Précision double
    )
    
    // Nombres complexes
    var (
        c64  complex64  = 1 + 2i
        c128 complex128 = 1.5 + 2.4i
    )
    
    // Booléens
    var (
        vrai  bool = true
        faux  bool = false
    )
    
    // Chaînes de caractères
    var (
        str1 string = "Hello"
        str2 string = `Multiple
                      lignes`
        char rune   = 'A'  // Unicode character
    )
    
    // Pointeurs
    var ptr *int
    nombre := 42
    ptr = &nombre
}
```

### 3.4 Opérateurs

```go
func exemplesOperateurs() {
    // Arithmétiques
    a := 10
    b := 3
    
    somme := a + b      // 13
    diff := a - b       // 7
    prod := a * b       // 30
    quot := a / b       // 3
    reste := a % b      // 1
    
    // Incrémentation/Décrémentation
    a++                 // a = a + 1
    b--                 // b = b - 1
    
    // Comparaison
    egal := a == b      // false
    diff := a != b      // true
    sup := a > b        // true
    inf := a < b        // false
    supEg := a >= b     // true
    infEg := a <= b     // false
    
    // Logiques
    et := true && false // false
    ou := true || false // true
    non := !true        // false
    
    // Bits
    x := 12             // 1100 en binaire
    y := 5              // 0101 en binaire
    
    et_bit := x & y     // 4 (0100)
    ou_bit := x | y     // 13 (1101)
    xor := x ^ y        // 9 (1001)
    non_bit := ^x       // -13
    left := x << 1      // 24 (11000)
    right := x >> 1     // 6 (0110)
}
```

## 4. Structures de contrôle

### 4.1 Conditions

```go
func exemplesConditions() {
    // If simple
    age := 18
    if age >= 18 {
        fmt.Println("Majeur")
    } else {
        fmt.Println("Mineur")
    }
    
    // If avec initialisation
    if note := obtenirNote(); note >= 10 {
        fmt.Println("Réussite")
    } else if note >= 8 {
        fmt.Println("Rattrapage")
    } else {
        fmt.Println("Échec")
    }
    
    // Switch classique
    jour := "Lundi"
    switch jour {
    case "Lundi":
        fmt.Println("Début de semaine")
    case "Mercredi":
        fmt.Println("Milieu de semaine")
    case "Vendredi":
        fmt.Println("Fin de semaine")
    default:
        fmt.Println("Autre jour")
    }
    
    // Switch avec condition
    switch {
    case age < 13:
        fmt.Println("Enfant")
    case age < 20:
        fmt.Println("Adolescent")
    case age < 60:
        fmt.Println("Adulte")
    default:
        fmt.Println("Senior")
    }
    
    // Switch avec fallthrough
    switch nombre := 75; {
    case nombre >= 90:
        fmt.Println("A")
        fallthrough
    case nombre >= 80:
        fmt.Println("B")
        fallthrough
    case nombre >= 70:
        fmt.Println("C")
    default:
        fmt.Println("D")
    }
}
```

### 4.2 Boucles

```go
func exemplesBoucles() {
    // Boucle for classique
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
    
    // Boucle while-style
    compteur := 0
    for compteur < 5 {
        fmt.Println(compteur)
        compteur++
    }
    
    // Boucle infinie
    for {
        fmt.Println("Infini")
        break // Sort de la boucle
    }
    
    // Range sur slice
    nombres := []int{1, 2, 3, 4, 5}
    for index, valeur := range nombres {
        fmt.Printf("Index: %d, Valeur: %d\n", index, valeur)
    }
    
    // Range sur map
    scores := map[string]int{
        "Alice": 95,
        "Bob":   87,
        "Charlie": 92,
    }
    for cle, valeur := range scores {
        fmt.Printf("%s a obtenu %d\n", cle, valeur)
    }
    
    // Continue et break
    for i := 0; i < 10; i++ {
        if i == 5 {
            continue // Passe à l'itération suivante
        }
        if i == 8 {
            break // Sort de la boucle
        }
        fmt.Println(i)
    }
    
    // Labels et goto
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i*j > 4 {
                break outer
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
}
```

## 5. Collections et structures de données

### 5.1 Arrays

```go
func exemplesArrays() {
    // Déclaration et initialisation
    var nombres [5]int                    // Array de 5 entiers
    var notes [3]float64 = [3]float64{15.5, 17.0, 12.5}
    couleurs := [4]string{"rouge", "vert", "bleu", "jaune"}
    
    // Array avec taille automatique
    jours := [...]string{"lundi", "mardi", "mercredi"}
    
    // Accès aux éléments
    premier := nombres[0]
    dernier := nombres[len(nombres)-1]
    
    // Modification d'éléments
    nombres[2] = 42
    
    // Parcours d'array
    for i := 0; i < len(couleurs); i++ {
        fmt.Printf("Couleur %d: %s\n", i, couleurs[i])
    }
    
    // Parcours avec range
    for index, valeur := range notes {
        fmt.Printf("Note %d: %.1f\n", index, valeur)
    }
    
    // Arrays multidimensionnels
    var matrice [3][3]int
    matrice[0][0] = 1
    matrice[1][1] = 5
    matrice[2][2] = 9
    
    // Copie d'array
    var copieNotes [3]float64
    copieNotes = notes // Copie complète
}
```

### 5.2 Slices

```go
func exemplesSlices() {
    // Création de slices
    nombres := []int{1, 2, 3, 4, 5}
    
    // Création avec make
    slice := make([]int, 5)    // longueur 5, capacité 5
    slice2 := make([]int, 5, 10) // longueur 5, capacité 10
    
    // Append
    nombres = append(nombres, 6)
    nombres = append(nombres, 7, 8, 9)
    
    // Slicing
    partie := nombres[1:4]    // éléments 1 à 3
    debut := nombres[:3]      // du début à 2
    fin := nombres[3:]        // de 3 à la fin
    
    // Copy
    destination := make([]int, len(nombres))
    copied := copy(destination, nombres)
    
    // Capacité et longueur
    fmt.Printf("Longueur: %d, Capacité: %d\n", len(nombres), cap(nombres))
    
    // Slices de slices
    matrix := make([][]int, 3)
    for i := range matrix {
        matrix[i] = make([]int, 3)
    }
    
    // Append efficace
    efficace := make([]int, 0, 1000)
    for i := 0; i < 1000; i++ {
        efficace = append(efficace, i)
    }
}
```

### 5.3 Maps

```go
func exemplesMaps() {
    // Création
    scores := make(map[string]int)
    
    // Initialisation directe
    ages := map[string]int{
        "Alice":   25,
        "Bob":     30,
        "Charlie": 35,
    }
    
    // Ajout et modification
    scores["Alice"] = 95
    scores["Bob"] = 87
    scores["Charlie"] = 92
    
    // Suppression
    delete(scores, "Bob")
    
    // Vérification d'existence
    score, existe := scores["Alice"]
    if existe {
        fmt.Printf("Score d'Alice: %d\n", score)
    }
    
    // Parcours
    for nom, age := range ages {
        fmt.Printf("%s a %d ans\n", nom, age)
    }
    
    // Maps de maps
    utilisateurs := map[string]map[string]string{
        "alice": {
            "email": "alice@example.com",
            "role":  "admin",
        },
        "bob": {