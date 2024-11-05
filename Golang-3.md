# Cours Ultra Complet de Go (Golang) - Partie 3 : Applications Pratiques

## Table des matières - Partie 3
1. Applications Web avancées
2. Microservices avec Go
3. Performance et optimisation
4. Sécurité en Go
5. Base de données et persistance
6. DevOps et déploiement
7. Intégration avec d'autres systèmes

## 1. Applications Web avancées

### 1.1 Serveur HTTP avec middleware avancé
```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"
    
    "github.com/gorilla/mux"
    "github.com/rs/cors"
)

// Middleware de logging
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Créer un ID de requête unique
        requestID := uuid.New().String()
        ctx := context.WithValue(r.Context(), "requestID", requestID)
        
        // Wrapper pour capturer le status code
        ww := NewResponseWriter(w)
        
        // Appel du handler suivant
        next.ServeHTTP(ww, r.WithContext(ctx))
        
        // Logging après la requête
        duration := time.Since(start)
        log.Printf(
            "RequestID: %s Method: %s Path: %s Status: %d Duration: %v",
            requestID,
            r.Method,
            r.URL.Path,
            ww.Status(),
            duration,
        )
    })
}

// Configuration CORS avancée
func configureCORS() *cors.Cors {
    return cors.New(cors.Options{
        AllowedOrigins:   []string{"http://localhost:3000"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Authorization", "Content-Type"},
        ExposedHeaders:   []string{"X-Total-Count"},
        AllowCredentials: true,
        MaxAge:           300,
    })
}

// Configuration du routeur
func configureRouter() *mux.Router {
    r := mux.NewRouter()
    
    // API versioning
    api := r.PathPrefix("/api/v1").Subrouter()
    
    // Routes
    api.HandleFunc("/users", handleUsers).Methods("GET")
    api.HandleFunc("/users/{id}", handleUser).Methods("GET")
    api.HandleFunc("/users", createUser).Methods("POST")
    
    // Middleware
    r.Use(loggingMiddleware)
    r.Use(securityMiddleware)
    r.Use(compressionMiddleware)
    
    return r
}

func main() {
    router := configureRouter()
    corsHandler := configureCORS().Handler(router)
    
    srv := &http.Server{
        Handler:      corsHandler,
        Addr:         ":8080",
        WriteTimeout: 15 * time.Second,
        ReadTimeout:  15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    log.Fatal(srv.ListenAndServe())
}
```

### 1.2 WebSocket avec authentification
```go
package main

import (
    "github.com/gorilla/websocket"
    "net/http"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // Vérification de l'origine
        return true
    },
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
    id   string
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

func handleWebSocket(hub *Hub, w http.ResponseWriter, r *http.Request) {
    // Vérification du token
    token := r.URL.Query().Get("token")
    if !validateToken(token) {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    
    client := &Client{
        hub:  hub,
        conn: conn,
        send: make(chan []byte, 256),
        id:   generateClientID(),
    }
    
    client.hub.register <- client
    
    go client.writePump()
    go client.readPump()
}
```

## 2. Microservices avec Go

### 2.1 Service Discovery et Configuration
```go
package main

import (
    "github.com/hashicorp/consul/api"
    "github.com/spf13/viper"
)

type ServiceConfig struct {
    Name     string
    Port     int
    Version  string
    Database DatabaseConfig
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    Name     string
}

func loadConfig() (*ServiceConfig, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AutomaticEnv()
    
    var config ServiceConfig
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}

func registerService(config *ServiceConfig) error {
    consulConfig := api.DefaultConfig()
    client, err := api.NewClient(consulConfig)
    if err != nil {
        return err
    }
    
    registration := &api.AgentServiceRegistration{
        ID:      config.Name + "-" + config.Version,
        Name:    config.Name,
        Port:    config.Port,
        Address: "localhost",
        Check: &api.AgentServiceCheck{
            HTTP:     fmt.Sprintf("http://localhost:%d/health", config.Port),
            Interval: "10s",
            Timeout:  "3s",
        },
    }
    
    return client.Agent().ServiceRegister(registration)
}
```

### 2.2 Circuit Breaker et Resilience
```go
package main

import (
    "github.com/sony/gobreaker"
    "time"
)

type ServiceClient struct {
    breaker *gobreaker.CircuitBreaker
}

func NewServiceClient() *ServiceClient {
    settings := gobreaker.Settings{
        Name:        "API",
        MaxRequests: 3,
        Interval:    10 * time.Second,
        Timeout:     60 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            log.Printf("Circuit breaker %s changed from %v to %v", name, from, to)
        },
    }
    
    return &ServiceClient{
        breaker: gobreaker.NewCircuitBreaker(settings),
    }
}

func (s *ServiceClient) CallService() (interface{}, error) {
    result, err := s.breaker.Execute(func() (interface{}, error) {
        // Appel au service externe
        return http.Get("http://api.example.com/data")
    })
    
    if err != nil {
        return nil, err
    }
    
    return result, nil
}
```

