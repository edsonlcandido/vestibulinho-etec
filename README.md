# 📚 Estudo Vestibulinho ETEC

Ambiente interativo de estudo para o vestibulinho da ETEC.

## 🚀 Como usar

1. Abra o arquivo `index.html` no navegador
2. Escolha o modo de estudo:
   - **Simulado**: Questões na ordem da prova (vários semestres)
   - **Desafio Rápido**: Questões aleatórias de todos os semestres (10, 20 ou 40)

## 📁 Estrutura

```
estudo-vestibulinho/
├── index.html              # App principal
├── vestibulinho_data.json  # Questões e gabaritos
└── .skills/                # Skills para extração de dados
    ├── data-extraction/
    └── pdf-to-quiz-app/
```

## ✨ Funcionalidades

- ✅ **238 questões** de 5 provas (2024-2026)
- ✅ Simulado e Desafio Rápido
- ✅ Sistema de ajudas por dificuldade (Fácil, Médio, Difícil)
- ✅ Eliminar respostas erradas
- ✅ Textos de apoio extraídos automaticamente
- ✅ Gabaritos oficiais completos

## 📊 Provas Disponíveis

| Prova | Questões |
|-------|----------|
| 2º Semestre 2026 | 46 |
| 1º Semestre 2026 | 49 |
| 1º Semestre 2025 | 47 |
| 2º Semestre 2024 | 48 |
| 1º Semestre 2024 | 48 |

## 🎯 Dificuldade (Desafio)

| Nível | Ajudas |
|-------|--------|
| Fácil | Igual à quantidade de questões |
| Médio | Metade das questões |
| Difícil | Sem ajuda |

## 🔧 Extração de Dados

Para reextrair questões de novos PDFs:

```bash
# Extrair texto dos PDFs
pdftotext -layout "caderno.pdf" - > saida.txt

# Usar a skill etec-vestibulinho-extractor
```

## 📖 Sobre

Desenvolvido para ajudar nos estudos do vestibulinho ETEC.

**Autor**: Edson Luiz Candido  
**GitHub**: [edsonlcandido/vestibulinho-etec](https://github.com/edsonlcandido/vestibulinho-etec)
