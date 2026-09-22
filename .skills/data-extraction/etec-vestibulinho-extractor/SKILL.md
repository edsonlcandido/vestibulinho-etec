---
name: etec-vestibulinho-extractor
description: Extract questions from ETEC vestibulinho PDFs and create quiz data with support text and image filtering
---

# ETEC Vestibulinho Exam Extractor

Extract questions from ETEC vestibulinho PDFs and create quiz data.

## Trigger Conditions
- User provides ETEC vestibulinho exam PDFs (caderno + gabarito)
- Need to create study quiz from exam questions

## Steps

### 1. Extract Text from PDFs
```bash
pdftotext -layout "/path/to/caderno.pdf" "/path/to/caderno.txt"
pdftotext -layout "/path/to/gabarito.pdf" "/path/to/gabarito.txt"
```

### 2. Parse Gabarito (Answer Key)
The gabarito PDFs have **two tables side-by-side** (questões 1-25 na esquerda, 26-50 na direita).
Each line contains: `001 D C1 026 E C2`

**Python pattern that works for both tables:**
```python
def parse_gabarito(text):
    gab = {}
    for line in text.split('\n'):
        line_clean = re.sub(r'\s+', ' ', line.strip())
        # Find ALL matches in line (captures both tables)
        matches = re.findall(r'(\d{3})\s+([A-E])\s', line_clean)
        for q_num, resposta in matches:
            gab[int(q_num)] = resposta
    return gab
# Returns: {1: 'D', 2: 'C', ..., 26: 'E', ..., 50: 'A'}
```

Output: 50 questions per semester

### 3. Extract Questions with Support Text
Support text patterns in ETEC exams:
```javascript
// Pattern 1: "Leia o texto para responder às questões XX e YY"
r'Leia o texto para responder às questões (\d+) e (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)'

// Pattern 2: "Leia o texto para responder às questões de XX a YY"
r'Leia o texto para responder às questões de (\d+) a (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)'

// Pattern 3: Tirinha (comic strip - usually has image)
// IMPORTANT: Use Questão\s+\d not just Questão to avoid stopping at "questões"
r'Leia a tirinha\.\s*\n([\s\S]*?)(?=Questão\s+\d)'

// Pattern 4: Song lyrics
// IMPORTANT: Use Questão\s+\d not just Questão - the text contains "questões" which breaks the pattern
r'Leia o trecho da canção[^\n]*\n([\s\S]*?)(?=Questão\s+\d)'

// Song lyrics association: Extract question range from the full match
// The song text includes "questões de XX a YY" - use regex to find it
song_match = re.search(r'questões de (\d+) a (\d+)', original_text, re.IGNORECASE)
if song_match:
    start, end = int(song_match.group(1)), int(song_match.group(2))
    for q in range(start, end + 1):
        support_blocks[q] = song_text
```

### 4. Question Data Structure
```javascript
{
  num: 1,
  enunciado: "Question text...",
  alternativas: {A: "...", B: "...", C: "...", D: "...", E: "..."},
  texto_apoio: "Support text if exists",
  tem_imagem: true/false,
  gabarito: "D"
}
```

### 5. Detect Image Questions (CRITICAL - This detection is error-prone!)
**WARNING**: Not all questions with `tinyurl` links have images - some are just references.

**Better detection logic:**
```python
def tem_realmente_imagem(questao):
    enunciado = questao.get('enunciado', '').lower()
    texto_apoio = questao.get('texto_apoio', '').lower()
    
    # Definitely have image
    if 'observe a imagem' in enunciado: return True
    if 'observe a figura' in enunciado: return True
    if 'observe o gráfico' in enunciado: return True
    if 'observe o mapa' in enunciado: return True
    if 'analise a imagem' in enunciado: return True
    if 'analise a figura' in enunciado: return True
    if 'analise o quadrinho' in enunciado: return True
    if 'observe o esquema' in enunciado: return True
    if 'leia a tirinha' in enunciado: return True
    if 'leia a charge' in enunciado: return True
    if 'tirinha' in enunciado[:200]: return True
    if 'quadrinho' in enunciado[:200]: return True
    if 'charge' in enunciado[:200]: return True
    
    # Check texto_apoio for short content that looks like image description
    if texto_apoio and len(texto_apoio) < 200:
        if any(w in texto_apoio for w in ['imagem', 'figura', 'tirinha', 'charge', 'quadrinho']):
            return True
    
    return False
```

**Simple approach (less accurate):**
- Questions with `tinyurl` in enunciado MAY have images
- But `tinyurl` can also be just a reference link in the text
- If in doubt, mark as `tem_imagem: false` and let the quiz filter handle it later

**Filter in quiz app:**
```javascript
allQuestions = allQuestions.filter(q => !q.tem_imagem);
```

