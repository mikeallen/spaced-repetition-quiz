# Quiz App with SPaced Repetition

A single page web app that quizes me on some details about japan I have in a tab separated file which has one line per prefecture


To run 
$ npm run dev

Then use a browser to access http://localhost:5173/

---

## Project Structure


```
spaced-repetition-quiz/
├── node_modules/
├── public/
├── src/
│   ├── App.jsx          # Main quiz app (code above)
│   ├── index.css        # Tailwind styles
│   └── main.jsx         # Entry point
├── .gitignore
├── index.html
├── package.json
├── tailwind.config.js
├── vite.config.js
└── README.md            # This file
```

---

## Development History

### Phase 1: Initial Development
- Created basic quiz structure with TSV file upload
- Implemented bidirectional questions (entity→detail and detail→entity)
- Added multiple choice (5 options) and free text modes

### Phase 2: Enhanced Question Logic
- Fixed comma-separated values with parentheses preservation
- Added importance weighting (80% importance=1, 20% importance=2)
- Implemented fuzzy matching for close answers with confirmation dialog
- Added "Don't Know" button

### Phase 3: Spaced Repetition System
- Implemented scheduling system:
  - Wrong answers: Review in 5 minutes
  - First correct: Review in 1 day
  - Second correct: Review in 3 days
  - Third+ correct: Review in 7 days
- Added question tracking with repeat counter (#0, #1, #2, etc.)
- Fixed priority system: 90% due questions (reviews), 10% new questions

### Phase 4: UI/UX Improvements
- Changed to 80% multiple choice, 20% free text
- Added AI-powered explanations via Anthropic API
- Made answers and multiple choice options clickable for explanations
- Added "What is [quoted text]?" links for terms in questions
- Implemented "Correct" override button
- Added "Ignore" button to permanently skip questions
- Always show repeat counter (even for first-time questions)

### Phase 5: Generic Data Support
- Made app work with any entity type (not just prefectures)
- Uses first column name dynamically in questions
- Generic titles ("Quiz" instead of "Japan Quiz")

### Phase 6: Bug Fixes & Optimization
- Fixed quote stripping in TSV parsing
- Ensured at least one correct answer in multiple choice
- Improved question generation algorithm to properly track and schedule reviews
- Added proper categorization of due vs new questions
- Implemented randomization within importance tiers

---

## TSV File Format

```tsv
Entity	Field1	Field2	Field3	importance
Value1	Data1	Data2	Data3	1
Value2	Data1	Data2	Data3	2
```

### Rules
- First row: Headers (field names)
- First column: Main entity (e.g., Prefecture, Country, City)
- `importance` column (optional): `1` for high priority, `2` for lower
- Empty values: Use `""` or leave blank
- Multiple values: Comma-separated (e.g., `Tokyo, Edo`)
- Parentheses: Commas inside are preserved (e.g., `Tokyo (Kanto), Osaka`)

---

## Features

✅ **Spaced Repetition** - Intelligent review scheduling
✅ **Multiple Choice & Free Text** - 80/20 split
✅ **Bidirectional Questions** - Tests both directions
✅ **Fuzzy Matching** - Accepts close answers
✅ **AI Explanations** - Powered by Claude API
✅ **Question Tracking** - See how many times asked
✅ **Ignore System** - Skip questions permanently
✅ **Override Corrections** - Mark wrong answers as correct
✅ **Importance Weighting** - Focus on priority items
✅ **Generic Data Support** - Works with any entity type

---

## Running with Claude CLI

Once you have the project set up:

```bash
# Start development server
npm run dev

# Work with Claude CLI for further development
claude code review src/App.jsx
claude code "add feature X to the quiz"
```

---

## Git Setup (Optional)

```bash
git init
git add .
git commit -m "Initial commit: Spaced repetition quiz app"
```

---

## Notes

- All data stays in browser memory (no persistence across sessions)
- AI explanations require internet connection
- Press Enter to advance to next question after seeing results
- Multiple choice options show where each answer came from

---

## License

Free to use and modify for personal and educational purposes.
