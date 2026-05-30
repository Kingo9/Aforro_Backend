# Aforro Backend Assignment

A production-oriented backend service built with Django REST Framework, PostgreSQL, Redis, Celery, and Docker.

This project implements an inventory and order management system with product search, autocomplete, asynchronous task processing, rate limiting, and containerized deployment.

---

## Tech Stack

* Python 3.12
* Django 5
* Django REST Framework
* PostgreSQL
* Redis
* Celery
* Docker & Docker Compose

---

## Features

### Product Management

* Categories and Products
* Product listing and filtering
* Category-based organization

### Store & Inventory Management

* Multiple stores
* Store-specific inventory
* Inventory listing endpoint
* Inventory consistency enforcement

### Order Processing

* Atomic order creation
* Inventory validation
* Stock deduction
* Order status management

### Product Search

* Keyword search
* Category filtering
* Price range filtering
* Store-specific inventory lookup
* Pagination support
* Sorting by:

  * Price
  * Newest
  * Relevance

### Autocomplete

* Fast product suggestions
* Prefix matching prioritization
* Redis-based rate limiting

### Asynchronous Processing

* Celery integration
* Redis message broker
* Background order confirmation task

### Containerized Deployment

* Dockerized services
* PostgreSQL container
* Redis container
* Celery worker container

---

## Project Structure

```text
project/
│
├── apps/
│   ├── products/
│   ├── stores/
│   ├── orders/
│   └── search/
│
├── tests/
│
├── project/
│   ├── settings.py
│   ├── urls.py
│   └── celery.py
│
├── manage.py
├── requirements.txt
├── docker-compose.yml
└── README.md
```

---

## Project Flow

### Order Creation Flow

Client
→ POST /api/orders/
→ Validate Request
→ Validate Store
→ Validate Products
→ Start Database Transaction
→ Lock Inventory Rows
→ Check Stock Availability

If stock is sufficient:

→ Deduct Inventory
→ Create Order (CONFIRMED)
→ Create Order Items
→ Trigger Celery Task
→ Return Response

Else:

→ Create Order (REJECTED)
→ No Inventory Deduction
→ Return Response

### Search Flow

Client
→ GET /api/search/products/
→ Apply Keyword Search
→ Apply Filters
→ Apply Sorting
→ Paginate Results
→ Return Response

### Autocomplete Flow

Client
→ GET /api/search/suggest/
→ Redis Rate Limit Check
→ Validate Query Length
→ Search Product Titles
→ Prioritize Prefix Matches
→ Return Top 10 Suggestions


## Running with Docker

### Build and Start

```bash
docker compose up --build
```

### Run in Detached Mode

```bash
docker compose up -d --build
```

### Stop Containers

```bash
docker compose down
```

---

## Seed Data

Generate sample data:

```bash
docker compose exec web python manage.py seed_data
```

Creates:

* 10+ Categories
* 1000+ Products
* 20+ Stores
* Inventory records for stores

---

## API Endpoints

### Categories

```http
GET /api/categories/
GET /api/categories/{id}/
```

---

### Products

```http
GET /api/products/
GET /api/products/{id}/
```

Optional filtering:

```http
GET /api/products/?category=1
```

---

### Stores

```http
GET /api/stores/
GET /api/stores/{id}/
```

---

### Store Inventory

```http
GET /api/stores/{store_id}/inventory/
```

Response:

```json
[
  {
    "product_title": "Laptop",
    "price": "50000.00",
    "category": "Electronics",
    "quantity": 15
  }
]
```

---

### Store Orders

```http
GET /api/stores/{store_id}/orders/
```

Returns:

* Order ID
* Status
* Created Timestamp
* Total Items

---

### Create Order

```http
POST /api/orders/
```

Request:

```json
{
  "store_id": 1,
  "items": [
    {
      "product_id": 10,
      "quantity_requested": 2
    },
    {
      "product_id": 25,
      "quantity_requested": 1
    }
  ]
}
```

