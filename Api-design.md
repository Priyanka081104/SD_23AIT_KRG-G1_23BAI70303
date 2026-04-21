# API Design: Food Delivery System
## Authentication
All APIs (except public endpoints) require JWT token in Authorization header
## Common Response Format

### Success Response
```json
{
    "success": true,
    "status_code": 200,
    "message": "Operation successful",
    "data": {},
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "uuid-trace-id"
}
```
### Error Respsonse
```json
{
    "success": false,
    "status_code": 400,
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid input parameters",
        "details": [
            "email must be valid",
            "password must be at least 8 characters"
        ]
    },
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "uuid-trace-id"
}
```
HTTP Status Codes
Status Code	Description	When Used
200	OK	Successful GET, PUT, PATCH requests
201	Created	Successful POST requests (new resource created)
204	No Content	Successful DELETE requests
400	Bad Request	Invalid input, missing required fields
401	Unauthorized	Missing or invalid authentication token
403	Forbidden	Authenticated but not authorized
404	Not Found	Resource doesn't exist
409	Conflict	Duplicate resource (e.g., email already exists)
422	Unprocessable Entity	Business logic violation
429	Too Many Requests	Rate limit exceeded
500	Internal Server Error	Server-side error
503	Service Unavailable	Server under maintenance/overloaded
