---
hide:
  - toc
---
# Optimization

## Tools

- **django-silk**: A live profiling tool. When to use: Identifying inefficient SQL queries.
- **pyinstrument**: A call stack profiler. When to use: Pinpointing the exact line of Python code causing bottlenecks in complex business logic.
- **New Relic**: Production monitoring. When to use: Real-time APM and error tracking in a live environment, also provides profiling.
- **Locust**: Load testing. When to use: Simulating concurrent users to find breaking points and assess system stability.

## Query Optimization Best Practices

1. **Select Related**: Use `select_related` for foreign key and one-to-one relationships (joins at the SQL level).
2. **Prefetch Related**: Use `prefetch_related` for many-to-many and reverse-foreign key relationships (separate queries, joined in Python).
3. Retrieve only necessary fields to reduce database load and memory usage.

## Optimization Techniques

We implement several layers of optimization across the stack:

- **Database Indexing**: Ensuring UUID paths and frequently queried fields are indexed.
- **Query Optimization**: Using `select_related` and `prefetch_related` to solve N+1 structural problems.
- **Redis Cache**: Performance boost for high-traffic catalog data by caching response.
- **Background Processing**: Offloading heavy tasks (like email and payment reconciliation) to **Celery**.

## Load Testing with Locust

Run load tests to simulate user traffic:

```bash
locust -f locustfile.py
```

## Reports

### Locust Load Testing 

**Users - 500**
<iframe src="/assets/ramp_up_500_users_local.html" width="100%" height="500px"></iframe>

**Users - 1000**
<iframe src="/assets/hight_ramp_failure.html" width="100%" height="500px"></iframe>

### Pyinstrument

**Login API**
<iframe src="/assets/_api_v1_auth_login_ 1773741268.html" width="100%" height="500px"></iframe>

**Login Verify API**
<iframe src="/assets/_api_v1_auth_verify-login_ 1773741408.html" width="100%" height="500px"></iframe>

### Django Silk

<iframe src="/assets/silk-index.html" width="100%" height="500px"></iframe>
<iframe src="/assets/silk-request-detail_api_v1_tenant-games_.html" width="100%" height="500px"></iframe>
<iframe src="/assets/silk-SQL_api_v1_tenant-games_.html" width="100%" height="500px"></iframe>
