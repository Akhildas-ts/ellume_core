# 🔢 The 10-Step Flow (Very Simple)

1.  **User sends:** `POST /v1/users`
2.  The server receives the request.
3.  **Global middleware** runs (CORS, logging, panic recovery).
4.  **Auth middleware** checks JWT and confirms the caller is an admin.
5.  The request body is decoded into `CreateUserRequest`.
6.  Endpoint wrapper starts timing and logging for monitoring.
7.  **Business logic runs** (this is the main part):
    *   **7A:** Check if the email already exists in the SQL database.
    *   **7B:** Create the user in Firebase (Auth DB).
    *   **7C:** Save the user record in SQL (App DB).
8.  Repository builds the SQL insert query.
9.  SQL database executes the `INSERT`.
10. The created user is returned to the client.

---

## 🔍 Dual Database Explained (Step 7)

### 7A — Check Email in SQL DB
SQL checks if the user already exists:

```sql
SELECT * FROM users WHERE email = 'john@example.com';
```

### 7B — Create User in Firebase
Firebase is your **Authentication Database**, so you create the user there:

```json
{
  "email": "john@example.com",
  "displayName": "John Doe"
}
```

Firebase returns:

```text
uid = "xyz123abc"
```

### 7C — Save User in SQL DB
Since Firebase only stores authentication info, you save user details in your **Application Database**:

**SQLite example:**
```sql
INSERT INTO users (...)
```

**PostgreSQL example:**
```sql
INSERT INTO users VALUES ($1, $2, ...)
```

---

## 💾 Final Result (Two Databases Updated)

| Feature | Firebase (Auth DB) | SQL DB (App DB) |
| :--- | :--- | :--- |
| **Stores** | • `uid`<br>• `email`<br>• `displayName` | • `id`<br>• `name`<br>• `email`<br>• `roles`<br>• `created_at` |
| **Used For** | • Login<br>• JWT tokens<br>• Passwords | • App data<br>• User lists<br>• Dashboards<br>• Relationships |

---

## 🎯 Simple Summary

*   **Firebase** = authentication storage
*   **SQL DB** = your application data
*   **Business Logic** updates both in correct order:
    `Check → Create in Firebase → Save in SQL`
*   SQLite/Postgres SQL differs slightly but logic is same.
