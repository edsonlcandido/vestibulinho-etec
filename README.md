# 📚 Estudo Vestibulinho ETEC

Ambiente interativo de estudo para o vestibulinho da ETEC.

## 🚀 Como usar

1. Abra o arquivo `index.html` no navegador
2. Escolha o modo de estudo:
   - **Simulado**: 40 questões na ordem
   - **Desafio Rápido**: questões aleatórias (10, 20 ou 40)

## 📁 Estrutura

```
estudo-vestibulinho/
├── index.html              # App principal
├── vestibulinho_data.json  # Questões e gabaritos
├── materials/              # PDFs e textos de apoio
│   ├── A-CADERNO-VESTIBULINHO-1SEM2026.pdf
│   ├── A-CADERNO-VESTIBULINHO-2SEM2026.pdf
│   ├── A-GABARITO-VESTIBULINHO-1SEM2026.pdf
│   ├── A-GABARITO-VESTIBULINHO-2SEM2026.pdf
│   ├── A-CADERNO-VESTIBULINHO-1SEM2026.txt
│   └── A-CADERNO-VESTIBULINHO-2SEM2026.txt
└── .skills/                # Skills para extração de dados
    ├── data-extraction/
    └── pdf-to-quiz-app/
```

## ✨ Funcionalidades

- ✅ 50 questões por semestre (sem imagens)
- ✅ Textos de apoio extraídos automaticamente
- ✅ Modo simulado e desafio
- ✅ Ajuda "Eliminar Respostas" (10 por sessão)
- ✅ Gabaritos oficiais completos

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