## 3. Performance et optimisation

### 3.1 Profilage avancé
```go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "runtime/pprof"
)

func startProfiling() {
    // CPU profiling
    f, err := os.Create("cpu.prof")
    if err != nil {
        log.Fatal(err)
    }
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // Memory profiling
    runtime.MemProfileRate = 1
    
    // HTTP endpoint pour pprof
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}

func memoryOptimizedProcess() {
    // Pool d'objets pour réduire les allocations
    pool := sync.Pool{
        New: func() interface{} {
            return make([]byte, 1024)
        },
    }
    
    // Utilisation du pool
    buffer := pool.Get().([]byte)
    defer pool.Put(buffer)
    
    // Process avec le buffer
}
```

### 3.2 Optimisation de la mémoire
```go
type Cache struct {
    sync.RWMutex
    items map[string]Item
}

type Item struct {
    Value      interface{}
    Expiration int64
}

func NewCache() *Cache {
    c := &Cache{
        items: make(map[string]Item),
    }
    
    // Nettoyage périodique
    go c.janitor()
    
    return c
}

func (c *Cache) Set(key string, value interface{}, duration time.Duration) {
    c.Lock()
    defer c.Unlock()
    
    c.items[key] = Item{
        Value:      value,
        Expiration: time.Now().Add(duration).UnixNano(),
    }
}

func (c *Cache) janitor() {
    ticker := time.NewTicker(time.Minute)
    for {
        select {
        case <-ticker.C:
            c.DeleteExpired()
        }
    }
}
```

## 4. Sécurité en Go

### 4.1 Authentification JWT
```go
package auth

import (
    "github.com/dgrijalva/jwt-go"
    "time"
)

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.StandardClaims
}

func GenerateToken(userID, role string, secret []byte) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(24 * time.Hour).Unix(),
            IssuedAt:  time.Now().Unix(),
            Issuer:    "myapp",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secret)
}

func ValidateToken(tokenString string, secret []byte) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return secret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, jwt.ErrSignatureInvalid
}
```

### 4.2 Protection contre les attaques
```go
package security

import (
    "crypto/rand"
    "encoding/base64"
    "golang.org/x/crypto/bcrypt"
    "net/http"
)

// CSRF Protection
func generateCSRFToken() string {
    b := make([]byte, 32)
    rand.Read(b)
    return base64.URLEncoding.EncodeToString(b)
}

func CSRFMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" {
            token := r.Header.Get("X-CSRF-Token")
            if !validateCSRFToken(token) {
                http.Error(w, "Invalid CSRF token", http.StatusForbidden)
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}

// Rate Limiting
type RateLimiter struct {
    requests map[string][]time.Time
    mu       sync.RWMutex
}

func (rl *RateLimiter) Allow(ip string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-time.Minute)
    
    // Nettoyer les anciennes requêtes
    var recent []time.Time
    for _, t := range rl.requests[ip] {
        if t.After(windowStart) {
            recent = append(recent, t)
        }
    }
    
    rl.requests[ip] = recent
    
    // Vérifier la limite
    if len(recent) >= 100 {
        return false
    }
    
    rl.requests[ip] = append(rl.requests[ip], now)
    return true
}
```

## 5. Base de données et persistance

### 5.1 Gestion avancée avec GORM
```go
package database

import (
    "gorm.io/gorm"
    "gorm.io/driver/postgres"
)

type User struct {
    gorm.Model
    Name     string
    Email    string `gorm:"uniqueIndex"`
    Profile  Profile
    Articles []Article
}

type Profile struct {
    gorm.Model
    UserID   uint
    Avatar   string
    Bio      string
}

type Article struct {
    gorm.Model
    Title    string
    Content  string
    UserID   uint
    Tags     []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    gorm.Model
    Name     string
    Articles []Article `gorm:"many2many:article_tags;"`
}

func InitDB() (*gorm.DB, error) {
    dsn := "host=localhost user=postgres password=secret dbname=myapp port=5432"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        PrepareStmt: true,
    })
    
    if err != nil {
        return nil, err
    }
    
    // Migrations automatiques
    db.AutoMigrate(&User{}, &Profile{}, &Article{}, &Tag{})
    
    return db, nil
}
```

### 5.2 Transactions et requêtes complexes
```go
func CreateArticleWithTags(db *gorm.DB, article *Article, tagNames []string) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // Créer l'article
        if err := tx.Create(article).Error; err != nil {
            return err
        }
        
        // Gérer les tags
        for _, name := range tagNames {
            var tag Tag
            
            // Trouver ou créer le tag
            if err := tx.FirstOrCreate(&tag, Tag{Name: name}).Error; err != nil {
                return err
            }
            
            // Associer