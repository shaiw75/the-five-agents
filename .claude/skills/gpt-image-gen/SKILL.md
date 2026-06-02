---
name: gpt-image-gen
description: Use when an agent needs to generate an image from a text prompt. Wrapper around the OpenAI Images API (model gpt-image-2). Handles sending the prompt and saving the returned PNG.
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

מעטפת לקריאת **OpenAI Images API**. הסקיל מקבל prompt ונתיב יעד,
שולח את הבקשה ל-API, ושומר את התמונה המוחזרת כקובץ PNG.

## המודל — `gpt-image-2`

⚠️ **אל תשנה את שם המודל. אל תציע אלטרנטיבות.**

- `gpt-image-2` הוא מודל **אמיתי וקיים** של OpenAI.
- הוא יצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי שלך אינו כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אסור** להחליף אותו ב-`dall-e-3`, `gpt-image-1` או כל מודל אחר.
- אם קריאת ה-API נכשלת, הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY`
  או ב-parameters — **לא** בשם המודל. אל "תתקן" את שם המודל.

## דרישות מקדימות

- משתנה הסביבה `OPENAI_API_KEY` מוגדר ב-`.env` שבשורש הפרויקט.
- `curl` מותקן (זמין כברירת מחדל ברוב הסביבות).
- רצוי `jq` ו/או `python3` ל-decode של ה-base64 שחוזר.

## פרמטרים נתמכים

| פרמטר | ערך ברירת מחדל | הערות |
|-------|----------------|-------|
| `model` | `gpt-image-2` | קבוע — לא לשנות |
| `prompt` | (חובה) | תיאור התמונה |
| `size` | `1024x1024` | גם `1536x1024`, `1024x1536` |
| `quality` | `medium` | גם `low`, `high` |
| `output_format` | `png` | |

## איך קוראים ל-API

הקריאה משתמשת ב-`OPENAI_API_KEY` מתוך `.env`. טען אותו קודם:

```bash
set -a; source .env; set +a
```

### דרך 1 — עם jq (מועדף)

```bash
curl -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > <output-path>.png
```

### דרך 2 — Python fallback ל-decode

`jq` לא תמיד מותקן (למשל ב-Git Bash על Windows). במקרה כזה שומרים את
התגובה ומחלצים את ה-base64 עם Python:

```bash
curl -s -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > /tmp/gpt-image-resp.json

python3 - "$output_path" <<'PY'
import base64, json, sys
out = sys.argv[1]
with open("/tmp/gpt-image-resp.json", encoding="utf-8") as f:
    resp = json.load(f)
if "data" not in resp:
    # שגיאה מה-API — הדפס אותה כדי לאבחן (key / parameters, לא שם המודל)
    sys.exit("OpenAI API error: " + json.dumps(resp, ensure_ascii=False))
with open(out, "wb") as f:
    f.write(base64.b64decode(resp["data"][0]["b64_json"]))
print("saved:", out)
PY
```

## אימות

לאחר השמירה — ודא שהקובץ קיים וגודלו גדול מ-0:

```bash
test -s "$output_path" && echo "OK: $output_path ($(wc -c < "$output_path") bytes)" \
  || echo "FAILED: no image written to $output_path"
```

## טיפול בשגיאות

- **`401 Unauthorized`** → `OPENAI_API_KEY` חסר/שגוי ב-`.env`.
- **`400 Bad Request`** → בדוק את ה-parameters (size/quality/prompt).
- שדה `error` בתגובה במקום `data` → קרא את ההודעה; היא תצביע על
  ה-key או ה-parameters. **אל** תשנה את `model`.
