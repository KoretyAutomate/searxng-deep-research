# SearXNG Deep Research Tool

A self-hosted deep research module using SearXNG for web search with async content scraping. Perfect for AI agents, research automation, and data gathering without relying on paid search APIs.

## 🌟 Features

- 🔍 **Self-Hosted Search** - Uses local SearXNG instance (no API keys, no rate limits)
- 🚀 **Async Operations** - Parallel web scraping for maximum performance
- 🧹 **Smart Content Extraction** - BeautifulSoup-based cleaning removes ads, navigation, scripts
- 🎯 **Priority-Based Extraction** - main → article → content divs → body
- 📊 **Token Management** - Automatic limiting to ~2000 tokens per page
- 🛡️ **Polite Scraping** - Real browser User-Agent headers
- ⚡ **Robust Error Handling** - Graceful handling of timeouts, connection errors, malformed HTML
- 🔧 **Production Ready** - Full type hints, docstrings, logging

## 🚀 Quick Start

### 1. Start SearXNG Docker Container

```bash
docker run -d \
  --name searxng \
  -p 8080:8080 \
  -e SEARXNG_BASE_URL=http://localhost:8080 \
  searxng/searxng:latest
```

Verify it's running:
```bash
docker ps | grep searxng
curl http://localhost:8080
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Run Example

```bash
python search_agent.py
```

## 📖 Usage

### Basic Search

```python
import asyncio
from search_agent import SearxngClient

async def main():
    async with SearxngClient() as client:
        results = await client.search("Python asyncio tutorial", num_results=5)

        for result in results:
            print(f"{result.title}: {result.url}")
            print(f"Snippet: {result.snippet}\n")

asyncio.run(main())
```

### Deep Research with Scraping

```python
import asyncio
from search_agent import SearxngClient, DeepResearch

async def main():
    async with SearxngClient() as client:
        async with DeepResearch(client) as research:
            results = await research.deep_dive(
                query="machine learning transformers",
                top_n=5,
                engines=['google', 'bing', 'brave']
            )

            print(f"Query: {results.query}")
            print(f"Total results: {results.total_results}")
            print(f"Successfully scraped: {results.total_scraped}")

            for content in results.scraped_pages:
                if not content.error:
                    print(f"\n{content.title} ({content.word_count} words)")
                    print(f"URL: {content.url}")
                    print(f"Content preview: {content.content[:300]}...\n")

asyncio.run(main())
```

## 📚 API Reference

### SearxngClient

```python
class SearxngClient:
    def __init__(
        self,
        base_url: str = "http://localhost:8080",
        timeout: float = 10.0
    )

    async def validate_connection(self) -> bool:
        """Check if SearXNG instance is accessible."""

    async def search(
        self,
        query: str,
        engines: List[str] = ['google', 'bing', 'brave'],
        num_results: int = 5,
        language: str = "en"
    ) -> List[SearchResult]:
        """Perform search and return results."""
```

### DeepResearch

```python
class DeepResearch:
    def __init__(self, searxng_client: SearxngClient)

    async def fetch_page_content(self, url: str) -> ScrapedContent:
        """Fetch and clean content from a single webpage."""

    async def deep_dive(
        self,
        query: str,
        top_n: int = 5,
        engines: List[str] = None
    ) -> ResearchResult:
        """Perform deep research: search + scrape top N URLs in parallel."""
```

### Data Models

```python
@dataclass
class SearchResult:
    title: str
    url: str
    snippet: str
    engine: Optional[str] = None
    score: Optional[float] = None

@dataclass
class ScrapedContent:
    url: str
    title: str
    content: str
    word_count: int
    error: Optional[str] = None

@dataclass
class ResearchResult:
    query: str
    search_results: List[SearchResult]
    scraped_pages: List[ScrapedContent]
    total_results: int
    total_scraped: int
    errors: List[str]
```

## 🔧 Configuration

### Custom SearXNG URL

```python
async with SearxngClient(base_url="http://<your-server-ip>:8080") as client:
    results = await client.search("query")