Success Response:

```json
{
  "id": 1,
  "status": "CONFIRMED"
}
```

Rejected Response:

```json
{
  "id": 2,
  "status": "REJECTED"
}
```

---

### Product Search

```http
GET /api/search/products/
```

Supported Parameters:

```text
q
category
min_price
max_price
store_id
in_stock
sort
page
```

Examples:

```http
GET /api/search/products/?q=laptop
```

```http
GET /api/search/products/?category=2
```

```http
GET /api/search/products/?store_id=1&in_stock=true
```

```http
GET /api/search/products/?sort=price
```

---

### Autocomplete

```http
GET /api/search/suggest/?q=lap
```

Response:

```json
{
  "suggestions": [
    "Laptop",
    "Laptop Stand",
    "Laptop Bag"
  ]
}
```

Requirements:

* Minimum 3 characters
* Maximum 10 results
* Prefix matches prioritized

---

## Database Consistency

Order creation uses:

```python
transaction.atomic()
```

to ensure all database operations succeed or fail together.

Inventory rows are locked using:

```python
select_for_update()
```

to prevent overselling during concurrent order requests.

Inventory updates use:

```python
F()
```

expressions to avoid race conditions.

---

## Redis Usage

Redis is used for request rate limiting.

Applied to:

```http
GET /api/search/suggest/
```

Limit:

```text
20 requests per minute per IP
```

Benefits:

* Prevents abuse
* Reduces server load
* Maintains responsiveness

---

## Celery Workflow

Order confirmation tasks are processed asynchronously.

Flow:

```text
Order Created
      │
      ▼
Celery Task Triggered
      │
      ▼
Redis Broker
      │
      ▼
Celery Worker
      │
      ▼
Background Processing
```

Run worker:

```bash
docker compose up celery
```

---

## Testing

Run tests:

```bash
docker compose exec web pytest
```

or

```bash
docker compose exec web python manage.py test
```

Covered areas:

* Order creation
* Inventory deduction
* Search functionality
* API responses
* Validation logic

---

## Assumptions & Design Decisions

### Database Consistency

Order creation is wrapped inside `transaction.atomic()` to ensure all inventory updates and order records are committed together or rolled back entirely.

### Inventory Locking

`select_for_update()` is used during order creation to prevent concurrent stock modifications and overselling.

### Search Implementation

Product search uses Django ORM filtering with `Q()` objects across:

* Product title
* Product description
* Category name

This approach was chosen for simplicity and maintainability while satisfying assignment requirements.

### Rate Limiting

Redis-backed rate limiting is applied to the autocomplete endpoint to prevent abuse and reduce unnecessary database load.

### Asynchronous Processing

Celery is used with Redis as the message broker to process order confirmation tasks asynchronously without blocking API responses.

### Dockerized Development

All services are containerized and started using Docker Compose:

* Django API
* PostgreSQL
* Redis
* Celery Worker

This ensures consistent local development and deployment environments.

### Query Optimization

The project uses:

* `select_related()`
* `prefetch_related()`
* `bulk_create()`
* `in_bulk()`

to minimize database queries and avoid N+1 query issues.


## Scalability Considerations

### Current Optimizations

* select_related()
* prefetch_related()
* bulk_create()
* in_bulk()
* transaction.atomic()
* select_for_update()
* Redis rate limiting

### Future Improvements

* Elasticsearch integration
* Read replicas
* Product search indexing
* Distributed caching
* API authentication
* Background analytics jobs
* Monitoring with Prometheus and Grafana

---

## Design Decisions

### Why PostgreSQL?

Reliable transactional database with strong support for relational data and indexing.

### Why Redis?

Low-latency in-memory storage ideal for rate limiting and task brokering.

### Why Celery?

Decouples background processing from request-response cycles.

### Why Docker?

Ensures consistent local and deployment environments.

---

## Author

Asad Shaikh

Backend Developer Assignment Submission
