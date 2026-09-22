# MeetingAgent — AI Agent

MeetingAgent is a local AI-assisted workflow for turning meeting recordings into speaker-labelled transcripts, structured meeting notes, draft minutes, and action-item reports.

The project was built as a practical MVP for Portuguese association/club meeting workflows, especially meetings where the final minutes need to be written in formal European Portuguese. The repository documentation is written in English for portfolio and project presentation purposes, while the generated minutes are currently optimized for Portuguese.

The system is intentionally **human-in-the-loop**: it automates transcription, speaker diarization, speaker-name assignment support, context loading, and draft generation, but the final meeting minutes should always be reviewed by a human before official use.

---

## What the project does

MeetingAgent connects three main stages:

1. **Google Colab transcription notebook**  
   Converts audio/video recordings into timestamped transcripts using `faster-whisper`.

2. **Google Colab diarization notebook**  
   Uses `pyannote.audio` to detect speaker turns and creates transcript files labelled with `SPEAKER_00`, `SPEAKER_01`, etc.

3. **Local Streamlit MeetingAgent app**  
   Runs on the user's computer, lets the user assign real names to the detected speaker labels, loads optional meeting context documents, and uses Gemini to generate draft meeting minutes and task reports.

The goal is not to replace the secretary or meeting reviewer. The goal is to reduce the manual workload by producing structured, reviewable drafts based on the audio transcript, speaker diarization, and optional meeting documents.

---

## Why Google Colab was used

The transcription and diarization stages are computationally expensive.

During development, the local laptop had an integrated GPU / non-CUDA graphics setup, which was much slower for this type of workload. Running Whisper and pyannote locally was possible in principle, but it would significantly increase processing time, especially for long meetings.

Google Colab was therefore chosen for the first two stages because Colab can provide access to NVIDIA T4 GPUs. The T4 runtime was used to speed up:

- `faster-whisper` transcription;
- audio preprocessing workflows;
- `pyannote.audio` speaker diarization;
- processing of long meetings of around two hours.

This design choice keeps the heavy audio processing in Colab while keeping the review, speaker naming, document upload, and minutes-generation workflow local and interactive through Streamlit.

The architecture is therefore:

```text
Heavy audio processing → Google Colab / T4 GPU
Human review and minutes workflow → Local Streamlit app
```

This hybrid approach was selected to balance speed, cost, usability, and privacy.

---

## Current status

Functional MVP.

The current version can:

- read diarized transcript files generated in Google Colab;
- list all detected speakers;
- show useful sample phrases for each speaker;
- let the user assign real names to `SPEAKER_00`, `SPEAKER_01`, etc.;
- merge duplicated speaker labels by assigning the same real name;
- generate a transcript with real speaker names;
- upload optional meeting documents such as agendas, notes, attachments, or preliminary drafts;
- split long transcripts into manageable analysis blocks;
- call Gemini with retry and fallback logic;
- generate draft meeting minutes;
- generate intermediate analysis files;
- generate a task and pending-action report;
- export `.txt` and `.docx` files.

---

## Full workflow overview

```text
Audio recording
   ↓
Colab Notebook 1 — Transcription
   ↓
Timestamped transcript + JSON segments + SRT + DOCX
   ↓
Colab Notebook 2 — Speaker diarization
   ↓
Speaker-labelled transcript with SPEAKER_00, SPEAKER_01, ...
   ↓
Local MeetingAgent app
   ↓
Manual speaker-name assignment
   ↓
Transcript with real names
   ↓
Optional meeting context upload
   ↓
Gemini analysis and draft minutes generation
   ↓
DOCX/TXT exports
```

---

## Repository structure

```text
MeetingAgent---AI-Agent/
├── app.py
├── requirements.txt
├── README.md
├── .gitignore
└── colab/
    ├── README.md
    ├── 01_transcription_faster_whisper_colab.py
    └── 02_diarization_pyannote_one_click_colab.py
```

### Main files

- `app.py` — main local Streamlit application.
- `requirements.txt` — Python dependencies for the local app.
- `.gitignore` — excludes secrets, local data, meeting files, generated outputs, and media files.
- `colab/README.md` — documentation for the Colab-side workflow.
- `colab/01_transcription_faster_whisper_colab.py` — transcription notebook code, stored as a Python script for easier version control.
- `colab/02_diarization_pyannote_one_click_colab.py` — diarization notebook code, stored as a Python script for easier version control.

