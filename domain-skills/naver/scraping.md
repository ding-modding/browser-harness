# Naver — Scraping & Data Extraction

`https://www.naver.com` — Korea's dominant portal. News, blogs, 지식iN (Q&A), shopping, maps, dictionary, cafe. **Most article/post pages are SSR with clean `og:*` meta tags** and work great with `http_get`. **Search result listings are JS-infinite-scroll** and only expose ~5 items per SSR page — use the browser for bulk enumeration.

## Do this first: pick your access path

| Goal | Best approach | Auth | Latency |
|------|--------------|------|---------|
| Autocomplete / query suggestions | `ac.search.naver.com/nx/ac` JSON endpoint | none | ~150ms |
| Fetch a news article by URL | `http_get` + regex on `og:*` and `dic_area` | none | ~300ms |
| Fetch a blog post by URL | `http_get` on **`PostView.naver`** (not `blog.naver.com/...`) | none | ~300ms |
| Fetch a 지식iN Q&A by URL | `http_get` + regex on `og:*` | none | ~400ms |
| Enumerate a user's blog posts (up to 100s) | RSS: `rss.blog.naver.com/{blogId}.xml` | none | ~200ms |
| Enumerate >5 search results | **browser** with infinite-scroll | none | ~1–3s/page |
| Shopping price / product details | browser or official OpenAPI (needs key) | — | — |

**Key gotcha up front:** `https://blog.naver.com/{user}/{postId}` returns a **2.9KB frameset wrapper**, not the post. Fetch `https://blog.naver.com/PostView.naver?blogId={user}&logNo={postId}` instead — that's the real content (~200KB with body).

---

## Path 1: Autocomplete (public JSON, no key)

```python
import json, urllib.parse
from helpers import http_get

q = urllib.parse.quote('파이썬')
raw = http_get(
    f"https://ac.search.naver.com/nx/ac?q={q}"
    "&con=1&frm=nv&ans=2&r_format=json&r_enc=UTF-8"
    "&r_unicode=0&t_koreng=1&run=2&rev=4&q_enc=UTF-8&st=100"
)
data = json.loads(raw)
suggestions = [item[0] for item in data['items'][0]]
print(suggestions)
# Confirmed output (2026-04-21):
# ['파이썬', '파이썬 뜻', '파이썬 독학', '파이썬 다운로드', '파이썬 기초',
#  '파이썬 자격증', '파이썬 코딩', '파이썬 책', '파이썬 코드', '점프투파이썬']
```

Useful for keyword expansion, trending-topic analysis, and SEO research. The endpoint is used by naver.com's live search box — no bot challenge, no rate limit observed in light use.

---

## Path 2: News articles (naver-hosted)

Every naver-hosted news article lives under `https://n.news.naver.com/mnews/article/{PRESS_ID}/{ARTICLE_ID}?sid={SECTION_ID}`. The page is SSR and has clean structured fields.

```python
import re
from helpers import http_get

def parse_naver_news(url):
    page = http_get(url)
    m = lambda pat, flags=0: (re.search(pat, page, flags) or type('X', (), {'group': lambda _s, _i=1: None})())

    # press name is the alt= on the header logo img
    press_logo = m(r'<a[^>]*class="media_end_head_top_logo"[^>]*>.*?<img[^>]*alt="([^"]+)"', re.DOTALL)

    # body: everything inside <article id="dic_area">
    body_raw = m(r'<article[^>]*id="dic_area"[^>]*>(.*?)</article>', re.DOTALL).group(1) or ''
    body_text = re.sub(r'\s+', ' ', re.sub(r'<[^>]+>', ' ', body_raw)).strip()

    return {
        'url':       url,
        'title':     m(r'<meta property="og:title" content="([^"]+)"').group(1),
        'summary':   m(r'<meta property="og:description" content="([^"]+)"').group(1),
        'published': m(r'data-date-time="([^"]+)"').group(1),       # 'YYYY-MM-DD HH:MM:SS'
        'press':     press_logo.group(1),                           # e.g. '한겨레'
        'body':      body_text,
    }

art = parse_naver_news("https://n.news.naver.com/mnews/article/028/0002799752?sid=105")
print(art['title'])      # '‘아첨꾼’ 제미나이에게 보컬 레슨을 받아보았다 [두런두런 AI ④]'
print(art['press'])      # '한겨레'
print(art['published'])  # '2026-04-08 15:22:16'
print(len(art['body']))  # ~6268 chars of clean text
```

The URL `sid` values: `100` 정치, `101` 경제, `102` 사회, `103` 생활/문화, `104` 세계, `105` IT/과학, `106` 연예, `107` 스포츠.

**Not all press outlets host on naver.** Search results mix naver-hosted (`n.news.naver.com`) and external (`hani.co.kr`, `chosun.com`, ...). For external URLs, scrape directly or use the press site's own scraper.

