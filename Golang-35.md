```go
            // Suite de la gestion des transactions
            if err := tx.Model(article).Association("Tags").Append(&tag); err != nil {
                return err
            }
        }
        
        return nil
    })
}

// Requêtes complexes avec GORM
func GetUserArticlesWithStats(db *gorm.DB, userID uint) ([]Article, error) {
    var articles []Article
    
    result := db.Model(&Article{}).
        Preload("Tags").
        Joins("LEFT JOIN comments ON comments.article_id = articles.id").
        Where("articles.user_id = ?", userID).
        Group("articles.id").
        Select("articles.*, COUNT(comments.id) as comment_count").
        Find(&articles)
        
    return articles, result.Error
}
```

## 6. DevOps et déploiement

### 6.1 Docker et Containerisation
```dockerfile
# Dockerfile multi-stage
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM alpine:latest
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /app/main .
COPY --from=builder /app/config.yaml .

EXPOSE 8080
CMD ["./main"]
```

### 6.2 Kubernetes Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: db_host
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
          requests:
            cpu: "200m"
            memory: "256Mi"
```

### 6.3 Monitoring et Logging
```go
package monitoring

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    responseTime = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_response_time_seconds",
            Help:    "Response time distribution",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        recorder := &statusRecorder{
            ResponseWriter: w,
            Status:        200,
        }
        
        next.ServeHTTP(recorder, r)
        
        duration := time.Since(start).Seconds()
        
        requestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            strconv.Itoa(recorder.Status),
        ).Inc()
        
        responseTime.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

// Structured Logging
type LogEntry struct {
    Level     string    `json:"level"`
    Timestamp time.Time `json:"timestamp"`
    Message   string    `json:"message"`
    RequestID string    `json:"request_id,omitempty"`
    Method    string    `json:"method,omitempty"`
    Path      string    `json:"path,omitempty"`
    Status    int       `json:"status,omitempty"`
    Duration  string    `json:"duration,omitempty"`
    Error     string    `json:"error,omitempty"`
}

func logRequest(w http.ResponseWriter, r *http.Request, start time.Time, status int, err error) {
    entry := LogEntry{
        Level:     "info",
        Timestamp: time.Now(),
        RequestID: r.Context().Value("requestID").(string),
        Method:    r.Method,
        Path:      r.URL.Path,
        Status:    status,
        Duration:  time.Since(start).String(),
    }
    
    if err != nil {
        entry.Level = "error"
        entry.Error = err.Error()
    }
    
    json.NewEncoder(os.Stdout).Encode(entry)
}
```

## 7. Intégration avec d'autres systèmes

### 7.1 gRPC Services
```go
// proto/service.proto
syntax = "proto3";
package service;

service UserService {
    rpc GetUser (UserRequest) returns (UserResponse) {}
    rpc CreateUser (CreateUserRequest) returns (UserResponse) {}
    rpc UpdateUser (UpdateUserRequest) returns (UserResponse) {}
}

message UserRequest {
    string id = 1;
}

message UserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
}

// Implementation
package main

import (
    "context"
    pb "myapp/proto"
)

type userServer struct {
    pb.UnimplementedUserServiceServer
}

func (s *userServer) GetUser(ctx context.Context, req *pb.UserRequest) (*pb.UserResponse, error) {
    // Implementation
    return &pb.UserResponse{
        Id:    req.Id,
        Name:  "John Doe",
        Email: "john@example.com",
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    
    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &userServer{})
    
    log.Printf("server listening at %v", lis.Addr())
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### 7.2 Message Queue Integration
```go
package queue

import (
    "github.com/streadway/amqp"
)

type RabbitMQ struct {
    conn    *amqp.Connection
    channel *amqp.Channel
}

func NewRabbitMQ(url string) (*RabbitMQ, error) {
    conn, err := amqp.Dial(url)
    if err != nil {
        return nil, err
    }
    
    ch, err := conn.Channel()
    if err != nil {
        return nil, err
    }
    
    return &RabbitMQ{
        conn:    conn,
        channel: ch,
    }, nil
}

func (r *RabbitMQ) PublishMessage(exchange, routingKey string, message []byte) error {
    return r.channel.Publish(
        exchange,
        routingKey,
        false,
        false,
        amqp.Publishing{
            ContentType: "application/json",
            Body:       message,
        },
    )
}

func (r *RabbitMQ) ConsumeMessages(queue string, handler func([]byte) error) error {
    msgs, err := r.channel.Consume(
        queue,
        "",
        true,
        false,
        false,
        false,
        nil,
    )
    if err != nil {
        return err
    }
    
    forever := make(chan bool)
    
    go func() {
        for d := range msgs {
            if err := handler(d.Body); err != nil {
                log.Printf("Error processing message: %v", err)
            }
        }
    }()
    
    log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
    <-forever
    
    return nil
}
```

### 7.3 Cache Redis
```go
package cache

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/go-redis/redis/v8"
)

type RedisCache struct {
    client *redis.Client
    ctx    context.Context
}

func NewRedisCache(addr string) (*RedisCache, error) {
    client := redis.NewClient(&redis.Options{
        Addr:     addr,
        Password: "",
        DB:       0,
    })
    
    ctx := context.Background()
    
    if err := client.Ping(ctx).Err(); err != nil {
        return nil, err
    }
    
    return &RedisCache{
        client: client,
        ctx:    ctx,
    }, nil
}

func (c *RedisCache) Set(key string, value interface{}, expiration time.Duration) error {
    json, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    return c.client.Set(c.ctx, key, json, expiration).Err()
}

func (c *RedisCache) Get(key string, dest interface{}) error {
    val, err := c.client.Get(c.ctx, key).Result()
    if err != nil {
        return err
    }
    
    return json.Unmarshal([]byte(val), dest)
}

func (c *RedisCache) Delete(key string) error {
    return c.client.Del(c.ctx, key).Err()
}

// Example d'utilisation du cache distribué
type UserCache struct {
    cache *RedisCache
}

func (uc *UserCache) GetUser(id string) (*User, error) {
    var user User
    
    // Essayer de récupérer depuis le cache
    err := uc.cache.Get("user:"+id, &user)
    if err == nil {
        return &user, nil
    }
    
    // Si pas dans le cache, récupérer depuis la DB
    user, err = getUserFromDB(id)
    if err != nil {
        return nil, err
    }
    
    // Mettre en cache pour les futures requêtes
    if err := uc.cache.Set("user:"+id, user, 1*time.Hour); err != nil {
        log.Printf("Error caching user: %v", err)
    }
    
    return &user, nil
}
```

Voilà la suite et fin de la partie 3 du cours. Cette dernière section couvre les aspects pratiques de l'intégration avec différents systèmes et services externes. Voulez-vous que je développe davantage un aspect particulier ?