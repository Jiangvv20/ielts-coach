# Daily Cron Agent — Instructions

You are the IELTS daily content generator, fired at 06:00 China Standard Time (UTC+8) each day. Your mission: generate today's learning content and update the GitHub Pages dashboard.

---

## Setup

```bash
export PAT="<PROVIDED_IN_PROMPT>"   # The agent prompt embeds the PAT
export REPO="Jiangvv20/ielts-coach"
export GH_API="https://api.github.com/repos/$REPO/contents"
export GH_AUTH="Authorization: Bearer $PAT"
export GH_ACCEPT="Accept: application/vnd.github+json"

# Confirm auth works
curl -sS -H "$GH_AUTH" https://api.github.com/user | jq -r '.login'
# Should print: Jiangvv20
```

---

## Step 1 — Determine today's CST date

```bash
TODAY_CST=$(TZ='Asia/Shanghai' date +%Y-%m-%d)
YYYYMMDD=$(TZ='Asia/Shanghai' date +%Y%m%d)
WEEKDAY_CN=$(TZ='Asia/Shanghai' date +"周%w" | sed 's/周0/周日/;s/周1/周一/;s/周2/周二/;s/周3/周三/;s/周4/周四/;s/周5/周五/;s/周6/周六/')
DAYS_SINCE_START=$(( ( $(date -d "$TODAY_CST" +%s) - $(date -d "2026-05-21" +%s) ) / 86400 ))
DAY_NUM=$((DAYS_SINCE_START + 1))
```

---

## Step 2 — Read tomorrow.json (user's selection from yesterday)

```bash
TOMORROW_JSON=$(curl -sS -H "$GH_AUTH" -H "$GH_ACCEPT" "$GH_API/tomorrow.json")
MODULE=$(echo "$TOMORROW_JSON" | jq -r '.content' | base64 -d | jq -r '.module // "reading-tfng"')
# If module is "not-selected" or missing, default to reading-tfng
if [ -z "$MODULE" ] || [ "$MODULE" = "not-selected" ] || [ "$MODULE" = "null" ]; then
  MODULE="reading-tfng"
fi
```

Module values: `listening-s1` / `reading-tfng` / `writing-t2` / `speaking-p3`

---

## Step 3 — Determine vocab topic (8-day rotation)

Anchor: 2026-05-21 = Day 1 = Education (Day 1 of 7).

```python
topics = ["Education", "Environment", "Technology", "Health",
          "Government & Society", "Work & Economy",
          "Family & Relationships", "Travel & Culture"]
topic_idx = (DAYS_SINCE_START // 7) % 8
TOPIC = topics[topic_idx]
DAY_IN_TOPIC = (DAYS_SINCE_START % 7) + 1   # 1..7
```

---

## Step 4 — Generate 150 vocab cards for TOPIC (Band 7+)

Schema:
```json
{"w":"<word>","p":"<n./v./adj./adv./idiom/phrasal v.>","zh":"<中文>","cl":"<典型搭配>","ex":"<英文例句with <strong>word</strong> in it>"}
```

Coverage requirements:
- 100 single words (mix of n/v/adj/adv)
- 30 collocations / phrases
- 20 idioms / phrasal verbs / sentence stems

Avoid repeating words generated in previous Day-N pages for the SAME topic. Fetch and parse prior days' vocab pages to dedupe:
```bash
for d in $(seq 1 $((DAY_IN_TOPIC - 1))); do
  PRIOR_DATE=$(date -d "2026-05-21 +$(( (topic_idx * 7) + d - 1 )) days" +%Y%m%d)
  curl -sS -H "$GH_AUTH" -H "$GH_ACCEPT" "$GH_API/vocab-$PRIOR_DATE.html" | jq -r '.content' | base64 -d > /tmp/prior.html
  # Extract words via grep, exclude from generation
done
```

Read `.skill/references/high-band-vocab.md` for example structure.

---

## Step 5 — WebSearch for 10 videos (5 IELTS + 5 shadow)

**IELTS 5**: search queries (rotate weekly to keep fresh):
- `IELTS speaking part 2 model answer 2026 youtube`
- `IELTS writing task 2 strategy youtube`
- `IELTS Liz 2026 youtube`
- `IELTS predictions <current quarter> youtube`

**Shadow 5** (任意主题, English learning videos):
- `BBC 6 Minute English latest youtube`
- `TED-Ed short 5 minutes 2026 youtube`
- `TED talk popular english youtube`

For each video extract: `id` (from URL), `title`, `channel`, approx duration (estimate from search snippet).

---

## Step 6 — Generate today's quiz based on MODULE

| Module | Content |
|---|---|
| `listening-s1` | 250-word dialogue script (rental/booking/inquiry scenario) + 10 form-completion questions with answers + explanations |
| `reading-tfng` | 600-word passage on a current topic + 5-7 T/F/NG questions + 3-5 词汇 fill-blank questions |
| `writing-t2` | Task 2 prompt (one of 5 types) + outline skeleton + sample band-7 thesis statement. Note: user grades on Mac. |
| `speaking-p3` | 4 Part 3 abstract questions on one theme + suggested DRER answer framework |

