# Start here — Log-Triage SLM (Week 1 bundle)

You don't need to understand the code to run this. Just follow the steps in order.

## What's already done for you
- The 4 log datasets are downloaded (in `data/raw/`).
- Parsing is done: `data/labeled/records_partial.jsonl` = 8000 clean records,
everything filled in EXCEPT the `recommended_action` field.
- `data/labeled/eyeball_50.txt` = 50 sample records you can just read to sanity-check.

## What you do now

### 1. Open this folder in Antigravity
File > Open Folder > pick this `log-triage-slm-starter` folder. Trust it when asked.

### 2. Open the terminal inside Antigravity
Top menu: Terminal > New Terminal. A command box opens at the bottom. You type commands there.

### 3. Install the 3 tools (one time). Paste and press enter:
    pip install -r requirements.txt

### 4. Get a free API key (the "teacher" that fills the last field). Pick ONE:
- Groq:   console.groq.com  -> make a free key
- Gemini: aistudio.google.com -> make a free key

Tell the terminal your key (use YOUR real key, keep the quotes):
- Groq:   export GROQ_API_KEY="your_key_here"
- Gemini: export GEMINI_API_KEY="your_key_here"
(Windows PowerShell instead:  $env:GROQ_API_KEY="your_key_here")
# 

### 5. Test run (no key needed) - just checks nothing is broken:
    python src/label_with_teacher.py --provider mock

If it prints "groups labeled" with no errors, you're good.
WARNING: the file this makes is FAKE (dumb keyword guessing). Don't train on it.
Delete it before the real run:
    rm data/labeled/action_cache.jsonl data/labeled/records_labeled.jsonl

### 6. Real run - the teacher actually labels.
Ask the provider's site for a CURRENT free model name to put after --model.
Groq example:
    python src/label_with_teacher.py --provider groq --model llama-3.3-70b-versatile
Gemini example:
    python src/label_with_teacher.py --provider gemini --model gemini-2.0-flash

It labels ~222 log patterns (a few minutes). If it stops, just run it again -
it remembers what it already finished.

### 7. THE ONE REAL JOB THAT'S YOURS: check the labels by eye
Open `data/labeled/groups_for_review.csv` (double-click in Antigravity, or open in Excel).
Each row says "this kind of log -> this action".
- Fix any that look wrong (edit the recommended_action cell).
- Any row saying NEEDS_REVIEW must be changed to a real action.
This is the part that earns marks. ~222 rows, doable in one sitting.

That's Week 1 done. Next chat, say:
"Week 2 - split, freeze eval set, write scorer, score the untuned baseline."

## If something breaks
Copy the red error text, paste it into the chat. Don't guess at fixes.
