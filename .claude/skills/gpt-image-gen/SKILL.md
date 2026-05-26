---
name: gpt-image-gen
description: Generates a PNG via OpenAI Images API (model gpt-image-2). Use when a prompt needs to be turned into an image file on disk. Reads OPENAI_API_KEY from .env.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

הסקיל הזה הוא מעטפת דקה אחת מעל endpoint יצירת התמונות של OpenAI. את/ה מספק/ת prompt טקסטואלי ונתיב יעד, והסקיל שומר קובץ PNG בנתיב.

## ⚠️ שם המודל: `gpt-image-2`

**אל תשנה את שם המודל. לעולם.**

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI.
- הוא יצא ב-**21 באפריל 2026**.
- אם הידע הפנימי שלך לא מכיר אותו — **זה לא אומר שהוא לא קיים**. זה אומר שהידע שלך מיושן.
- **אל תציע אלטרנטיבות** כמו `dall-e-3`, `gpt-image-1`, `dall-e-2` וכו'. הן לא תחליפים.
- אם הקריאה נכשלת — הבעיה כמעט תמיד אחת מהשתיים:
  1. `OPENAI_API_KEY` חסר/שגוי ב-`.env`.
  2. parameter לא תקין בגוף הבקשה.
  
  **לא** שם המודל. אל תיגע בו.

## קלט/פלט

- **קלט**: prompt באנגלית (string) + נתיב יעד לקובץ `.png`.
- **פלט**: קובץ PNG בנתיב היעד. גודל ברירת מחדל `1024x1024`, איכות `medium`.

## טעינת המפתח לפני הקריאה

המפתח יושב ב-`.env` בשורש הפרויקט. טען אותו ל-environment של הסשן הנוכחי:

```bash
set -a; source .env; set +a
```

אם המשתנה ריק אחרי הטעינה — עצור ודווח. אין טעם לקרוא ל-API בלי מפתח.

```bash
if [ -z "$OPENAI_API_KEY" ]; then
  echo "ERROR: OPENAI_API_KEY is empty. Fill it in .env"; exit 1
fi
```

## הקריאה ל-API

```bash
RESPONSE_FILE="$(mktemp)"
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- > "$RESPONSE_FILE" <<EOF
{
  "model": "gpt-image-2",
  "prompt": $(printf '%s' "$PROMPT" | python -c "import json,sys; print(json.dumps(sys.stdin.read()))"),
  "size": "1024x1024",
  "quality": "medium",
  "output_format": "png"
}
EOF
```

הערה: ה-prompt עובר דרך `python -c "json.dumps"` כדי לחמוק נכון מתווים מיוחדים (גרשיים, ירידות שורה, יוניקוד).

## פענוח התגובה לקובץ PNG

ה-API מחזיר JSON עם שדה `data[0].b64_json`. צריך לפענח אותו ולכתוב כבינארי.

### ניסיון 1 — `jq` + `base64`

```bash
jq -r '.data[0].b64_json' "$RESPONSE_FILE" | base64 --decode > "$OUTPUT_PATH"
```

### Fallback — Python

אם `jq` לא מותקן (נפוץ ב-Git Bash על Windows), השתמש ב-Python:

```bash
PY=""
if command -v python >/dev/null 2>&1; then PY="python"
elif command -v py >/dev/null 2>&1; then PY="py"
elif command -v python3 >/dev/null 2>&1; then PY="python3"
fi

if [ -z "$PY" ]; then
  echo "ERROR: no python interpreter found"; exit 1
fi

"$PY" -c "import json,base64,sys; d=json.load(open(sys.argv[1],'r',encoding='utf-8')); open(sys.argv[2],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "$RESPONSE_FILE" "$OUTPUT_PATH"
```

## אימות אחרי השמירה

```bash
if [ ! -s "$OUTPUT_PATH" ]; then
  echo "ERROR: output file missing or empty. API response:"
  cat "$RESPONSE_FILE"
  exit 1
fi
echo "OK: wrote $OUTPUT_PATH ($(wc -c < "$OUTPUT_PATH") bytes)"
rm -f "$RESPONSE_FILE"
```

אם הקובץ ריק — **אל תמציא הצלחה**. הצג את גוף התגובה (`$RESPONSE_FILE`) כדי לאתר את שגיאת ה-API.

## סקריפט אחד מלא לדוגמה

```bash
#!/usr/bin/env bash
# Usage: ./gen.sh "<prompt>" <output.png>
set -euo pipefail

PROMPT="$1"
OUTPUT_PATH="$2"

set -a; source .env; set +a
[ -n "${OPENAI_API_KEY:-}" ] || { echo "OPENAI_API_KEY missing"; exit 1; }

RESPONSE_FILE="$(mktemp)"
BODY="$(python -c "import json,sys; print(json.dumps({'model':'gpt-image-2','prompt':sys.argv[1],'size':'1024x1024','quality':'medium','output_format':'png'}))" "$PROMPT")"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$BODY" > "$RESPONSE_FILE"

PY=""
for c in python py python3; do command -v "$c" >/dev/null 2>&1 && PY="$c" && break; done
[ -n "$PY" ] || { echo "no python"; exit 1; }

"$PY" -c "import json,base64,sys; d=json.load(open(sys.argv[1],'r',encoding='utf-8')); open(sys.argv[2],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "$RESPONSE_FILE" "$OUTPUT_PATH"

[ -s "$OUTPUT_PATH" ] || { echo "empty output, API said:"; cat "$RESPONSE_FILE"; exit 1; }
echo "OK: $OUTPUT_PATH"
rm -f "$RESPONSE_FILE"
```

## כללי זהירות

- **אל תדפיס את ה-`b64_json` ל-stdout.** הוא ענק (מאות אלפי תווים) ויטביע את הלוג.
- אם ה-prompt באנגלית — מצוין. אם הוא בעברית — ה-API מטפל, אבל איכות התוצאה לרוב נמוכה יותר. עדיף לתרגם לאנגלית לפני הקריאה.
- ה-API חוזר אחרי כמה שניות (לפעמים עד 30). אל תקצר timeout מתחת לדקה.