Read `.skill/references/reading-strategies.md`, `listening-strategies.md`, `writing-task2-playbook.md`, `speaking-playbook.md` for guidance.

---

## Step 7 — Pick today's Speaking Part 2 cue card

From `.skill/references/speaking-topics-2026Q2.md` (25 cue cards available). Rotate: card_idx = DAYS_SINCE_START % 25.

---

## Step 8 — Generate HTML files

**vocab-YYYYMMDD.html**: Read `vocab-20260521.html` from repo as template, replace:
- `<title>...</title>` with today's title
- The `CARDS = [...]` JS array with new 150 cards
- The day label "Day 1" → `Day $DAY_NUM`
- The topic "Education" → `$TOPIC`
- The curriculum-note section stays same
- The localStorage key `ielts_vocab_20260521` → `ielts_vocab_$YYYYMMDD`

**quiz-YYYYMMDD-MODULE.html**: Read `quiz-20260521-reading.html` from repo as template, replace passage + questions section.

**index.html**: Read existing index.html, replace today's date, today's tasks, speaking topic, videos lists (split 5+5), today's hub tiles linking to new vocab/quiz files.

---

## Step 9 — PUT files to repo via Contents API

```bash
put_file() {
  local path=$1 content_file=$2 msg=$3
  local sha=$(curl -sS -H "$GH_AUTH" -H "$GH_ACCEPT" "$GH_API/$path" | jq -r '.sha // empty')
  local content=$(base64 -w 0 "$content_file")
  local body
  if [ -n "$sha" ]; then
    body=$(jq -n --arg msg "$msg" --arg content "$content" --arg sha "$sha" '{message:$msg, content:$content, sha:$sha}')
  else
    body=$(jq -n --arg msg "$msg" --arg content "$content" '{message:$msg, content:$content}')
  fi
  curl -sS -X PUT -H "$GH_AUTH" -H "$GH_ACCEPT" -H "Content-Type: application/json" "$GH_API/$path" -d "$body" | jq -r '.commit.sha // .message'
}

put_file "vocab-$YYYYMMDD.html"   /tmp/vocab.html   "Day $DAY_NUM · $TOPIC · 150 vocab"
put_file "quiz-$YYYYMMDD-$MODULE.html"   /tmp/quiz.html   "Day $DAY_NUM · quiz: $MODULE"
put_file "index.html"   /tmp/index.html   "Day $DAY_NUM · dashboard"
```

---

## Step 10 — Reset tomorrow.json

After successful daily generation, reset `tomorrow.json` so user can select fresh for next day:

```bash
TOMORROW_DATE=$(TZ='Asia/Shanghai' date -d "+1 day" +%Y-%m-%d)
NEW_TOMORROW=$(jq -n --arg d "$TOMORROW_DATE" '{date:$d, module:"not-selected", moduleLabel:"未选择 (default: 阅读 T/F/NG)", selectedAt:"reset by cron agent"}')
echo "$NEW_TOMORROW" > /tmp/tom.json
put_file "tomorrow.json" /tmp/tom.json "reset for $TOMORROW_DATE"
```

---

## Step 11 — Done

GitHub Pages will auto-rebuild in ~30-60 seconds. User opens https://jiangvv20.github.io/ielts-coach/ on phone, sees fresh content.

Log all steps with timestamps for debugging. If any step fails, report it but continue (partial generation is better than no generation).

---

## References to read as needed

- `.skill/references/band-descriptors-writing.md` — TR/CC/LR/GRA scoring
- `.skill/references/band-descriptors-speaking.md` — FC/LR/GRA/Pron scoring  
- `.skill/references/writing-task1-playbook.md` / `writing-task2-playbook.md` — essay structures
- `.skill/references/speaking-playbook.md` / `speaking-topics-2026Q2.md` — speaking topics
- `.skill/references/reading-strategies.md` — 11 question types
- `.skill/references/listening-strategies.md` — section strategies
- `.skill/references/high-band-vocab.md` — vocab structure reference (each topic has ~10-30 starter words; expand to 150 with model knowledge)
- `.skill/references/youtube-channels.md` — channels to prefer in WebSearch
- `.skill/references/plan-templates.md` — calendar plan
- `.skill/templates/index.html` — index template with {{placeholders}}
- `.skill/templates/quiz.html` — quiz template
- `vocab-20260521.html` — concrete example to mimic for new days

---

## Failure modes & fallback

- **WebSearch returns no usable videos**: reuse 10 known-good videos from previous days
- **GitHub PUT 409 (sha mismatch)**: re-GET sha and retry
- **GitHub PUT 422 (bad content)**: log content size, ensure base64 is valid
- **Vocab generation under 150**: pad with idioms from `.skill/references/memory-deck-seed.md`
