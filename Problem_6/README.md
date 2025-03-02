<!-- PROJECT LOGO -->
<br />
<div align="center">
  <h1 align="center">Scoreboard API Module Specification</h3>
</div>

<!-- ABOUT THE PROJECT -->
## About The Module
This document specifies the requirements for a backend API module responsible for managing the top 10 user's scores and updating the scoreboard in real-time. This module will be integrated into the 
existing website to provide a dynamic and secure scorekeeping system.

## Module Requirement
### Functional requirement:
- **Score Storage:** Persist the scores of all users in a primary database.
- **Live Leaderboard:** Provide a live-updating leaderboard displaying the top 10 users and their scores.
- **Score Update:** Update user scores when receiving authorized API calls from the website.
- **Security:** Prevent unauthorized score manipulation by:
  - Extracting the user identity from a validated JWT.
  - Computing score increments on the server side based on the user’s action.
  - Enforcing rate limiting on score update requests.
### Non-functional requirement:
- **Performance:**
  - The leaderboard must load in 2 seconds or less.
  - Score updates should be applied in real-time with minimal latency.
- **Security:**
  - Secure all API endpoints, especially the score update API.
  - Implement JWT authentication and middleware for role-based or attribute-based access control.
  - Apply rate limiting.

## Technology Stack
### Real-Time Updates
 Depending on the total number of users, we can choose between using WebSocket or Kafka to achieve real-time updating for the scoreboard. Here, I recommend using a web socket for three main reasons:
 - WebSockets are easier to implement and require less infrastructure than Kafka.
 - The number of users is limited,so Kafka may be overkill for this module.
 - WebSockets deliver updates only to connected clients interested in the scoreboard. With Kafka, messages might be sent to a broader audience depending on the topic configuration, potentially creating unnecessary processing.
### Data Storage and Caching
#### Primary Database: 
 A relational database to store persistent user score data.
#### Caching
 Use a cached database (Redis) to quickly query and manage the top 10 leaderboard.
 - Redis Sorted Sets: Use Redis sorted sets (ZADD, ZRANGE,...) for efficient ranking.
 - Cache Consistency: Maintain consistency between Redis and the primary database using event-driven updates and periodic background refresh jobs.

## Database design
### User Scores Table:
- This table stores individual user scores and related data.
- Columns include:
  - user_id (unique identifier for the user)
  - score (current score of the user)
  - updated_at (timestamp of the last score update)
  - created_at (timestamp of the score created)
### Index on Score column:
Here, to optimize the query, we need to create a single index on the score column so that it will be faster when we want to get the top 10 list.

### Diagram 1: Database ERD
<br />
<div align="center">
  <img src="images/erdDatabase.png" alt="Logo" width="450">
  <p>Diagram 1. Illustration database ERD</p>
</div>

## Authentication and Authorization:
- **JWT Authentication:** All API requests must include a valid JSON Web Token (JWT) in the authorization header.
  - The token is parsed to extract the user_id and validate permissions.
- **Middleware:**
  - Implement middleware at the gateway to parse the JWT and forward the authenticated user information to the service.
- **Server-Side Validation:**
  - Do not accept user_id in the request payload; extract it from the JWT.
  - Compute the score increment (score_delta) based on a validated action_type provided by the client, ensuring that only authorized actions can update scores.

## API Endpoints:
### 1. GET /api/v1/scores/leaderboard
- Description: Retrieves the top 10 user scores and their corresponding user IDs.
- Success Response:
   ```
  {
    "data": [
      {
        "rank": 1,
        "user_id": 11,
        "score": 900
      },
      {
        "rank": 2,
        "user_id": 13,
        "score": 856
      },
      // ... (other top 10 entries)
    ]
  }
   ```
- Error Response: Refer to the error codes section below.

### 2. POST /api/v1/scores/update
- Description: Updates a user's score upon completion of an action.
- Request Body:
  ```
  {
    "action_type": "win_game"
  }
   ```
- Success Response:
   ```
  {
    "message": "Score updated successfully"
  }
   ```
- Error Response: Refer to the error codes section below.

## Error Codes:
The following error codes might be returned in the response body:

- **400 Bad Request:** Invalid request body format or missing required parameters.
- **401 Unauthorized:** Invalid or missing authorization token.
- **403 Forbidden:** The user does not have permission to access the requested resource.
- **404 Not Found:** Resource not found (e.g., a user with the provided ID doesn't exist).
- **500 Internal Server Error:** Unexpected server error occurred.


## Score Update Workflow:
### Diagram 2: Score Update Workflow
<br />
<div align="center">
  <img src="images/workflow.png" alt="Logo" width="1000">
  <p>Diagram 2. Illustration workflow of score updation</p>
</div>

1. **User Action:** A user action triggers a score update API call.
2. **Authentication:** The request passes through JWT validation middleware.
3. **Server Processing:** The server validates the action, computes the score increment, and updates the user_scores table.
4. **Leaderboard Update:** The system checks if the new score qualifies for the top 10 and updates Redis using a sorted set.
5. **Real-Time Notification:** If the leaderboard changes, a WebSocket broadcast is sent to all connected clients.

### Process Description:
#### 1. User Score Update:
- A POST request is sent to /api/v1/scores/update with an action_type ("win_game", "lose_game").
- The server extracts the user identity from the JWT and validates the action.
- The user’s score is updated in the user_scores table based on server-side computed score_delta.
#### 2. Redis Leaderboard Update:
- The system verifies if the new score qualifies for the top 10 by comparing it with the current lowest score in Redis.
- If it qualifies, update the Redis leaderboard immediately using a sorted set.
- If it does not qualify, no change is made to the Redis cache.
#### 3. Cache Consistency:
- Event-Driven Updates: Immediately update Redis when a score change affects the leaderboard.
- Background Refresh Job: Periodically refresh the leaderboard from the primary database before the Redis cache expires.
- Set a short expiration time for the Redis cache to minimize stale data.
#### 4. Real-Time Notifications:
- Use WebSocket to broadcast leaderboard changes.
- Optionally, implement a Pub/Sub mechanism (e.g., via Redis) to send updates only to clients whose leaderboard view may be affected.

## Additional Considerations and Improvements
### - Rate Limiting:
- Implement rate limiting on the /api/v1/scores/update endpoint to prevent abuse (e.g., limit to X requests per minute per user).
### - Logging and Monitoring:
- Log all score update events for auditing and debugging.
- Monitor API performance and WebSocket connections to ensure the system is healthy.
### - Scalability:
- Design the system to extend beyond the top 10 leaderboard easily (e.g., top 1000) without major refactoring.
### - Testing:
- Develop unit tests, integration tests, and load tests to ensure the module handles high update volumes and remains secure.
### - Documentation:
- Maintain and update this documentation with version control. Ensure that all diagram changes and design updates are reflected accordingly.
  

