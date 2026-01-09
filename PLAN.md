# Squarespace Blog XML Parser with AI Categorization

## Overview

Build a Python script that parses a Squarespace blog export (WordPress
XML format), uses Claude API to intelligently categorize posts, and
stores everything in a SQLite database.

## Key Findings

- **Squarespace export**: Provides WordPress-compatible XML export
  with all blog data
- **Export includes**: Posts, pages, comments, existing tags/categories,
  images, metadata
- **Solution**: Parse structured XML (much simpler than web scraping!)
- **Your workspace**: Clean slate - no existing code to integrate with

## Architecture

```text
XML Parser → AI Categorizer → SQLite Database
     ↓              ↓                ↓
ElementTree   Claude API      posts + categories
```

## Why This Approach is Better Than Web Scraping

✅ **Much simpler**: No HTTP requests, no HTML parsing, no rate limiting
to worry about

✅ **More reliable**: Structured XML data vs unpredictable HTML across
different templates

✅ **Faster**: Single file parse vs hundreds of web requests

✅ **Complete data**: Already includes categories, tags, dates, authors
in structured format

✅ **Fewer dependencies**: Only need `anthropic` SDK, everything else is
Python built-ins

✅ **No network issues**: Works offline once you have the XML file

## Typical User Workflow

1. **One-time setup**: Install Python dependencies, configure Claude API
   key
2. **Export from Squarespace**: Download XML file (takes 2 minutes)
3. **Initial import**: Run `python main.py --mode full --file export.xml`
   to import all posts and categorize
4. **Review results**: Check database to see categorized posts
5. **Periodic updates**: Export from Squarespace again when you have new
   posts, re-import

**For posts with existing categories**: The script will preserve them
and optionally add AI-suggested categories

**For uncategorized posts**: Claude will analyze content and suggest
2-4 relevant categories

## Implementation Steps

### 1. Project Setup

- Create directory structure: `src/`, `data/`, `logs/`, `tests/`
- Install dependencies: `anthropic`, `python-dotenv`, `python-dateutil`
  (built-in `xml.etree.ElementTree` for parsing)
- Create `.env` file for Claude API key
- Set up logging configuration
- **User action**: Export blog from Squarespace (Settings → Advanced →
  Import & Export → Export → WordPress)

### 2. Database Layer (`src/database.py`)

**Schema**:

- `posts` table: url, title, content, publish_date, categorization_status
- `categories` table: name, source (original vs ai-generated)
- `post_categories` junction table: many-to-many with confidence scores
- `api_usage` table: track Claude API token usage for cost management
- `scrape_runs` table: audit trail of scraping operations

**Key features**:

- Use `post_url` as unique constraint for idempotency
- Context manager for safe connection handling
- Methods: insert_post, get_uncategorized_posts, assign_category, log_api_usage

### 3. XML Parser (`src/parser.py`)

**Functionality**:

- Parse WordPress XML export file (WXR format)
- Extract all blog posts with metadata
- Handle: title, content, publish date, author, existing categories/tags
- Convert HTML content to plain text for AI processing
- Handle both published and draft posts

**WordPress XML structure**:

- Uses `<item>` elements for each post
- Post type: `<wp:post_type>post</wp:post_type>`
- Content in CDATA: `<content:encoded>`
- Categories/tags: `<category>` elements with domain attribute
- Publish date: `<wp:post_date>` or `<pubDate>`
- Author: `<dc:creator>`

### 4. AI Categorizer (`src/categorizer.py`)

**Functionality**:

- Use Claude API (claude-3-5-sonnet-20241022) to analyze post content
- Generate 2-4 relevant categories per post
- Return structured JSON: categories, confidence score, reasoning
- Track token usage for cost management
- Batch processing with configurable limits

**Prompt strategy**:

- System prompt: instructions for consistent categorization
- User prompt: post title + excerpt + content (truncated to ~3000 chars)
- Request JSON output for reliable parsing
- Include existing categories for context

### 5. Main Application (`main.py`)

**CLI modes**:

- `--mode import`: Parse XML and import to database
- `--mode categorize`: Only categorize existing posts
- `--mode full`: Complete pipeline (import → categorize)
- `--file PATH`: Path to XML export file (required)
- `--limit N`: Limit categorization to N posts
- `--force-reimport`: Re-import even if posts exist

**Workflow**:

1. Parse WordPress XML export file
2. Extract all blog posts with existing metadata
3. Store in database (skip duplicates unless --force-reimport)
4. Store existing categories/tags from XML
5. Get posts needing AI categorization (no categories OR user wants enhancement)
6. Send to Claude for categorization
7. Store AI-generated categories with confidence scores
8. Log all API usage and errors

### 6. Configuration (`src/config.py` + `.env`)

**Settings**:

- Claude API key
- Database path
- AI model and parameters (temperature, max_tokens)
- Batch size for categorization
- Default XML file path (optional)

## WordPress XML Format (WXR) Details

The Squarespace export uses WordPress eXtended RSS format with these
key elements:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0"
    xmlns:excerpt="http://wordpress.org/export/1.2/excerpt/"
    xmlns:content="http://purl.org/rss/1.0/modules/content/"
    xmlns:wfw="http://wellformedweb.org/CommentAPI/"
    xmlns:dc="http://purl.org/dc/elements/1.1/"
    xmlns:wp="http://wordpress.org/export/1.2/">
  <channel>
    <item>
      <title>Post Title</title>
      <link>https://blog.com/post-slug</link>
      <pubDate>Mon, 01 Jan 2024 12:00:00 +0000</pubDate>
      <dc:creator><![CDATA[Author Name]]></dc:creator>
      <guid>https://blog.com/?p=123</guid>
      <description></description>
      <content:encoded>
        <![CDATA[<p>Full HTML content here...</p>]]>
      </content:encoded>
      <excerpt:encoded><![CDATA[Excerpt text]]></excerpt:encoded>
      <wp:post_id>123</wp:post_id>
      <wp:post_date>2024-01-01 12:00:00</wp:post_date>
      <wp:post_type>post</wp:post_type>
      <wp:status>publish</wp:status>
      <category domain="category" nicename="tech">
        <![CDATA[Technology]]>
      </category>
      <category domain="post_tag" nicename="python">
        <![CDATA[Python]]>
      </category>
    </item>
  </channel>