```

Or set `SEARXNG_BASE_URL` in your `.env` file and `SearxngClient()` will pick it up automatically:

```ini
# .env
SEARXNG_BASE_URL=http://<your-server-ip>:8080
```

### Custom Search Engines

```python
results = await client.search(
    query="research topic",
    engines=['google', 'duckduckgo', 'wikipedia'],
    num_results=10
)
```

### Custom Timeouts

```python
# Search timeout
client = SearxngClient(timeout=15.0)

# Scraping timeout is set to 30s by default
# Modify SCRAPING_TIMEOUT constant in search_agent.py if needed
```

## 🎯 Use Cases

### AI Agent Integration

Perfect for CrewAI, LangChain, or custom AI agents:

```python
from crewai import Agent, Task, tool
from search_agent import SearxngClient, DeepResearch

@tool("DeepSearch")
def deep_search_tool(query: str) -> str:
    """Perform deep research on a topic."""
    async def search():
        async with SearxngClient() as client:
            async with DeepResearch(client) as research:
                return await research.deep_dive(query, top_n=5)

    import asyncio
    results = asyncio.run(search())

    # Format for agent consumption
    output = f"Research: {results.query}\n\n"
    for content in results.scraped_pages:
        if not content.error:
            output += f"## {content.title}\n{content.content}\n\n"
    return output

researcher = Agent(
    role="Research Specialist",
    tools=[deep_search_tool],
    goal="Gather comprehensive information"
)
```

### Batch Research

```python
async def batch_research(queries: List[str]):
    async with SearxngClient() as client:
        async with DeepResearch(client) as research:
            tasks = [research.deep_dive(q, top_n=3) for q in queries]
            results = await asyncio.gather(*tasks)
            return results

# Research multiple topics in parallel
queries = ["Python asyncio", "Rust async", "Go concurrency"]
results = asyncio.run(batch_research(queries))
```

## 🐛 Troubleshooting

### SearXNG Not Accessible

```bash
# Check container status
docker ps | grep searxng

# View logs
docker logs searxng

# Restart container
docker restart searxng

# Remove and recreate
docker stop searxng && docker rm searxng
docker run -d --name searxng -p 8080:8080 searxng/searxng:latest
```

### Scraping Failures

- **Timeout errors**: Increase `SCRAPING_TIMEOUT` in search_agent.py
- **403/429 errors**: Some sites block scrapers (even with User-Agent)
- **JavaScript-heavy sites**: This tool uses static HTML parsing, won't execute JS
- **Network issues**: Check firewall, proxy settings

### Memory Issues

If scraping many large pages:
- Reduce `top_n` parameter
- Decrease `MAX_CONTENT_LENGTH` in search_agent.py
- Process results in batches

## 🏗️ Architecture

```
┌─────────────────────┐
│  SearxngClient      │  → Search across multiple engines
└──────────┬──────────┘
           │
           ↓
┌─────────────────────┐
│  Search Results     │  → URLs, titles, snippets
└──────────┬──────────┘
           │
           ↓
┌─────────────────────┐
│  DeepResearch       │  → Async parallel scraping
└──────────┬──────────┘
           │
           ↓
┌─────────────────────┐
│  BeautifulSoup      │  → Clean HTML, extract text
└──────────┬──────────┘
           │
           ↓
┌─────────────────────┐
│  Cleaned Content    │  → ~2000 tokens per page
└─────────────────────┘
```

## ⚡ Performance

- **Parallel Scraping**: Uses `asyncio.gather()` for concurrent requests
- **Typical Speed**: 5 URLs in 2-5 seconds (depends on network and target sites)
- **Memory**: ~10-50MB per scraped page (varies by content size)

## 🤝 Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Submit a pull request

## 📄 License

MIT License - see LICENSE file for details

## 🙏 Acknowledgments

- [SearXNG](https://github.com/searxng/searxng) - Privacy-respecting metasearch engine
- [httpx](https://www.python-httpx.org/) - Modern async HTTP client
- [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/) - HTML parsing library

## 🔗 Links

- [SearXNG Documentation](https://docs.searxng.org/)
- [SearXNG Docker Hub](https://hub.docker.com/r/searxng/searxng)
- [httpx Documentation](https://www.python-httpx.org/)

## 📞 Support

For issues and questions:
- Open an issue on GitHub
- Check existing issues for solutions
- Review SearXNG documentation for search engine configuration

---

Built with ❤️ for the self-hosted community
