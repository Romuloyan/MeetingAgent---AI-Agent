# MeetingAgent — AI Agent

Agente local para apoio à criação de atas a partir de transcrições de reuniões.

## Estado atual

MVP funcional.

O fluxo atual permite:

1. Usar transcrição gerada no Google Colab.
2. Usar diarização com identificação de `SPEAKER_00`, `SPEAKER_01`, etc.
3. Atribuir nomes reais aos oradores através de interface Streamlit.
4. Gerar transcrição com nomes reais.
5. Carregar documentos opcionais da reunião.
6. Gerar ata preliminar com Gemini.
7. Exportar ficheiros `.txt` e `.docx`.

## Estrutura

- `app.py` — aplicação principal Streamlit.
- `requirements.txt` — dependências Python.
- `.gitignore` — exclusões de ficheiros sensíveis e resultados locais.

## Segurança

Este repositório não deve conter:

- ficheiros `.env`;
- chaves API;
- áudios;
- transcrições reais;
- atas reais;
- ficheiros pessoais ou sensíveis.

A chave Gemini deve ficar num ficheiro local `.env` com:

```env
GEMINI_API_KEY=a_tua_chave
```

## Como correr

```powershell
cd "C:\Users\Pc\OneDrive\Agente IA\MeetingAgent"
python -m venv .venv
& .\.venv\Scripts\python.exe -m pip install -r requirements.txt
& .\.venv\Scripts\python.exe -m streamlit run app.py
```

## Entradas esperadas

O agente espera encontrar ficheiros gerados pelo pipeline de Colab, em especial:

- `*_segmentos_oradores.json`

Depois da identificação de oradores, gera:

- `*_segmentos_com_nomes.json`
- `*_transcricao_com_nomes.txt`
- `*_transcricao_com_nomes.docx`
- `*_mapa_oradores.json`

Depois da ata com Gemini, gera:

- `*_ata_preliminar_gemini.txt`
- `*_ata_preliminar_gemini.docx`
- `*_analise_intermedia_gemini.txt`
- `*_tarefas_pendentes_gemini.txt`

## Próximas melhorias

- Modo de ata formal institucional.
- Separar ata formal de relatório interno.
- Campo próprio para ordem de trabalhos formal.
- Relatório de validação separado.
- Botões de download dos ficheiros gerados.
- Melhor gestão de modelos Gemini e limites temporários de serviço.
