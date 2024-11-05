# Cours Ultra Complet de Go (Golang) - Partie 2 : Concepts Avancés

## Table des matières - Partie 2
1. Fonctions et méthodes avancées
2. Interfaces et polymorphisme
3. Gestion des erreurs
4. Concurrence et parallélisme
5. Tests avancés
6. Outils et bonnes pratiques
7. Patterns de conception en Go

## 1. Fonctions et méthodes avancées

### 1.1 Fonctions comme types
```go
// Définition de type fonction
type Operation func(a, b int) int

// Utilisation comme paramètre
func calculer(a, b int, op Operation) int {
    return op(a, b)
}

// Exemples d'implémentation
func addition(a, b int) int { return a + b }
func multiplication(a, b int) int { return a * b }

func main() {
    // Utilisation
    resultat1 := calculer(5, 3, addition)
    resultat2 := calculer(5, 3, multiplication)
    
    // Fonction anonyme
    resultat3 := calculer(5, 3, func(a, b int) int {
        return a - b
    })
}
```

### 1.2 Closures avancées
```go
func generateurCompteur(debut int) func() int {
    compteur := debut
    return func() int {
        temp := compteur
        compteur++
        return temp
    }
}

func exempleClosure() {
    // Création de plusieurs compteurs indépendants
    compteur1 := generateurCompteur(0)
    compteur2 := generateurCompteur(10)
    
    fmt.Println(compteur1()) // 0
    fmt.Println(compteur1()) // 1
    fmt.Println(compteur2()) // 10
    fmt.Println(compteur2()) // 11
}
```

### 1.3 Méthodes avec pointeurs
```go
type Compte struct {
    solde float64
}

// Méthode avec récepteur pointeur
func (c *Compte) debiter(montant float64) error {
    if montant > c.solde {
        return errors.New("solde insuffisant")
    }
    c.solde -= montant
    return nil
}

// Méthode avec récepteur valeur
func (c Compte) getSolde() float64 {
    return c.solde
}

// Méthode variadic
func (c *Compte) deposerMultiple(montants ...float64) {
    for _, montant := range montants {
        c.solde += montant
    }
}
```

### 1.4 Génériques (Go 1.18+)
```go
// Type générique simple
type Stack[T any] struct {
    elements []T
}

func (s *Stack[T]) Push(element T) {
    s.elements = append(s.elements, element)
}

func (s *Stack[T]) Pop() (T, error) {
    var zero T
    if len(s.elements) == 0 {
        return zero, errors.New("stack vide")
    }
    element := s.elements[len(s.elements)-1]
    s.elements = s.elements[:len(s.elements)-1]
    return element, nil
}

// Fonction générique avec contraintes
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

## 2. Interfaces et polymorphisme

### 2.1 Interfaces avancées
```go
// Interface composée
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}

// Interface vide
type Any interface{}

// Interface avec génériques
type Container[T any] interface {
    Add(item T)
    Get() T
}

// Implémentation
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Add(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Get() T {
    if len(s.items) == 0 {
        var zero T
        return zero
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}
```

### 2.2 Type assertions et switches
```go
func processerInterface(i interface{}) {
    // Type assertion simple
    str, ok := i.(string)
    if ok {
        fmt.Printf("C'est une chaîne: %s\n", str)
    }

    // Type switch
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %v\n", v)
    case []interface{}:
        fmt.Printf("Slice: %v\n", v)
    default:
        fmt.Printf("Type inconnu: %T\n", v)
    }
}
```

## 3. Gestion des erreurs avancée

### 3.1 Erreurs personnalisées
```go
// Type d'erreur personnalisé
type ValidationError struct {
    Field string
    Error string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("erreur de validation pour %s: %s", e.Field, e.Error)
}

// Erreurs avec données additionnelles
type QueryError struct {
    Query   string
    Message string
    Code    int
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("erreur de requête [%d]: %s", e.Code, e.Message)
}

// Wrapping d'erreurs
func processerDonnees() error {
    err := faireQuelqueChose()
    if err != nil {
        return fmt.Errorf("échec du traitement: %w", err)
    }
    return nil
}
```

### 3.2 Gestion des paniques
```go
func gestionPanique() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Récupération de la panique: %v\n", r)
            debug.PrintStack()
        }
    }()
    
    // Code pouvant paniquer
    panic("une erreur critique")
}

// Utilisation dans une fonction HTTP
func handlerSecurise(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("Panique dans le handler: %v", err)
            http.Error(w, "Erreur interne", http.StatusInternalServerError)
        }
    }()
    
    // Code du handler
}
```

## 4. Concurrence et parallélisme

### 4.1 Patterns de concurrence avancés
```go
// Pattern Worker Pool
func workerPool(jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    
    // Lancement des workers
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- traiterJob(job)
            }
        }()
    }
    
    // Attendre la fin de tous les workers
    go func() {
        wg.Wait()
        close(results)
    }()
}

