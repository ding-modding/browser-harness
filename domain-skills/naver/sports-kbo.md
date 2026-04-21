# Naver Sports — KBO Rankings & Stats

`https://m.sports.naver.com/kbaseball/record/kbo` — Naver's live KBO (Korean Baseball) rankings and per-player stats. **Browser required.** The page is an SPA — `http_get` returns a ~3KB shell with no data. Once the browser renders, `document.body.innerText` gives you a predictable whitespace-separated feed that parses cleanly.

## URL shape

```
https://m.sports.naver.com/kbaseball/record/kbo?seasonCode={YEAR}&tab={TAB}
```

- `seasonCode` — 4-digit year, e.g. `2026`. The page also supports `정규시즌` vs `시범경기` via an in-page selector (no URL param).
- `tab` values (confirmed 2026-04-21):
  - `teamRank` — team standings (W/L/T, win%, streak, avg, ERA, last 5 games)
  - `teamRecord` — aggregate team stats
  - `hitter` — batter stats
  - `pitcher` — pitcher stats

Desktop URL `sports.news.naver.com/kbaseball/...` redirects to the same SPA.

---

## Extract team standings

```python
import json
from helpers import new_tab, wait_for_load, wait, js

new_tab("https://m.sports.naver.com/kbaseball/record/kbo?seasonCode=2026&tab=teamRank")
wait_for_load(15); wait(3)

# The team table renders as alternating cells; pull rows by the team-row class.
raw = js("""
  const rows = document.querySelectorAll('[class*="TeamRankTable"] tr, table.TeamRank_table__5X5Tc tr');
  // fallback: innerText-based parse is simpler and stable across redesigns
  const txt = document.body.innerText;
  JSON.stringify({text: txt});
""")
text = json.loads(raw)['text']

# Team rows appear after '팀 순위' header, one team per 11-line block:
# rank, team, win%, GB, W, D, L, games, streak, AVG, ERA
# Confirmed output (2026-04-21, 2026 season):
#   1 삼성  .706  0.0  12 1 5  18 1패  .272 4.17
#   2 KT    .684  0.0  13 0 6  19 1패  .285 3.95
#   3 LG    .667  0.5  12 0 6  18 1승  .262 3.60
```

For robust parsing, slice `text` between `팀 순위` and the next section header (`공지사항` or `더보기`), then split into 11-field chunks.

---

## Extract batter rankings

```python
import json, re
from helpers import new_tab, wait_for_load, wait, js

new_tab("https://m.sports.naver.com/kbaseball/record/kbo?seasonCode=2026&tab=hitter")
wait_for_load(15); wait(3)

raw = js("""
  const txt = document.body.innerText;
  const start = txt.indexOf('기록별 순위');
  JSON.stringify({txt: txt.substring(start, start + 6000)});
""")
text = json.loads(raw)['txt']

# Two sections in order:
#   1) 기록별 순위 — top-4 for each stat (타율, 홈런, 타점, 도루, OPS, WAR), separated by '더보기'
#   2) 타자 기록 — full table, top 19+ rows with every column per player

def parse_category_top4(text):
    """Returns {'타율': [(rank, player, team, value), ...], '홈런': [...], ...}"""
    out = {}
    # Categories come in fixed order. Split by '더보기'.
    blocks = text.split('더보기')
    for block in blocks[:6]:  # 6 categories for hitters
        lines = [l.strip() for l in block.split('\n') if l.strip()]
        # first meaningful token is the category name
        if not lines or lines[0] == '기록별 순위':
            lines = lines[1:]
        if not lines: continue
        category = lines[0]
        entries = []
        # each entry: rank, '위', player, team, value
        i = 1
        while i + 4 < len(lines):
            try:
                rank = int(lines[i])
                if lines[i+1] != '위': break
                entries.append((rank, lines[i+2], lines[i+3], lines[i+4]))
                i += 5
            except ValueError:
                break
        out[category] = entries
    return out

tops = parse_category_top4(text)
# Confirmed output (2026-04-21):
#   tops['타율'] = [(1, '박성한', 'SSG', '0.470'), (2, '류지혁', '삼성', '0.415'),
#                    (3, '천성호', 'LG', '0.391'), (4, '문현빈', '한화', '0.382')]
#   tops['홈런'] = [(1, '김도영', 'KIA', '6개'), (1, '장성우', 'KT', '6개'), ...]
#   categories: 타율, 홈런, 타점, 도루, OPS, WAR
```

