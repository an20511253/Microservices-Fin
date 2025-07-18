# Microservices-Fin
Microservices and API End Points

# Financial Microservices API (Spring Boot)

This repository contains two independently deployable microservices built using **Java Spring Boot**:

1. **Data Collection Service**: Collects and processes financial data.
2. **Credit Scoring Service**: Calculates and caches user credit scores.

Authentication is handled via **Token-based Security** (e.g., JWT).

---


## 1. Data Collection Service

### 🔐 Token Security
Every endpoint must be accessed with a valid token passed in the `Authorization` header.
Use this dummy token for testing: `Bearer dummy-auth-token`

### 📌 Endpoints & Implementation

#### `GET /data/{userId}`
// Marks this class as a REST controller, which means it will handle HTTP requests and return JSON responses
@RestController
// Sets the base URL path for all endpoints in this controller to "/data"
@RequestMapping("/data")
public class DataController {
```java
    // Automatically injects an instance of DataService into this controller
    @Autowired
    private DataService dataService;

    // Maps HTTP GET requests to "/data/{userId}" to this method
    @GetMapping("/{userId}")
    public ResponseEntity<List<FinancialData>> getAllData(
            // Binds the value of {userId} from the URL to this method parameter
            @PathVariable String userId,

            // Extracts the "Authorization" header from the HTTP request
            @RequestHeader("Authorization") String token) {

        // Validates the token; if it doesn't match the expected dummy token, return 401 Unauthorized
        if (!token.equals("Bearer dummy-auth-token")) 
            return ResponseEntity.status(401).build();

        // If token is valid, fetch all financial data for the user and return it with 200 OK status
        return ResponseEntity.ok(dataService.getAllFinancialData(userId));
    }
}
```

#### `GET /data/{userId}/transactions`
```java
// Maps HTTP GET requests to "/{userId}/transactions" to this method
@GetMapping("/{userId}/transactions")
public ResponseEntity<List<Transaction>> getTransactions(
        // Binds the value of {userId} from the URL to this method parameter
        @PathVariable String userId,

        // Extracts the "Authorization" header from the HTTP request
        @RequestHeader("Authorization") String token) {

    // Validates the token; if it doesn't match the expected dummy token, return 401 Unauthorized
    if (!token.equals("Bearer dummy-auth-token")) 
        return ResponseEntity.status(401).build();

    // If token is valid, fetch the transaction history for the user and return it with 200 OK
    return ResponseEntity.ok(dataService.getTransactionHistory(userId));
}

```

#### `POST /data/{userId}`
```java
    @PostMapping("/{userId}")
    public ResponseEntity<String> postData(
            @PathVariable String userId,
            @RequestBody FinancialData data,
            @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        dataService.saveData(userId, data);
        return ResponseEntity.ok("Data submitted successfully.");
    }
```

#### `GET /data/{userId}/loans`
```java
    @GetMapping("/{userId}/loans")
    public ResponseEntity<List<Loan>> getLoans(
            @PathVariable String userId,
            @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        return ResponseEntity.ok(dataService.getLoans(userId));
    }
}
```

---

## 2. Credit Scoring Service

### 🚀 Redis Caching
Redis is used to cache credit scores and minimize recalculations.

### 📌 Endpoints & Implementation

```java
@RestController
@RequestMapping("/score")
public class ScoreController {

    @Autowired
    private ScoreService scoreService;

    @Autowired
    private RedisTemplate<String, Integer> redisTemplate;
```

#### `GET /score/{userId}`
```java
// Maps HTTP GET requests to "/{userId}" to this method
@GetMapping("/{userId}")
public ResponseEntity<Integer> getScore(
        // Binds the value of {userId} from the URL to this method parameter
        @PathVariable String userId,

        // Extracts the "Authorization" header from the HTTP request
        @RequestHeader("Authorization") String token) {

    // Validates the token; if it doesn't match the expected dummy token, return 401 Unauthorized
    if (!token.equals("Bearer dummy-auth-token")) 
        return ResponseEntity.status(401).build();

    // Constructs the Redis key using the userId
    String redisKey = "score:" + userId;

    // Attempts to retrieve the score from Redis cache
    Integer score = redisTemplate.opsForValue().get(redisKey);

    // If the score is not found in Redis (i.e., cache miss)
    if (score == null) {
        // Calculate the score using the scoreService
        score = scoreService.calculateScore(userId);

        // Store the newly calculated score in Redis for future requests
        redisTemplate.opsForValue().set(redisKey, score);
    }

    // Return the score with a 200 OK response
    return ResponseEntity.ok(score);
}

```

#### `POST /score/calculate`
```java
// Maps HTTP POST requests to "/calculate" to this method
@PostMapping("/calculate")
public ResponseEntity<Integer> calculateScore(
        // Binds the request body JSON to a ScoreRequest object
        @RequestBody ScoreRequest req,

        // Extracts the "Authorization" header from the HTTP request
        @RequestHeader("Authorization") String token) {

    // Validates the token; if it doesn't match the expected dummy token, return 401 Unauthorized
    if (!token.equals("Bearer dummy-auth-token")) 
        return ResponseEntity.status(401).build();

    // Calls the scoreService to calculate the score for the given userId
    int score = scoreService.calculateScore(req.getUserId());

    // Stores the calculated score in Redis with a key formatted as "score:{userId}"
    redisTemplate.opsForValue().set("score:" + req.getUserId(), score);

    // Returns the calculated score with a 200 OK response
    return ResponseEntity.ok(score);
}

```

#### `PUT /score/{userId}`
```java
    @PutMapping("/{userId}")
    public ResponseEntity<String> updateScore(@PathVariable String userId,
                                              @RequestBody int newScore,
                                              @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        scoreService.updateScore(userId, newScore);
        redisTemplate.opsForValue().set("score:" + userId, newScore);
        return ResponseEntity.ok("Score updated.");
    }
```

#### `GET /score/history/{userId}`
```java
    @GetMapping("/history/{userId}")
    public ResponseEntity<List<Integer>> getScoreHistory(@PathVariable String userId,
                                                         @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        return ResponseEntity.ok(scoreService.getScoreHistory(userId));
    }
}
```

---

## 🧱 Tech Stack
- Java 17
- Spring Boot 3.x
- Spring Security (JWT/Token Auth)
- Redis
- Maven

## 🚀 Getting Started
- Clone the repo
- Configure `application.properties` (DB, Redis, etc.)
- Run each service using `mvn spring-boot:run`

---

## 📁 Project Structure
Each microservice is organized independently with:
- `controller/`
- `service/`
- `model/`
- `repository/`

---

## 📫 Contributions
Pull requests welcome for enhancements and bug fixes.

---

## 📄 License
[MIT](LICENSE)
