
## 2XX – Success

200 OK: The request succeeded (for GET requests)
201 Created: A new resource was successfully created (for POST requests)
204 No Content: The request succeeded, but there’s nothing to return (often used for DELETE)

## 4XX – Client Errors

400 Bad Request: The client sent something invalid (malformed JSON, invalid parameters, etc.)
401 Unauthorized: Authentication is required but wasn’t provided
403 Forbidden: The client is authenticated but doesn’t have permission
404 Not Found: The requested resource doesn’t exist
409 Conflict: The request conflicts with the current state (e.g., creating a duplicate resource)
422 Unprocessable Entity: The request was well-formed but had semantic errors

## 5XX – Server Errors

500 Internal Server Error: Something unexpected went wrong on the server
502 Bad Gateway: An upstream service returned an invalid response
503 Service Unavailable: The server is temporarily unavailable (maintenance, overloaded)

Ref:  https://www.c-sharpcorner.com/article/beyond-500-building-professional-error-handling-for-net-apis/ 