# 📘 Go-Kit Microservice Documentation (Beginner Friendly)

**For:** akhil — Ellume Core Project

---

## 1. What Go-Kit Actually Is (Simple Explanation)

Go-Kit is an architecture framework that separates your code into layers:

**Transport → Endpoint → Business Logic**

### Meaning:

*   **Transport** = HTTP layer (decode request, encode response)
*   **Endpoint** = Wrap business logic into a standard function
*   **Business Logic (BL)** = Real work (DB calls, rules)
*   **Spec** = Shared request/response structures

This makes big projects cleaner and easy to maintain.

> **You don’t write Go-Kit logic — Go-Kit does the heavy lifting.**

---

## 🧠 2. Go-Kit Execution Flow (THE MOST IMPORTANT PART)

When an HTTP request comes:

1.  **Middleware** runs (auth, logging)
2.  **Decoder** converts JSON → Go struct
3.  **Endpoint** runs your business logic
4.  **Encoder** converts Go struct → JSON response

> **You don't control this order. Go-Kit controls it.**
> You just provide components.

---

## 🗂 3. Project File Structure (The SIMPLIFIED Version)

Your user service is split like this:

```text
📁 spec/user/
    ├── request_response.go   <-- Structs: Request / Response
    └── endpoint.go           <-- Big endpoint struct listing all APIs

📁 module/user/
    ├── bl.go                 <-- Business Logic (actual work)
    ├── endpoint.go           <-- Wrap BL into Go-Kit endpoints
    └── transport.go          <-- HTTP handlers (decode, encode, route)
```

That’s it.

### Every new API touches the SAME 6 places:

| File | What you add |
| :--- | :--- |
| `spec/user/request_response.go` | Request/Response struct |
| `module/user/bl.go` | Business Logic method |
| `module/user/endpoint.go` | `Make<API>Endpoint` wrapper |
| `spec/user/endpoint.go` | Add endpoint field to struct |
| `module/user/transport.go` | Decoder function |
| `module/user/transport.go` | Route mapping |

---

## 💡 4. Why the Code Looks Complex (Truth)

Because Go-Kit separates layers clearly.

But **you don’t write complex logic.**

You always follow a simple **COPY-PASTE** pattern.

> 90% of your code when adding API = change names.

---

## 🚀 5. How to Create a New API (Super Simple 6-Step Template)

Let’s say you want: `GET /v1/users/stats`

### ✅ Step 1: Add Request/Response structs

**File:** `spec/user/request_response.go`

```go
type GetUserStatsRequest struct {
    UserID string `json:"user_id"`
}

type GetUserStatsResponse struct {
    Posts     int `json:"posts"`
    Followers int `json:"followers"`
}
```

### ✅ Step 2: Add business logic

**File:** `module/user/bl.go`

```go
func (b *BL) GetUserStats(ctx context.Context, req *spec.GetUserStatsRequest) (*spec.GetUserStatsResponse, error) {
    // DB queries here
    return &spec.GetUserStatsResponse{
        Posts: 20,
        Followers: 300,
    }, nil
}
```

### ✅ Step 3: Wrap business logic in endpoint

**File:** `module/user/endpoint.go`

```go
func MakeGetUserStatsEndpoint(bl *BL) endpoint.Endpoint {
    return func(ctx context.Context, request interface{}) (interface{}, error) {
        req := request.(*spec.GetUserStatsRequest)
        return bl.GetUserStats(ctx, req)
    }
}
```

And add it to the endpoint set (top of file):

```go
GetUserStatsEndpoint endpoint.Endpoint
```

### ✅ Step 4: Add decoder

**File:** `module/user/transport.go`

```go
func decodeGetUserStatsRequest(_ context.Context, r *http.Request) (interface{}, error) {
    id := chi.URLParam(r, "id")
    return &spec.GetUserStatsRequest{UserID: id}, nil
}
```

### ✅ Step 5: Register route in MakeHTTPHandler

**File:** `module/user/transport.go`

```go
r.Get("/v1/users/{id}/stats", httptransport.NewServer(
    endpoints.GetUserStatsEndpoint,
    decodeGetUserStatsRequest,
    httptransport.EncodeJSONResponse,
))
```

### ✅ Step 6: Add field in endpoint struct

**File:** `spec/user/endpoint.go`

```go
GetUserStatsEndpoint endpoint.Endpoint
```

---

## 🧩 6. Why This Architecture Is Used

Go-Kit helps in:

*   ✔ Clean separation of concerns
*   ✔ Testability
*   ✔ Structured layering
*   ✔ Easy scaling
*   ✔ Easy swapping transports (HTTP, gRPC, events)

**Your job is only the business logic.**
The architecture does the rest.

---

## 🧱 7. Mental Model (IMPORTANT)

Don’t think:
> "How do I build APIs from scratch?"

Instead think:
> "Which 6 places do I copy-paste from?"

After creating 2–3 APIs, you will master it.

---

## 🎯 8. Super-Simple Summary

Here is the final cheat sheet:

**NEW API = SAME 6 STEPS EVERY TIME**

1.  `spec/user/request_response.go` → **structs**
2.  `module/user/bl.go` → **business logic**
3.  `module/user/endpoint.go` → **MakeXEndpoint**
4.  `spec/user/endpoint.go` → **add field**
5.  `module/user/transport.go` → **decoder**
6.  `module/user/transport.go` → **route**

That’s Go-Kit.
**Simple. Predictable. Repeatable.**