### 6. Complete Python Extraction Script
```python
import re
import subprocess
import json

def tem_realmente_imagem(questao):
    """Detect if question actually has an image (not just a tinyurl reference)"""
    enunciado = questao.get('enunciado', '').lower()
    texto_apoio = questao.get('texto_apoio', '').lower()
    
    if 'observe a imagem' in enunciado: return True
    if 'observe a figura' in enunciado: return True
    if 'observe o gráfico' in enunciado: return True
    if 'observe o mapa' in enunciado: return True
    if 'analise a imagem' in enunciado: return True
    if 'analise a figura' in enunciado: return True
    if 'analise o quadrinho' in enunciado: return True
    if 'observe o esquema' in enunciado: return True
    if 'leia a tirinha' in enunciado: return True
    if 'leia a charge' in enunciado: return True
    if 'tirinha' in enunciado[:200]: return True
    if 'quadrinho' in enunciado[:200]: return True
    if 'charge' in enunciado[:200]: return True
    if texto_apoio and len(texto_apoio) < 200:
        if any(w in texto_apoio for w in ['imagem', 'figura', 'tirinha', 'charge']):
            return True
    return False

def extract_all_questions(text, semester):
    # 1. Clean exam header
    text = re.sub(r'VESTIBULINHO[^\n]*\n?', '', text)
    text = re.sub(r'\f', '', text)
    text = re.sub(r'\n{3,}', '\n\n', text)
    
    support_blocks = {}
    
    # 2. Extract support texts FIRST (before questions)
    # IMPORTANT: Use Questão\s+\d to avoid stopping at "questões"
    patterns = [
        (r'Leia o texto para responder às questões (\d+) e (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)', 'pair'),
        (r'Leia o texto para responder às questões de (\d+) a (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)', 'range'),
        (r'Leia a tirinha\.\s*\n([\s\S]*?)(?=Questão\s+\d)', 'tirinha'),
        (r'Leia o trecho da canção[^\n]*\n([\s\S]*?)(?=Questão\s+\d)', 'song'),
    ]
    
    for pattern, type_ in patterns:
        for m in re.finditer(pattern, text):
            if type_ == 'pair':
                q1, q2 = int(m.group(1)), int(m.group(2))
                support_text = ' '.join(m.group(3).strip().split())[:2000]
                support_blocks[q1] = support_text
                support_blocks[q2] = support_text
            elif type_ == 'range':
                start, end = int(m.group(1)), int(m.group(2))
                support_text = ' '.join(m.group(3).strip().split())[:2000]
                for q in range(start, end + 1):
                    support_blocks[q] = support_text
            elif type_ == 'song':
                # Find question range from full match text
                song_text = ' '.join(m.group(1).strip().split())[:2000]
                original = m.group(0)
                range_match = re.search(r'questões de (\d+) a (\d+)', original, re.IGNORECASE)
                if range_match:
                    start, end = int(range_match.group(1)), int(range_match.group(2))
                    for q in range(start, end + 1):
                        support_blocks[q] = song_text
            else:
                support_blocks[type_] = ' '.join(m.group(1).strip().split())[:2000]
    
    # 3. Split and parse questions
    parts = re.split(r'Questão\s+(\d{1,2})', text)
    questions = []
    
    for i in range(1, len(parts), 2):
        q_num = int(parts[i].strip())
        q_content = parts[i+1] if i+1 < len(parts) else ""
        q_content = ' '.join(q_content.split())
        
        # 4. Extract alternatives
        alt_pattern = r'\(([A-E])\)\s*([^\(](?:.*?))?(?=\([A-E]\)|$)'
        alt_matches = re.findall(alt_pattern, q_content, re.DOTALL)
        
        alts = {}
        for letter, text_alt in alt_matches[:5]:
            text_alt = ' '.join(text_alt.split())
            if text_alt:
                alts[letter] = text_alt[:500]
        
        if len(alts) >= 4:
            first_alt_pos = q_content.find('(A)')
            enunciado = q_content[:first_alt_pos].strip()[:1000] if first_alt_pos > 0 else q_content[:500]
            
            # Use better image detection
            has_image = tem_realmente_imagem({'enunciado': enunciado, 'texto_apoio': support_blocks.get(q_num, '')})
            
            questions.append({
                'num': q_num,
                'enunciado': enunciado,
                'alternativas': alts,
                'texto_apoio': support_blocks.get(q_num, ""),
                'tem_imagem': has_image
            })
    
    return sorted(questions, key=lambda x: x['num'])

def parse_gabarito(text):
    gab = {}
    for line in text.split('\n'):
        line_clean = re.sub(r'\s+', ' ', line.strip())
        matches = re.findall(r'(\d{3})\s+([A-E])\s', line_clean)
        for q_num, resposta in matches:
            gab[int(q_num)] = resposta
    return gab
```

## Adding New Exams

When new PDF exams arrive, update the extraction script with all files:

```python
arquivos = [
    ('1SEM2024', 'caderno.pdf', 'gabarito.pdf'),
    ('1SEM2025', 'caderno.pdf', 'gabarito.pdf'),
    # ... add new ones
]

# Then run extraction - app automatically shows new exams
```

