# Browse the Web

Let agents navigate websites, extract content, and interact with web pages. Use Playwright
for browser automation with proper sandboxing and rate limiting.

## Browser setup

```python
# app/features/agents/browse/browser.py
from playwright.async_api import async_playwright
from app.core.logging import get_logger

logger = get_logger(__name__)

class Browser:
    def __init__(self):
        self._playwright = None
        self._browser = None

    async def start(self):
        self._playwright = await async_playwright().start()
        self._browser = await self._playwright.chromium.launch(headless=True)
        logger.info("Browser started")

    async def stop(self):
        if self._browser:
            await self._browser.close()
        if self._playwright:
            await self._playwright.stop()
        logger.info("Browser stopped")

    async def navigate(self, url: str) -> dict:
        """Navigate to URL and extract content."""
        page = await self._browser.new_page()
        try:
            await page.goto(url, wait_until="domcontentloaded", timeout=30000)

            content = await page.content()
            title = await page.title()

            logger.info("Page loaded", extra={"url": url, "title": title})

            return {
                "title": title,
                "content": content[:5000],  # Limit content size
                "url": url,
            }
        finally:
            await page.close()
```

## Agent tool

```python
# app/features/agents/browse/tool.py
from app.features.agents/browse.browser import Browser
from app.core.logging import get_logger

logger = get_logger(__name__)

browser = Browser()

async def browse_web(ctx: dict, *, url: str) -> dict:
    """Navigate to a URL and return the page content."""
    result = await browser.navigate(url)
    logger.info("Web browse", extra={"url": url, "title": result["title"]})
    return result
```

## Content extraction

```python
# app/features/agents/browse/extract.py
from bs4 import BeautifulSoup
from app.core.logging import get_logger

logger = get_logger(__name__)

def extract_text(html: str) -> str:
    """Extract clean text from HTML."""
    soup = BeautifulSoup(html, "html.parser")

    # Remove script and style elements
    for element in soup(["script", "style", "nav", "footer"]):
        element.decompose()

    text = soup.get_text(separator="\n", strip=True)

    # Limit to reasonable size
    if len(text) > 10000:
        text = text[:10000] + "\n\n[Content truncated]"

    return text

def extract_links(html: str) -> list[dict]:
    """Extract all links from HTML."""
    soup = BeautifulSoup(html, "html.parser")
    links = []
    for link in soup.find_all("a", href=True):
        links.append({
            "text": link.get_text(strip=True),
            "href": link["href"],
        })
    return links[:50]  # Limit to 50 links
```

## Search integration

```python
# app/features/agents/browse/search.py
import httpx
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)

async def search_web(query: str, num_results: int = 5) -> list[dict]:
    """Search the web using a search API."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            "https://api.searchapi.com/v1/search",
            params={
                "q": query,
                "num": num_results,
                "api_key": settings.search_api_key,
            },
            timeout=30,
        )
        resp.raise_for_status()
        results = resp.json().get("organic_results", [])
        logger.info("Web search", extra={"query": query, "result_count": len(results)})
        return results
```

## DO NOT

- **Never** run browser in the main process — use a separate worker/container.
- **Never** skip rate limiting on web requests — you'll get blocked.
- **Never** store full page content in logs — they can be huge.
- **Never** skip timeout on navigation — pages can hang indefinitely.
- **Never** allow browsing internal/private IPs — that's SSRF.
- **Never** skip content size limits — huge pages will consume memory.
- **Never** execute JavaScript from untrusted pages — it can be malicious.
- **Never** forget to close browser pages — memory leaks.
- **Never** browse without user consent — respect robots.txt and ToS.