---

## Path 3: Blog posts (use PostView.naver)

**Do not fetch `blog.naver.com/{user}/{postId}` directly** — it returns a 2.9KB frameset wrapper that loads the real content in an iframe. Hit the iframe URL directly:

```python
import re
from helpers import http_get

def parse_naver_blog(blog_id, log_no):
    url = f"https://blog.naver.com/PostView.naver?blogId={blog_id}&logNo={log_no}"
    page = http_get(url)
    m = lambda pat, flags=0: (re.search(pat, page, flags) or type('X', (), {'group': lambda _s, _i=1: None})())

    # SE3 editor body — everything inside div.se-main-container
    body_raw = m(r'<div[^>]*class="se-main-container"[^>]*>(.*?)(?=</div>\s*(?:<div[^>]*class="(?:post_footer|blog2_container|_post_comment)))', re.DOTALL).group(1) or ''
    body_text = re.sub(r'\s+', ' ', re.sub(r'<[^>]+>', ' ', body_raw)).strip()

    return {
        'blog_id':   blog_id,
        'log_no':    log_no,
        'url':       url,
        'title':     m(r'<meta property="og:title" content="([^"]+)"').group(1),
        'summary':   m(r'<meta property="og:description" content="([^"]+)"').group(1),
        'author':    m(r'<meta property="og:article:author" content="([^"]+)"').group(1),
        'published': m(r'class="se_publishDate[^"]*"[^>]*>([^<]+)<').group(1),  # 'YYYY. M. D. HH:MM'
        'body':      body_text,
    }

post = parse_naver_blog('urmyver', '224155686212')
print(post['title'])     # '파이썬 프로그래밍 가장 좋은 입문 방법은?'
print(post['author'])    # '네이버 블로그 | 주들의 그리 대단하진 않지만 굉장한 이야기'
print(post['published']) # '2026. 1. 22. 12:19'
```

### URL shapes you may encounter

- `https://blog.naver.com/{user}/{postId}` — wrapper, useless for scraping
- `https://blog.naver.com/PostView.naver?blogId={user}&logNo={postId}` — the real content
- `https://m.blog.naver.com/{user}/{postId}` — mobile version, also SSR, has the post body directly (no frameset). Good fallback if `PostView.naver` ever breaks.

---

## Path 4: Bulk-enumerate a user's blog via RSS

Every public naver blog has an RSS feed. No auth, no pagination headaches, ~latest 20 posts per call.

```python
import re
from helpers import http_get

rss = http_get("https://rss.blog.naver.com/urmyver.xml")
items = re.findall(
    r'<item>.*?<title><!\[CDATA\[([^\]]+)\]\]></title>'
    r'.*?<link>([^<]+)</link>'
    r'.*?<pubDate>([^<]+)</pubDate>',
    rss, re.DOTALL
)
for title, link, pub in items[:5]:
    print(pub[:16], title[:60], '->', link)
# Pair with Path 3 to fetch each post's full body.
```

`link` values come back as `https://blog.naver.com/{user}/{postId}` — convert to `PostView.naver?blogId=...&logNo=...` for Path 3.

---

## Path 5: 지식iN (Q&A) articles

```python
import re
from helpers import http_get

url = "https://kin.naver.com/qna/detail.naver?d1id=1&dirId=10501&docId=..."  # from search
page = http_get(url)
title = re.search(r'<meta property="og:title" content="([^"]+)"', page).group(1)
body  = re.search(r'<meta property="og:description" content="([^"]+)"', page).group(1)
# og:description holds the question text; answer bodies are in div._endContents
answers = re.findall(
    r'<div[^>]*class="_endContents[^"]*"[^>]*>(.*?)</div>\s*<div[^>]*class="answerDetail',
    page, re.DOTALL
)
# Strip tags on each answer for clean text
```

Search URL: `https://search.naver.com/search.naver?where=kin&query={q}` — link pattern `kin.naver.com/qna/detail.naver?...` (~32 links per SSR page, more than other verticals).

---

## Path 6: Search result enumeration (browser required for >5 items)

The unified search and every vertical (`where=blog`, `where=news`, `where=view`, `where=kin`) use infinite-scroll lazy loading. **`http_get` gives you ~5 results; `&start=11/21/31` returns the same first 5.** The actual list is under a `<ul class="list_news _infinite_list">` container that lazy-loads as you scroll.

For any workflow needing more than ~5 search results, use the browser:

