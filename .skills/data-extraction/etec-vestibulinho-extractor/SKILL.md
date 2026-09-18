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
r'Leia a tirinha\.\s*\n([\s\S]*?)(?=Questão)'

// Pattern 4: Song lyrics
r'Leia o trecho da canção[^\n]*\n([\s\S]*?)(?=Questão)'
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

### 5. Detect Image Questions
- Questions with `tinyurl` or similar shortened URLs have images
- Mark with `tem_imagem: true`
- Filter out when starting quiz: `allQuestions.filter(q => !q.tem_imagem)`

### 6. Complete Python Extraction Script
```python
import re
import subprocess
import json

def extract_all_questions(text, semester):
    # 1. Clean exam header
    text = re.sub(r'VESTIBULINHO[^\n]*\n?', '', text)
    text = re.sub(r'\f', '', text)
    text = re.sub(r'\n{3,}', '\n\n', text)
    
    support_blocks = {}
    
    # 2. Extract support texts FIRST (before questions)
    patterns = [
        (r'Leia o texto para responder às questões (\d+) e (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)', 'pair'),
        (r'Leia o texto para responder às questões de (\d+) a (\d+)\.\s*\n([\s\S]*?)(?=Questão\s+\d+|$)', 'range'),
        (r'Leia a tirinha\.\s*\n([\s\S]*?)(?=Questão)', 'tirinha'),
        (r'Leia o trecho da canção[^\n]*\n([\s\S]*?)(?=Questão)', 'song'),
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
                alts[letter] = text_alt[:500]  # Limit length
        
        if len(alts) >= 4:  # Valid question
            first_alt_pos = q_content.find('(A)')
            enunciado = q_content[:first_alt_pos].strip()[:1000] if first_alt_pos > 0 else q_content[:500]
            
            has_image = 'tinyurl' in enunciado.lower()
            
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

## Pitfalls
- **Gabarito parsing**: Two tables side-by-side - must use pattern that captures both simultaneously
- **URLs in text**: Remove tinyurl.com links before displaying
- **Image questions**: Filter these out in quiz unless user specifically wants them
- **pdftotext timing**: Use subprocess instead of Python library (faster)
- **GitHub PAT**: Token needs "Contents: Read and Write" permission, not just "Pull Requests"

## GitHub Push with PAT
```bash
git remote set-url origin https://USER:PAT@github.com/USER/repo.git
git push origin main
```
PAT must have: Contents → Read and Write

## Verification
- Check question count: should be 50 per semester
- Check gabarito: should have 50 answers
- Verify `tem_imagem` is correctly set for questions with images

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