**App pattern**: The quiz app generates buttons dynamically from JSON keys:
```javascript
function generateSimuladoButtons() {
    const semesters = Object.keys(quizData).sort();
    semesters.forEach(key => {
        // Create button from quizData[key]
    });
}
```
No HTML changes needed when adding new exams!

## Common texto_apoio Issues (IMPORTANT - Verify After Extraction!)

The regex-based extraction may miss texto_apoio in these cases:

### 1. Text is embedded in enunciado (DUPLICATED)
Some questions have the support text inside the enunciado field itself (not as a separate block).
Look for questions with "Baseando-se no texto" and check if they have `texto_apoio: ""`.

**How to detect:**
```python
for key, val in data.items():
    for q in val['questoes']:
        if 'Baseando-se no texto' in q.get('enunciado', ''):
            if not q.get('texto_apoio', '').strip():
                print(f"MISSING: {key} Q{q['num']}")
```

**Fix: Manually extract from PDF and separate:**
```python
# Bad: texto_apoio is embedded in enunciado
q['enunciado'] = "A águia-de-asa-redonda... texto longo ... Baseando-se no texto, sobre essa águia..."

# Good: Separate them
q['texto_apoio'] = "A águia-de-asa-redonda é uma ave de rapina..."
q['enunciado'] = "Baseando-se no texto, sobre essa águia é correto afirmar que"
```

### 2. Multi-question support text (e.g., 34-36, 25)
When a single text supports multiple questions, the text might be split across PDF pages with images between sections.

**Fix:**
```bash
# Extract text before specific question from PDF
pdftotext -layout "caderno.pdf" - | grep -B 100 "Questão 34" | head -50
```

### 4. Questions with "Baseando-se no texto" WITHOUT separate texto_apoio
Some questions (like Q25, Q34, Q35 in various semesters) have the support text BEFORE the question header in the PDF, not after. This causes regex to miss them.

**Detection pattern:**
```python
for key, val in data.items():
    for q in val['questoes']:
        if 'Baseando-se no texto' in q.get('enunciado', ''):
            if not q.get('texto_apoio', '').strip():
                print(f"MISSING: {key} Q{q['num']}")
```

**Fix - manual extraction from PDF:**
```bash
# Find text before question
pdftotext -layout "caderno.pdf" - | grep -B 100 "Questão 34" | head -50
```

### 5. Song Lyrics (Frequent Issue)
Songs like "O Trenzinho do Caipira" have lyrics that appear BEFORE the question. Look for:
- "Leia a letra da música" or "Leia o trecho da canção"
- "questões de XX a YY" in the header

**Known songs to verify:**
- 2SEM2026: "O Trenzinho do Caipira" (Ferreira Gullar / Heitor Villa-Lobos) → Q34, Q35

### 6. Known Problematic Questions (Verify After Every Extraction)
Always check these question numbers for missing texto_apoio:
- Q25 (often has texto before header)
- Q34, Q35, Q36 (often share a common text)
- Any question with "Baseando-se no texto" in enunciado

### 7. Post-Extraction Verification Checklist
After running extraction, ALWAYS verify:
1. Questions with "Baseando-se no texto" have non-empty `texto_apoio`
2. Questions 25, 34-36 of any semester have texto_apoio if the PDF has a text before them
3. No questions have both texto_apoio AND the same text in enunciado (duplicated)

## Pitfalls
- **"Baseando-se no texto" questions**: Often have texto_apoio NOT captured by regex. MANUALLY VERIFY EACH ONE.
- **Song/support text regex**: Use `Questão\s+\d` not just `Questão` - text contains "questões" which breaks pattern
- **Song question association**: After extracting song text, search full match for "questões de XX a YY"
- **Image detection**: Not all with `tinyurl` have images - some are just reference links
- **Gabarito parsing**: Two tables side-by-side - use pattern that captures both simultaneously
- **URLs in text**: Remove tinyurl.com links before displaying
- **pdftotext**: Use subprocess instead of Python library (faster)
- **GitHub PAT**: Token needs "Contents: Read and Write" permission
- **Question number 0**: Indicates extraction failure - verify and fix manually
- **Texto deapoio duplicated in enunciado**: Fix manually by separating text from question

## GitHub Push with PAT
```bash
git remote set-url origin https://USER:PAT@github.com/USER/repo.git
git push origin main
```
PAT must have: Contents → Read and Write

## Verification
- Check question count: should be 50 per semester (some may be filtered due to images)
- Check gabarito: should have 50 answers
- Verify `tem_imagem` is correctly set - questions with tinyurl but no actual image should be marked `false`
- Check questions 21-24 of 1st semester specifically - these often have song lyrics as support text
- Verify no questions have `num: 0` - this indicates extraction failure
- Check for duplicate question numbers

## Example JSON Structure
```json
{
  "1sem": {
    "nome": "1º Semestre 2026",
    "questoes": [...],
    "gabarito_oficial": {"1": "D", "2": "C", ...}
  },
  "2sem": {
    "nome": "2º Semestre 2026", 
    "questoes": [...],
    "gabarito_oficial": {"1": "C", "2": "B", ...}
  }
}
```