</rss>
```

**Key namespaces to handle**:

- `wp:` - WordPress-specific fields (post_id, post_date, post_type)
- `content:` - Full content with HTML
- `dc:` - Dublin Core (creator/author)
- `category` elements can be both categories and tags (check `domain` attribute)

## Critical Files to Create

1. **src/database.py** - Database schema and operations (foundation)
2. **src/parser.py** - WordPress XML parsing and data extraction (data source)
3. **src/categorizer.py** - Claude API integration (AI logic)
4. **src/config.py** - Configuration management (settings)
5. **src/utils.py** - Logging setup and HTML-to-text helpers
6. **main.py** - CLI entry point (orchestration)
7. **requirements.txt** - Python dependencies
8. **.env.example** - Template for environment variables

## Key Design Decisions

### Why XML Export over Web Scraping?

- **More reliable**: Structured data vs unpredictable HTML
- **Faster**: Single file parse vs multiple HTTP requests
- **Complete**: Includes all metadata, categories, dates already structured
- **No rate limits**: No network calls during import
- **One-time action**: User exports once, can re-process locally anytime

### Why xml.etree.ElementTree?

- Built into Python (no extra dependencies)
- Perfect for WordPress XML format (WXR)
- Lightweight and fast
- Standard library = well-tested and documented

### Why SQLite?

- Simple setup (no server required)
- Sufficient for personal blog scale
- Easy to back up (single file)
- Can migrate to PostgreSQL later if needed

### Why Claude 3.5 Sonnet?

- Best balance of quality, speed, and cost
- Strong at content analysis and categorization
- Structured output (JSON) support
- More cost-effective than Opus for this task

### Idempotency Strategy

- Use `post_url` or `post_id` from XML as unique identifier
- Check database before importing/categorizing
- Allows safe re-runs without duplicates
- Support force re-import with `--force-reimport` flag

## Error Handling

- **XML parsing errors**: Validate structure, log malformed items,
  continue with valid posts
- **Missing required fields**: Use defaults or skip post with warning
- **Categorization failures**: Mark post as 'failed', store error
  message, continue
- **Claude API rate limits**: Exponential backoff with tenacity library
- **Invalid AI responses**: Log and mark as failed, allow manual retry later

## Cost Management

- Track Claude API token usage in database
- Process in configurable batches (default: 10 posts)
- Use `--limit` flag to control spending
- Truncate long posts to ~3000 chars (balance quality vs cost)
- Estimated cost: ~$0.01-0.03 per post with Sonnet

## Verification Plan

After implementation, test the complete workflow:

1. **Export from Squarespace**:
   - Go to Settings → Advanced → Import & Export
   - Click Export → WordPress
   - Download the XML file (e.g., `squarespace-export.xml`)
   - Place it in the project directory

2. **Setup verification**:

   ```bash
   # Install and configure
   python -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   cp .env.example .env
   # Edit .env with ANTHROPIC_API_KEY
   ```

3. **Test XML import only**:

   ```bash
   python main.py --mode import --file squarespace-export.xml
   # Check logs/parser.log for success
   # Verify posts in data/blog_posts.db
   ```

4. **Test categorization** (limit to 2 posts for cost control):

   ```bash
   python main.py --mode categorize --limit 2
   # Check database for categories assigned
   # Review AI confidence scores
   ```

5. **Test full pipeline**:

   ```bash
   python main.py --mode full --file squarespace-export.xml --limit 5
   # Verify end-to-end: import → categorize → store
   ```

6. **Database inspection**:

   ```bash
   sqlite3 data/blog_posts.db
   SELECT COUNT(*) FROM posts;
   SELECT COUNT(*) FROM categories;
   SELECT p.title, c.name, pc.confidence
   FROM posts p
   JOIN post_categories pc ON p.id = pc.post_id
   JOIN categories c ON pc.category_id = c.id
   LIMIT 10;
   ```

7. **Cost tracking**:

   ```sql
   SELECT
     COUNT(*) as categorized_posts,
     SUM(prompt_tokens + completion_tokens) as total_tokens,
     SUM(cost_estimate) as estimated_cost
   FROM api_usage;
   ```

## Future Enhancements

- Web dashboard to browse categorized posts
- Export to CSV/JSON for analysis
- Category management (merge, rename, delete)
- Support for multiple blog XML imports
- Incremental updates (export only new posts from Squarespace)
- Category suggestions based on existing posts
- Bulk re-categorization with improved prompts

## References

- [Exporting your site – Squarespace Help Center][ss-export]
- [WordPress eXtended RSS (WXR) format documentation][wxr-docs]
- [Anthropic Claude API documentation][claude-docs]
- SQLite best practices for schema design
- Python xml.etree.ElementTree documentation

[ss-export]: https://support.squarespace.com/hc/en-us/articles/206566687-Exporting-your-site
[wxr-docs]: https://wordpress.org/support/article/tools-export-screen/
[claude-docs]: https://docs.anthropic.com/claude/reference/getting-started-with-the-api
