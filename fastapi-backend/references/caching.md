# Caching

Redis caching strategies, invalidation patterns, and cache-aside implementation for FastAPI.

## Redis cache setup

```python
# app/core/cache.py
import json
from typing import Any
from app.core.redis import get_redis
from app.core.logging import get_logger

logger = get_logger(__name__)

class Cache:
    def __init__(self, prefix: str, default_ttl: int = 300):
        self.prefix = prefix
        self.default_ttl = default_ttl

    def _key(self, key: str) -> str:
        return f"{self.prefix}:{key}"

    async def get(self, key: str) -> Any | None:
        redis = await get_redis()
        data = await redis.get(self._key(key))
        if data:
            logger.debug("Cache hit", extra={"key": self._key(key)})
            return json.loads(data)
        logger.debug("Cache miss", extra={"key": self._key(key)})
        return None

    async def set(self, key: str, value: Any, ttl: int | None = None) -> None:
        redis = await get_redis()
        await redis.setex(
            self._key(key),
            ttl or self.default_ttl,
            json.dumps(value, default=str),
        )

    async def delete(self, key: str) -> None:
        redis = await get_redis()
        await redis.delete(self._key(key))

    async def delete_pattern(self, pattern: str) -> int:
        """Delete all keys matching pattern."""
        redis = await get_redis()
        keys = await redis.keys(f"{self.prefix}:{pattern}")
        if keys:
            return await redis.delete(*keys)
        return 0
```

## Cache-aside pattern

```python
# app/features/users/repository.py
from app.core.cache import Cache

users_cache = Cache(prefix="users", default_ttl=600)

class UserRepository:
    async def get_by_id(self, user_id: str) -> User | None:
        # Check cache first
        cached = await users_cache.get(f"id:{user_id}")
        if cached:
            return User(**cached)

        # Cache miss — query database
        result = await self._session.execute(select(User).where(User.id == user_id))
        user = result.scalar_one_or_none()

        if user:
            await users_cache.set(f"id:{user_id}", user.to_dict())
        return user

    async def update(self, user: User) -> User:
        # Update database
        # ... update logic ...

        # Invalidate cache
        await users_cache.delete(f"id:{user.id}")
        await users_cache.delete_pattern(f"email:{user.email}")

        return user
```

## Per-user caching

```python
# app/features/billing/repository.py
from app.core.cache import Cache

class BillingCache:
    def __init__(self, user_id: str):
        self.cache = Cache(prefix=f"billing:{user_id}", default_ttl=300)

    async def get_subscription(self) -> dict | None:
        return await self.cache.get("subscription")

    async def set_subscription(self, data: dict) -> None:
        await self.cache.set("subscription", data)

    async def invalidate(self) -> None:
        await self.cache.delete_pattern("*")

# Usage
billing = BillingCache(user_id="user-123")
sub = await billing.get_subscription()
```

## Cache warming

```python
# app/workers/jobs/warm_cache.py
from app.core.cache import Cache
from app.features.users.repository import UserRepository
from app.workers.logging import get_worker_logger

logger = get_worker_logger(__name__)

async def warm_user_cache(ctx: dict) -> dict:
    """Pre-populate cache for active users."""
    cache = Cache(prefix="users", default_ttl=600)
    repo = UserRepository(ctx["session"])

    users = await repo.get_active_users(limit=1000)
    for user in users:
        await cache.set(f"id:{user.id}", user.to_dict())

    logger.info("Cache warmed", extra={"user_count": len(users)})
    return {"warmed": len(users)}
```

## Rate limiting

```python
# app/core/rate_limit.py
from app.core.redis import get_redis
from app.core.logging import get_logger

logger = get_logger(__name__)

class RateLimiter:
    def __init__(self, key: str, limit: int, window: int):
        self.key = key
        self.limit = limit
        self.window = window

    async def is_allowed(self) -> bool:
        redis = await get_redis()
        current = await redis.incr(f"ratelimit:{self.key}")

        if current == 1:
            await redis.expire(f"ratelimit:{self.key}", self.window)

        if current > self.limit:
            logger.warning("Rate limit exceeded", extra={"key": self.key, "count": current})
            return False
        return True

# Usage
limiter = RateLimiter(key="api:global", limit=100, window=60)
if not await limiter.is_allowed():
    raise HTTPException(status_code=429, detail="Too many requests")
```

## Cache invalidation strategies

```python
# 1. Time-based (TTL) — simplest, eventual consistency
await cache.set("key", value, ttl=300)  # Expires after 5 minutes

# 2. Event-based — invalidate on write
async def update_product(product_id: str, data: dict):
    product = await repo.update(product_id, data)
    await cache.delete(f"product:{product_id}")
    await cache.delete_pattern("products:list:*")  # Invalidate all list caches

# 3. Version-based — cache bust with version
async def get_products(version: str = "v1"):
    cache_key = f"products:{version}"
    cached = await cache.get(cache_key)
    if not cached:
        cached = await fetch_products()
        await cache.set(cache_key, cached, ttl=3600)
    return cached
```

## DO NOT

- **Never** cache without TTL — stale data forever is worse than no cache.
- **Never** cache sensitive data (passwords, tokens, PII) — use short TTLs or skip caching.
- **Never** forget to invalidate cache on writes — stale reads cause bugs.
- **Never** use cache as primary data store — it can be lost on restart.
- **Never** cache database queries that change frequently — use shorter TTLs.
- **Never** skip cache key namespacing — collisions cause wrong data.
- **Never** use `del` for pattern deletes — use `keys()` + `delete()` or Lua script.
- **Never** cache without logging hit/miss ratios — you need to know if caching helps.