// Pattern Pipeline
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Utilisation du pipeline
    c := generator(2, 3, 4)
    out := square(c)
    
    // Consommation des résultats
    for result := range out {
        fmt.Println(result)
    }
}
```

### 4.2 Synchronisation avancée
```go
// Utilisation de sync.Once
type Service struct {
    once     sync.Once
    instance *Instance
}

func (s *Service) getInstance() *Instance {
    s.once.Do(func() {
        s.instance = &Instance{}
    })
    return s.instance
}

// Pool d'objets
var pool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func handleRequest() {
    buffer := pool.Get().([]byte)
    defer pool.Put(buffer)
    // Utilisation du buffer
}

// WaitGroup avec timeout
func waitTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
    c := make(chan struct{})
    go func() {
        defer close(c)
        wg.Wait()
    }()
    select {
    case <-c:
        return false // Terminé normalement
    case <-time.After(timeout):
        return true // Timeout
    }
}
```

## 5. Tests avancés

### 5.1 Tests de table
```go
func TestCalculatrice(t *testing.T) {
    tests := []struct {
        nom     string
        a, b    int
        op      string
        attendu int
        erreur  bool
    }{
        {"addition simple", 2, 3, "+", 5, false},
        {"division par zéro", 1, 0, "/", 0, true},
        {"multiplication", 4, 5, "*", 20, false},
    }
    
    for _, tt := range tests {
        t.Run(tt.nom, func(t *testing.T) {
            resultat, err := calculer(tt.a, tt.b, tt.op)
            if (err != nil) != tt.erreur {
                t.Errorf("erreur inattendue: %v", err)
            }
            if err == nil && resultat != tt.attendu {
                t.Errorf("got %v, want %v", resultat, tt.attendu)
            }
        })
    }
}
```

### 5.2 Benchmarking avancé
```go
func BenchmarkConcatString(b *testing.B) {
    var str string
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        str += "x"
    }
}

func BenchmarkConcatBuilder(b *testing.B) {
    var builder strings.Builder
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        builder.WriteString("x")
    }
}

// Benchmark parallel
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // Code à benchmarker
        }
    })
}
```

### 5.3 Mock et tests d'intégration
```go
// Interface à mocker
type Database interface {
    Get(id string) (string, error)
    Set(id string, value string) error
}

// Implementation mock
type MockDB struct {
    data map[string]string
}

func (m *MockDB) Get(id string) (string, error) {
    if val, ok := m.data[id]; ok {
        return val, nil
    }
    return "", errors.New("not found")
}

// Test avec mock
func TestService(t *testing.T) {
    mock := &MockDB{data: make(map[string]string)}
    service := NewService(mock)
    
    // Test du service
    result, err := service.ProcessData("test")
    if err != nil {
        t.Errorf("unexpected error: %v", err)
    }
    // Vérifications
}
```

## 6. Outils et bonnes pratiques

### 6.1 Outils de développement
```bash
# Formattage de code
go fmt ./...

# Vérification de code
go vet ./...

# Documentation
godoc -http=:6060

# Tests de couverture
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Profilage
go test -cpuprofile=cpu.prof -memprofile=mem.prof -bench=.
go tool pprof cpu.prof
```

### 6.2 Patterns de performance
```go
// Pool d'objets pour réduire les allocations
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func processData() {
    buffer := bufferPool.Get().([]byte)
    defer bufferPool.Put(buffer)
    // Utilisation du buffer
}

// Réduction des allocations
type Parser struct {
    buffer []byte
}

func (p *Parser) Reset() {
    p.buffer = p.buffer[:0]
}

// Utilisation de strings.Builder
func concatenateStrings(items []string) string {
    var builder strings.Builder
    builder.Grow(len(items) * 8)
    for _, item := range items {
        builder.WriteString(item)
    }
    return builder.String()
}
```

## 7. Patterns de conception en Go

### 7.1 Singleton
```go
type singleton struct{}

var instance *singleton
var once sync.Once

func getInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
```

### 7.2 Factory
```go
type Animal interface {
    Parler() string
}

type Chat struct{}
type Chien struct{}

func (c *Chat) Parler() string { return "Miaou" }
func (c *Chien) Parler() string { return "Wouf" }

func CreateAnimal(typeAnimal string) Animal {
    switch typeAnimal {
    case "chat":
        return &Chat{}
    case "chien":
        return &Chien{}
    default:
        return nil
    }
}
```

### 7.3 Observer
```go
type Observer interface {
    Update(message string)
}

type Subject struct {
    observers []Observer
    message   string
}

func (s *Subject) Attach(o Observer) {
    s.observers = append(s.observers, o)
}

func (s *Subject) Notify() {
    for _, observer := range s.observers {
        observer.Update(s.message)
    }
}

func (s *Subject) UpdateMessage(message string) {
    s.message = message
    s.Notify()
}
```

### 7.4 Middleware pattern
```go
type Middleware func(http.HandlerFunc) http.HandlerFunc

func Chain(f http.HandlerFunc, middlewares ...Middleware) http.HandlerFunc {
    for _, m := range middlewares {
        f = m(f)
    }
    return f
}

// Exemple de middleware
func Logger() Middleware {
    return func(f http.HandlerFunc) http.