```python
from helpers import new_tab, wait_for_load, scroll, js, wait
import urllib.parse

q = urllib.parse.quote('파이썬 입문')
new_tab(f"https://search.naver.com/search.naver?where=blog&query={q}")
wait_for_load()

# Trigger infinite scroll
for _ in range(6):
    scroll(640, 400, dy=2000)
    wait(0.6)

# Extract blog URLs from the now-hydrated DOM
urls = js("""
  const out = new Set();
  for (const a of document.querySelectorAll('a[href*="blog.naver.com/"]')) {
    const m = a.href.match(/blog\\.naver\\.com\\/[^/?#]+\\/\\d+/);
    if (m) out.add('https://' + m[0]);
  }
  JSON.stringify([...out]);
""")
# hand the URLs off to parse_naver_blog from Path 3
```

Class names on the search page are obfuscated utility classes (`sds-comps-*`, `fender-ui_*`) that change across redesigns — **do not regex search HTML for titles**. Extract stable URLs (blog post IDs, naver-news article IDs) and hit each article page via Path 2/3 for clean metadata instead.

---

## URL shape cheatsheet

| Type | Stable URL shape | Notes |
|------|------------------|-------|
| Search, unified | `search.naver.com/search.naver?query={q}` | SSR, ~5 items |
| Search, vertical | `search.naver.com/search.naver?where={blog\|news\|view\|kin\|news\|shop}&query={q}` | same limit |
| News article | `n.news.naver.com/mnews/article/{PRESS}/{ID}?sid={SECTION}` | SSR, clean |
| Blog post | `blog.naver.com/PostView.naver?blogId={u}&logNo={id}` | SSR — **not** `/{u}/{id}` |
| Blog post (mobile) | `m.blog.naver.com/{u}/{id}` | SSR, alternative |
| Blog RSS | `rss.blog.naver.com/{blogId}.xml` | up to 20 posts |
| 지식iN | `kin.naver.com/qna/detail.naver?d1id=...&docId=...` | SSR |
| Autocomplete | `ac.search.naver.com/nx/ac?q={q}&r_format=json&...` | JSON |

---

## Gotchas

- **`blog.naver.com/{user}/{postId}` is a frameset, not the post.** It returns ~2.9KB of JS that loads the real content from `/PostView.naver`. Always call `PostView.naver?blogId=...&logNo=...` directly. `m.blog.naver.com/{u}/{id}` is an SSR alternative.

- **Search results are infinite-scroll; `start=11/21/31` don't paginate SSR.** Every page from `start=1` returns the same ~5 items. Use the browser + `scroll()` + `js()` to hydrate more, or use each vertical's RSS/API when available.

- **Search result HTML uses obfuscated utility classes (`sds-comps-*`, `fender-ui_{hash}`).** These change across redesigns. Never scrape titles from the search page HTML — extract stable article/post URLs (they follow fixed shapes listed above) and fetch each article page individually for metadata.

- **News `data-date-time` is KST, no timezone suffix.** Format `YYYY-MM-DD HH:MM:SS`; assume `+09:00` if you need a timezone-aware datetime.

- **`og:description` is truncated at ~160 chars** and usually ends mid-sentence with `...`. For full text, parse the body container (`article#dic_area` for news, `div.se-main-container` for blog).

- **Press-name-as-image.** The outlet that published a naver-hosted news article is only readable from the logo `<img alt="...">` on the header; there's no meta tag or JSON-LD for it. The regex `<a[^>]*class="media_end_head_top_logo"[^>]*>.*?<img[^>]*alt="([^"]+)"` with `re.DOTALL` works.

- **Blog `og:article:author` is `"네이버 블로그 | {blog title}"`, not the blog owner's user ID.** Use the URL's `blogId` for the stable user handle.

- **Not all naver news links in search results are naver-hosted.** `n.news.naver.com/...` is naver-hosted (scrapable via Path 2). Other press-domain URLs (`hani.co.kr`, `chosun.com`, ...) are direct-to-publisher — scrape those with whatever the publisher's page structure allows.

- **Autocomplete JSON has a nested array shape:** `data['items'][0]` is a list of `[suggestion, count_as_string]` pairs. Don't forget to index into `items[0]` first.

- **Mixed `&amp;` encoding in href.** Links in the search result HTML are often HTML-encoded (`?blogId=x&amp;logNo=y`). Decode with `html.unescape()` or `.replace('&amp;', '&')` before using the URL.

- **Naver robots policy is lenient for SSR pages, strict for APIs.** Light `http_get` polling (1 req/sec) works fine in practice. The autocomplete endpoint begins throttling if hammered — add `time.sleep(0.5–1.0)` if doing bulk expansion.

- **For the official Naver OpenAPI (`openapi.naver.com`)** you need a `X-Naver-Client-Id` + `X-Naver-Client-Secret` header. Request a free key at `developers.naver.com/apps`. It returns cleaner JSON for news/blog/kin search and lifts the 5-result SSR cap — preferred over scraping if the task is long-running.