The Colab code is currently stored as `.py` source files rather than `.ipynb` notebooks. These files document the working notebook logic and can be copied into Google Colab cells or converted into notebooks later.

---

## Colab Notebook 1 — Transcription

The first Colab notebook processes audio or video files stored in Google Drive.

### Purpose

The notebook automatically finds audio/video files in the configured Drive folder, transcribes them, and exports several useful formats.

### Main technologies

- Google Colab
- Google Drive mount
- `faster-whisper`
- `ffmpeg`
- `python-docx`

### Model configuration used during development

```python
MODELO = "large-v3"
IDIOMA = "pt"
```

The notebook detects whether CUDA/GPU is available:

```text
GPU/CUDA available → device="cuda", compute_type="float16"
CPU only           → device="cpu", compute_type="int8"
```

### Supported input formats

```text
.m4a, .mp3, .wav, .aac, .flac, .mp4, .mov, .mkv, .webm, .ogg
```

### Expected Google Drive folder

During development, the working folder was:

```text
/content/drive/MyDrive/Transcrições e atas
```

### Output files

For each recording, the transcription notebook creates a result folder and exports:

```text
<meeting_name>_transcricao_timestamp.txt
<meeting_name>_transcricao_corrida.txt
<meeting_name>_legendas.srt
<meeting_name>_segmentos.json
<meeting_name>_transcricao.docx
<meeting_name>_TRANSCRICAO_COMPLETA.zip
```

The most important file for the next stage is:

```text
<meeting_name>_segmentos.json
```

That JSON file contains the transcript split into timestamped segments.

---

## Colab Notebook 2 — Speaker diarization

The second Colab notebook uses the transcript generated by Notebook 1 and the original audio file to detect speakers.

### Purpose

The diarization notebook answers:

```text
Who spoke when?
```

It does not transcribe the audio again. It uses the original audio to identify speech intervals and then matches those intervals with the transcript segments generated by Whisper.

### Main technologies

- Google Colab
- Google Drive mount
- `pyannote.audio`
- Hugging Face token stored in Colab Secrets
- `ffmpeg`
- `python-docx`

### Hugging Face access

The diarization notebook requires access to the pyannote model through a Hugging Face token.

During development, the token was stored in Colab Secrets as:

```text
TOKEN
```

The token itself is not stored in this repository.

### Diarization model

```text
pyannote/speaker-diarization-community-1
```

### One-click subprocess design

The final working diarization notebook was designed as a one-click Colab workflow.

The notebook does **not** import `pyannote` directly in the active Colab kernel after installing dependencies. Instead, it:

1. mounts Google Drive;
2. installs the required packages;
3. writes an external Python runner script;
4. launches that runner in a clean Python subprocess;
5. imports and runs `pyannote.audio` inside that subprocess;
6. exports results back to Google Drive.

This design avoided dependency conflicts involving NumPy, Torch, Numba, and pyannote after runtime package installation.

### Audio preprocessing

Before diarization, the notebook converts the original audio to a clean WAV file:

```text
16 kHz
mono
PCM signed 16-bit
```

This avoids sample mismatch problems with compressed formats such as `.m4a`.

### Output files

For each meeting, the diarization notebook creates:

```text
<meeting_name>_segmentos_oradores.json
<meeting_name>_transcricao_oradores.txt
<meeting_name>_transcricao_oradores.docx
<meeting_name>_legendas_oradores.srt
<meeting_name>_amostras_oradores.txt
<meeting_name>_COM_ORADORES.zip
```

The most important file for the local Streamlit app is:

```text
<meeting_name>_segmentos_oradores.json
```

This file contains transcript segments with diarized speaker labels.

The file:

```text
<meeting_name>_amostras_oradores.txt
```

is useful for manually identifying each speaker.

---

## Local MeetingAgent app

After transcription and diarization in Colab, the local Streamlit app handles the human-in-the-loop part of the workflow.

### What the app does

The local app:

1. searches a local or synced folder for `*_segmentos_oradores.json` files;
2. loads the selected diarized meeting;
3. displays useful sample phrases for each detected speaker;
4. asks the user to map each speaker label to a real name;
5. generates a new transcript with real names;
6. optionally loads meeting documents;
7. calls Gemini to generate draft minutes;
8. exports the generated documents.

### Speaker assignment

The diarization step only produces generic labels:

```text
SPEAKER_00
SPEAKER_01
SPEAKER_02
...
```

