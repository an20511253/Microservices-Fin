// ============================
// README.md (GitHub Summary)
// ============================

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
Retrieve all financial data for a specific user.
```java
@RestController
@RequestMapping("/data")
public class DataController {

    @Autowired
    private DataService dataService;

    @GetMapping("/{userId}")
    public ResponseEntity<List<FinancialData>> getAllData(
            @PathVariable String userId,
            @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        return ResponseEntity.ok(dataService.getAllFinancialData(userId));
    }
```

#### `GET /data/{userId}/transactions`
```java
    @GetMapping("/{userId}/transactions")
    public ResponseEntity<List<Transaction>> getTransactions(
            @PathVariable String userId,
            @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
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

### 📂 Files (Data Collection Service)

#### `model/FinancialData.java`
```java
public class FinancialData {
    private String type;
    private double amount;
    private LocalDate date;
    // getters and setters
}
```

#### `model/Transaction.java`
```java
public class Transaction {
    private String description;
    private double amount;
    private LocalDate date;
    // getters and setters
}
```

#### `model/Loan.java`
```java
public class Loan {
    private String loanId;
    private double balance;
    private String status;
    // getters and setters
}
```

#### `repository/DataRepository.java`
```java
@Repository
public class DataRepository {
    // Dummy methods for demo
    public List<FinancialData> findAllData(String userId) { return new ArrayList<>(); }
    public List<Transaction> findTransactions(String userId) { return new ArrayList<>(); }
    public void save(String userId, FinancialData data) {}
    public List<Loan> findLoans(String userId) { return new ArrayList<>(); }
}
```

#### `service/DataService.java`
```java
@Service
public class DataService {

    @Autowired
    private DataRepository dataRepository;

    public List<FinancialData> getAllFinancialData(String userId) {
        return dataRepository.findAllData(userId);
    }

    public List<Transaction> getTransactionHistory(String userId) {
        return dataRepository.findTransactions(userId);
    }

    public void saveData(String userId, FinancialData data) {
        dataRepository.save(userId, data);
    }

    public List<Loan> getLoans(String userId) {
        return dataRepository.findLoans(userId);
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
    @GetMapping("/{userId}")
    public ResponseEntity<Integer> getScore(@PathVariable String userId,
                                            @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        String redisKey = "score:" + userId;
        Integer score = redisTemplate.opsForValue().get(redisKey);
        if (score == null) {
            score = scoreService.calculateScore(userId);
            redisTemplate.opsForValue().set(redisKey, score);
        }
        return ResponseEntity.ok(score);
    }
```

#### `POST /score/calculate`
```java
    @PostMapping("/calculate")
    public ResponseEntity<Integer> calculateScore(@RequestBody ScoreRequest req,
                                                  @RequestHeader("Authorization") String token) {
        if (!token.equals("Bearer dummy-auth-token")) return ResponseEntity.status(401).build();
        int score = scoreService.calculateScore(req.getUserId());
        redisTemplate.opsForValue().set("score:" + req.getUserId(), score);
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

### 📂 Files (Credit Scoring Service)

#### `model/ScoreRequest.java`
```java
public class ScoreRequest {
    private String userId;
    // getters and setters
}
```

#### `repository/ScoreRepository.java`
```java
@Repository
public class ScoreRepository {
    private final Map<String, List<Integer>> userScores = new HashMap<>();

    public void save(String userId, int score) {
        userScores.computeIfAbsent(userId, k -> new ArrayList<>()).add(score);
    }

    public List<Integer> getHistory(String userId) {
        return userScores.getOrDefault(userId, new ArrayList<>());
    }
}
```

#### `service/ScoreService.java`
```java
@Service
public class ScoreService {

    @Autowired
    private ScoreRepository scoreRepository;

    public int calculateScore(String userId) {
        int score = (int) (Math.random() * 300 + 500);
        scoreRepository.save(userId, score);
        return score;
    }

    public void updateScore(String userId, int score) {
        scoreRepository.save(userId, score);
    }

    public List<Integer> getScoreHistory(String userId) {
        return scoreRepository.getHistory(userId);
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
