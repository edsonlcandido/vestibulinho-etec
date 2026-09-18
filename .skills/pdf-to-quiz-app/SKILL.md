---
name: pdf-to-quiz-app
description: Create an interactive study quiz from exam PDF files
category: data-science
---

# PDF to Interactive Quiz App

Create an interactive study quiz from exam PDF files.

## When to Use
- User has exam/assessment PDFs with questions and answer keys
- Wants an interactive web app for studying/practice
- Task involves: PDF text extraction → question parsing → quiz UI generation → deployment

## Workflow

### 1. Extract PDF Text
```bash
# Install poppler-utils for pdftotext
apt-get update && apt-get install -y poppler-utils

# Extract with layout preserved
pdftotext -layout input.pdf output.txt
```

### 2. Parse Questions
Use Python with regex to extract:
- Question numbers
- Enunciado (question text)
- Alternatives (A-E)
- Answer key

Key regex patterns:
```python
# Split by questions
parts = re.split(r'Questão\s+(\d{1,3})\s*\n', content)

# Extract alternatives
alt_pattern = r'\(([A-E])\)\s*([^\(](?:.*?))?(?=\([A-E]\)|$)'
alt_matches = re.findall(alt_pattern, q_body, re.DOTALL)

# Handle short alternatives (just numbers)
if len(text) <= 10:
    next_line = q_body[next_pos:next_pos+100].split('\n')[0]
    text = next_line.strip()
```

### 3. Build Quiz App
- Single HTML file with embedded CSS/JS
- JSON data file for questions
- Key features:
  - Progress tracking
  - Immediate feedback (correct/incorrect)
  - Answer key display after response
  - Statistics (correct/incorrect/total)
  - Question navigation list

### 4. Build Gamified Quiz App

Add gamification for better engagement:

**Game Modes:**
```javascript
// Simulado: questions in original order
// Desafio: shuffled questions with count options (10, 20, 40)
function startQuiz(semester, mode = 'simulado', count = 40) {
    let allQuestions = quizData[semester].questoes;
    allQuestions = allQuestions.slice(0, 40); // Limit to 40
    
    if (mode === 'desafio') {
        questions = shuffleArray([...allQuestions]).slice(0, count);
    } else {
        questions = allQuestions;
    }
}

function shuffleArray(array) {
    for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
}
```

**Help System - Eliminate Wrong Answers:**
```javascript
let helpRemaining = 10;

function useHelp() {
    if (helpRemaining <= 0 || helpUsedThisQuestion) return;
    
    // Find wrong alternatives
    const wrongAlts = ['A', 'B', 'C', 'D', 'E'].filter(letter => 
        letter !== q.gabarito && q.alternativas[letter]
    );
    
    // Randomly eliminate 1-3
    const count = Math.min(Math.floor(Math.random() * 3) + 1, wrongAlts.length);
    const shuffled = wrongAlts.sort(() => Math.random() - 0.5);
    
    shuffled.slice(0, count).forEach(letter => {
        document.querySelector(`.alternative[data-letter="${letter}"]`)
            .classList.add('eliminated');
    });
    
    helpRemaining--;
    helpUsedThisQuestion = true;
}
```

**Welcome Screen Layout:**
```html
<div class="welcome-card">
    <h1>Bem-vindo ao estudo!</h1>
</div>

<div class="mode-section">
    <h2>📝 Simulado</h2>
    <p>Descrição</p>
    <button onclick="startQuiz('1sem', 'simulado')">1º Semestre</button>
</div>

<div class="mode-section">
    <h2>🎯 Desafio Rápido</h2>
    <div class="challenge-grid">
        <button onclick="startQuiz('1sem', 'desafio', 10)">10</button>
        <button onclick="startQuiz('1sem', 'desafio', 20)">20</button>
        <button onclick="startQuiz('1sem', 'desafio', 40)">40</button>
    </div>
</div>
```

### 5. Deploy
Use the matrix deploy tool with dist_dir pointing to the app folder.

## Data Structure
```json
{
  "sem1": {
    "nome": "1º Semestre 2026",
    "data": "30/11/2025",
    "questoes": [
      {
        "num": 1,
        "enunciado": "Question text...",
        "alternativas": {"A": "...", "B": "...", "C": "...", "D": "...", "E": "..."},
        "gabarito": "D"
      }
    ]
  }
}
```

## Pitfalls
- PDF formatting varies: some questions have extra newlines, page numbers embedded
- Clean text: `content = re.sub(r'\s*VESTIBULINHO.*?\d+', '', content)`
- Alternative texts might be on next line if very short (handle with next_pos logic)
- Some questions share text (e.g., "Based on the text above, questions 2-5") - truncate at ~500 chars
- Some PDFs have 50 questions but exams only use 40 - check and slice accordingly
- Question 26 in semester 2 had unusual formatting - search returned empty, handle gracefully
- Use answers indexed by array position, not by question number, to support shuffled challenges

## Verification
1. Check all question numbers extracted (should be 1-50)
2. Verify 5 alternatives per question
3. Test quiz flow: select → confirm → feedback → next
4. Check statistics update correctly