### Full per-player table

The same page scrolls further into a full stats table (columns: 타율, 경기, 타수, 안타, 홈런, 2루타, 3루타, 타점, 득점, 도루, 볼넷, 사구, 삼진, 출루율, 장타율, OPS, IsoP, BABIP, wOBA, wRC+, WPA, WAR). Each player row is one `[rank] 위 [player] [team]` then `[col_name] [value]` pairs. Parse by finding `\d+\n위\n` anchors and reading 22 label/value pairs after each.

---

## Extract pitcher rankings

```python
new_tab("https://m.sports.naver.com/kbaseball/record/kbo?seasonCode=2026&tab=pitcher")
wait_for_load(15); wait(3)
# Same shape as hitter: '기록별 순위' top-4 per category, then full table.
# Pitcher categories (6): 승, 평균자책, 탈삼진, 세이브, WHIP, WAR
# Full table columns: 평균자책, 경기, 승, 패, 세이브, 홀드, 이닝, 탈삼진, 피안타,
#                     피홈런, 실점, 자책점, 볼넷, 사구, 승률, QS, WHIP, K/9, BB/9,
#                     K/BB, K%, BB%, WPA, WAR
```

Confirmed outputs (2026-04-21):
- 다승 1위: 보쉴리 (KT) 4승
- 평균자책 1위: 보쉴리 (KT) 0.78
- 탈삼진 1위: 알칸타라 (키움) 29개
- WHIP 1위: 류현진 (한화) 0.72

---

## Gotchas

- **SPA — `http_get` is useless here.** The URL returns a ~3KB shell; all data is hydrated by JS after load. Always use `new_tab` + `wait_for_load` + `wait(2–3)` before extracting.

- **`tab=batterRank` / `tab=pitcherRank` do NOT work.** The correct values are `tab=hitter` and `tab=pitcher`. Wrong tab values silently render an empty page (no 404).

- **`seasonCode` changes the URL after render.** Navigating to `/record/index` redirects to `/record/kbo?seasonCode={current}&tab=teamRank` — always pin `seasonCode` explicitly if you want historical data.

- **Class names are utility-hashed (`TeamRankTable__...`, `RecordRank_item__...`).** They change between redesigns. Don't regex HTML for them — use `document.body.innerText` and parse by the stable Korean labels (`기록별 순위`, `타자 기록`, `위`, `더보기`).

- **Early-season caveat.** In April the sample is ~18 games and rate stats (타율, ERA, OPS) swing wildly. If you're using the data for analysis, surface `경기` / `이닝` alongside the headline number so users see the sample size.

- **Ties share a rank.** `기록별 순위` can return `[(1, ...), (1, ...), (3, ...), (3, ...)]` (two players tied for 1st, two tied for 3rd). Don't assume ranks are unique.

- **Value strings include units.** `6개`, `22점`, `0.470`, `23 1/3` (innings with fractional). Strip units before numeric conversion: `int(v.rstrip('개점'))`, and for innings convert `"23 1/3"` → `23 + 1/3`.

- **Regular season vs spring training.** The dropdown labeled `정규시즌 ▼` also exposes `시범경기` — if you see stats starting mid-Feb with tiny game counts, you may be on spring training data. There is no URL param for this; the default is regular season.

- **`더보기` links navigate to a full-list page, not an in-place expansion.** The top-4 preview on `기록별 순위` is all you get from a single fetch of the main tab; clicking `더보기` takes you to `?tab=hitter&category={stat}` (or similar). Only use if you need deeper than top-4 for a specific stat.