The local app lets the user manually map those labels to real names:

```text
SPEAKER_00 = Person A
SPEAKER_01 = Person B
SPEAKER_02 = Person C
```

If the diarization model split the same person into two labels, the user can assign the same name to both labels.

Example:

```text
SPEAKER_00 = John Smith
SPEAKER_03 = John Smith
```

### Output after speaker assignment

```text
<meeting_name>_segmentos_com_nomes.json
<meeting_name>_transcricao_com_nomes.txt
<meeting_name>_transcricao_com_nomes.docx
<meeting_name>_mapa_oradores.json
```

The main file used for AI minutes generation is:

```text
<meeting_name>_segmentos_com_nomes.json
```

---

## Gemini-based draft minutes generation

The local app uses Gemini through the Google GenAI SDK.

The API key must be stored locally in a `.env` file:

```env
GEMINI_API_KEY=your_api_key_here
```

The current app supports:

- long transcript splitting into analysis blocks;
- intermediate block summaries;
- final draft minutes generation;
- task and pending-action extraction;
- retry logic;
- Gemini model fallback logic.

### Gemini outputs

```text
<meeting_name>_ata_preliminar_gemini.txt
<meeting_name>_ata_preliminar_gemini.docx
<meeting_name>_analise_intermedia_gemini.txt
<meeting_name>_tarefas_pendentes_gemini.txt
```

The draft minutes are currently generated in European Portuguese.

---

## Installation

Clone the repository:

```powershell
git clone https://github.com/Romuloyan/MeetingAgent---AI-Agent.git
cd MeetingAgent---AI-Agent
```

Create a virtual environment:

```powershell
python -m venv .venv
```

Install dependencies:

```powershell
& .\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Create a local `.env` file:

```env
GEMINI_API_KEY=your_api_key_here
```

Run the app:

```powershell
& .\.venv\Scripts\python.exe -m streamlit run app.py
```

---

## Expected local data folder

During development, the app was configured to look for meeting outputs in a local or synced folder such as:

```text
C:\Users\Pc\OneDrive\Agente IA\Transcricoes_atas
```

This can be changed in the Streamlit interface.

The repository does not include real meeting files.

---

## Security and privacy

This project processes potentially sensitive meeting content. The repository must not contain:

- `.env` files;
- API keys;
- audio recordings;
- real transcripts;
- real meeting minutes;
- personal files;
- generated meeting outputs;
- zipped exports from Colab;
- raw meeting documents.

The `.gitignore` file is configured to exclude common sensitive files and generated outputs.

---

## Why the workflow is human-in-the-loop

Speaker diarization can detect different voices, but it cannot reliably know real names. It can also split one real person into multiple speaker labels, or merge speakers in noisy sections.

For this reason, the app asks the user to confirm the speaker mapping before generating final documents.

The minutes generated by Gemini should also be reviewed before being used officially.

---

## Current limitations

- The Colab notebooks are currently stored as `.py` code files rather than `.ipynb` notebooks.
- The app currently expects JSON files produced by the existing Colab pipeline.
- The draft minutes are optimized for Portuguese meeting workflows.
- The current minutes-generation prompt still needs improvement to produce more formal institutional minutes by default.
- Real meeting files are intentionally excluded for privacy.

---

## Planned improvements

The next development steps are:

- convert the Colab `.py` scripts into clean `.ipynb` notebook examples;
- create a formal institutional minutes mode;
- separate formal minutes from internal validation reports;
- add a dedicated field for the official agenda;
- avoid converting additional topics into agenda items;
- keep decisions/deliberations inside each agenda point;
- generate a separate validation report for uncertain or missing information;
- add download buttons for generated files;
- improve Gemini model fallback and temporary service-limit handling;
- add example sanitized input/output files;
- improve project structure into modules instead of a single `app.py`.

---

## Development notes

This project was developed incrementally from a real workflow:

1. first, a Colab transcription notebook was stabilized;
2. then a separate Colab diarization notebook was created;
3. the diarization notebook was changed to a one-click subprocess-based design to avoid Colab dependency conflicts;
4. Colab/T4 was selected for the heavy audio steps because it was much faster than processing the full workflow on the laptop's integrated graphics;
5. local speaker-name assignment was implemented in Streamlit;
6. Gemini-based draft minutes generation was added;
7. model fallback and retry behavior were added after temporary API overload errors;
8. the GitHub repository was created with sensitive data excluded.

---

## License

No license has been selected yet.
