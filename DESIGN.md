# URL Shortener - System Design Document

## 1. Requirements

### Functional Requirements
- Users can submit a long URL and receive a shortened URL
- Visiting the short URL redirects to the original URL
- Users can view analytics (hit count) for a short URL

### Non-Functional Requirements
- Low latency for redirects (< 50ms with cache hit)
- High read-to-write ratio (100:1 estimated)
- Short codes should be unique and non-sequential (Base62 encoding)
- System should handle concurrent requests safely

## 2. API Contract

### POST /api/shorten
Request:
```json
{ "url": "https://example.com/very/long/url" }
```
Response (201 Created):
```json
{ "shortCode": "abc123", "shortUrl": "http://localhost:8080/abc123", "originalUrl": "https://example.com/very/long/url" }
```

### GET /{shortCode}
Response: 302 Redirect to original URL
Error: 404 Not Found if short code does not exist

### GET /api/stats/{shortCode}
Response (200 OK):
```json
{ "shortCode": "abc123", "originalUrl": "https://example.com/very/long/url", "hitCount": 42, "createdAt": "2026-05-29T10:00:00" }
```

## 3. Data Model

### urls table
| Column       | Type         | Constraints          |
|-------------|--------------|---------------------|
| id          | BIGINT       | PRIMARY KEY, AUTO    |
| short_code  | VARCHAR(10)  | UNIQUE, NOT NULL     |
| original_url| VARCHAR(2048)| NOT NULL             |
| hit_count   | BIGINT       | DEFAULT 0            |
| created_at  | TIMESTAMP    | NOT NULL             |


## 4. Architecture

```
Client -> [Spring Boot API] -> [Redis Cache]
                            -> [PostgreSQL]
```

- **API Layer**: Spring Boot handles HTTP requests, validation, and business logic
- **Cache Layer**: Redis stores shortCode -> originalUrl mappings for fast reads
- **Storage Layer**: PostgreSQL persists all URL data permanently

### Flow: Redirect (GET /{shortCode})
1. Check Redis cache for shortCode
2. If cache hit: return URL, increment hit count async
3. If cache miss: query PostgreSQL, populate cache, return URL

### Flow: Shorten (POST /api/shorten)
1. Validate URL format
2. Save to PostgreSQL (generates ID)
3. Encode ID to Base62 short code
4. Update record with short code
5. Write to Redis cache
6. Return response

## 5. Scalability Considerations

- **Horizontal scaling**: Stateless API layer can scale behind a load balancer
- **Redis cluster**: Distribute cache across nodes for higher throughput
- **Read replicas**: PostgreSQL read replicas for analytics queries
- **CDN**: Short URL redirects could be cached at CDN edge nodes
- **CAP trade-off**: Favoring AP (Availability + Partition Tolerance). A stale cache entry is acceptable briefly.