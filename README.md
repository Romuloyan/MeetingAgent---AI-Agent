# MeetingAgent — AI Agent

MeetingAgent is a local AI-assisted workflow for turning meeting recordings into speaker-labelled transcripts, structured meeting notes, draft meeting minutes, and action-item reports.

The project was built as a practical MVP for Portuguese association/club meeting workflows, especially meetings where the final minutes need to be written in formal European Portuguese. The repository documentation is written in English for portfolio and project presentation purposes, while the generated meeting-minutes output is currently optimized for Portuguese.

## Repository contents

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

The repository now includes both pieces of Colab code used in the project workflow, stored as plain `.py` Colab-compatible scripts:

- [`colab/01_transcription_faster_whisper_colab.py`](colab/01_transcription_faster_whisper_colab.py)
- [`colab/02_diarization_pyannote_one_click_colab.py`](colab/02_diarization_pyannote_one_click_colab.py)

They are not stored as `.ipynb` yet, but the actual Colab logic is included and can be pasted into Colab cells or converted into notebooks later.

## What the project does

MeetingAgent connects three stages of a meeting-documentation pipeline:

1. **Google Colab transcription notebook**  
   Converts audio/video recordings into timestamped transcripts using `faster-whisper`.

2. **Google Colab diarization notebook**  
   Uses `pyannote.audio` to detect who spoke when and exports speaker-labelled transcript files with labels such as `SPEAKER_00`, `SPEAKER_01`, etc.

3. **Local Streamlit MeetingAgent**  
   Runs locally on the user's computer, lets the user assign real names to detected speakers, loads optional meeting context documents, and uses Gemini to generate draft meeting minutes and task reports.

The goal is not to fully automate official meeting minutes without human review. The goal is to reduce the manual workload by producing a structured, reviewable draft based on the audio transcript, speaker diarization, and optional meeting documents.

## Current status

Functional MVP.

The current version can:

- read diarized transcript files generated in Google Colab;
- list all detected speakers;
- show useful sample phrases for each speaker;
- let the user assign real names to `SPEAKER_00`, `SPEAKER_01`, etc.;
- generate a transcript with real speaker names;
- upload optional meeting documents such as agendas, notes, attachments, or preliminary drafts;
- split long transcripts into manageable analysis blocks;
- call Gemini with retry and fallback logic;
- generate draft meeting minutes;
- generate intermediate analysis files;
- generate a task and pending-action report;
- export `.txt` and `.docx` files.

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

## Colab code included in this repository

### 1. Transcription code

File:

```text
colab/01_transcription_faster_whisper_colab.py
```

This is the Colab-side transcription script. It is designed to run in Google Colab and process audio/video files stored in Google Drive.

Main responsibilities:

- install `faster-whisper`, `python-docx`, and `ffmpeg`;
- mount Google Drive;
- scan a Drive folder for supported media files;
- detect whether CUDA/GPU is available;
- load the Whisper model;
- transcribe the file into timestamped segments;
- export several formats needed by the following stages.

Model configuration used during development:

```python
MODEL_NAME = "large-v3"
LANGUAGE = "pt"
```

The script automatically chooses the processing mode:

```text
GPU/CUDA available → device="cuda", compute_type="float16"
CPU only           → device="cpu", compute_type="int8"
```

Supported input formats:

```text
.m4a, .mp3, .wav, .aac, .flac, .mp4, .mov, .mkv, .webm, .ogg
```

Default Drive folder used during development:

```text
/content/drive/MyDrive/Transcrições e atas
```

Generated transcription outputs:

```text
<meeting_name>_transcricao_timestamp.txt
<meeting_name>_transcricao_corrida.txt
<meeting_name>_legendas.srt
<meeting_name>_segmentos.json
<meeting_name>_transcricao.docx
<meeting_name>_TRANSCRICAO_COMPLETA.zip
```

The most important output for the next stage is:

```text
<meeting_name>_segmentos.json
```

This JSON file contains the transcript split into timestamped segments.

### 2. Diarization code

File:

```text
colab/02_diarization_pyannote_one_click_colab.py
```

This is the Colab-side speaker diarization script. It is designed to run after the transcription script has created `*_segmentos.json` files.

Main responsibilities:

- mount Google Drive;
- read `*_segmentos.json` files generated by the transcription stage;
- locate the matching original audio file;
- convert the audio to clean WAV format;
- run `pyannote.audio` speaker diarization;
- align detected speaker intervals with Whisper transcript segments;
- export speaker-labelled transcripts and speaker-identification samples.

Diarization model used during development:

```text
pyannote/speaker-diarization-community-1
```

Required Colab Secret:

```text
TOKEN
```

`TOKEN` must contain a Hugging Face token with access to the pyannote model. The token is not stored in this repository.

#### One-click subprocess design

The diarization code uses a specific Colab-safe design.

The notebook installs `pyannote.audio`, but it does **not** import `pyannote` directly in the active Colab kernel after installation. Instead, it:

1. installs the dependencies;
2. writes an external Python runner script to `/content/diarization_runner_one_click.py`;
3. starts a fresh Python subprocess;
4. imports `pyannote.audio` inside that subprocess;
5. runs the diarization pipeline;
6. exports the results back to Google Drive.

This design was added because direct imports in the same Colab kernel caused dependency conflicts involving NumPy, Torch, Numba, and pyannote after package installation.

#### Audio preprocessing

Before diarization, the script converts the original audio file to:

```text
16 kHz
mono
PCM signed 16-bit WAV
```

This avoids sample mismatch errors with compressed formats such as `.m4a`.

Generated diarization outputs:

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

The file used for manual speaker identification is:

```text
<meeting_name>_amostras_oradores.txt
```

## Local MeetingAgent app

File:

```text
app.py
```

After transcription and diarization in Colab, the local Streamlit app handles the human-in-the-loop part of the workflow.

The local app:

1. searches a local/synced folder for `*_segmentos_oradores.json` files;
2. loads the selected diarized meeting;
3. displays useful sample phrases for each detected speaker;
4. asks the user to map each speaker label to a real name;
5. generates a new transcript with real names;
6. optionally loads meeting documents;
7. calls Gemini to generate draft minutes;
8. exports the generated documents.

## Speaker assignment

The diarization stage only produces generic labels:

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

After speaker identification, the app creates:

```text
<meeting_name>_segmentos_com_nomes.json
<meeting_name>_transcricao_com_nomes.txt
<meeting_name>_transcricao_com_nomes.docx
<meeting_name>_mapa_oradores.json
```

## Gemini-based draft minutes generation

The app uses Gemini through the Google GenAI SDK.

The API key must be stored locally in a `.env` file:

```env
GEMINI_API_KEY=your_api_key_here
```

The current app supports:

- long transcript splitting into analysis blocks;
- intermediate block summaries;
- final draft minutes generation;
- task and pending-action extraction;
- basic retry logic;
- Gemini model fallback logic.

After generating draft minutes, the app exports:

```text
<meeting_name>_ata_preliminar_gemini.txt
<meeting_name>_ata_preliminar_gemini.docx
<meeting_name>_analise_intermedia_gemini.txt
<meeting_name>_tarefas_pendentes_gemini.txt
```

The minutes are currently generated in European Portuguese.

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

## Expected local data folder

During development, the app was configured to look for meeting outputs in a local/synced folder such as:

```text
C:\Users\Pc\OneDrive\Agente IA\Transcricoes_atas
```

This can be changed in the Streamlit interface.

The repository does not include real meeting files.

## Security and privacy

This project processes potentially sensitive meeting content. The repository must not contain:

- `.env` files;
- API keys;
- Hugging Face tokens;
- audio recordings;
- real transcripts;
- real meeting minutes;
- personal files;
- generated meeting outputs;
- zipped exports from Colab;
- raw meeting documents.

The `.gitignore` file is configured to exclude common sensitive files and generated outputs.

## Why the workflow is human-in-the-loop

The system deliberately keeps a human validation step between diarization and minutes generation.

Speaker diarization can detect different voices, but it cannot reliably know real names. It can also split one real person into multiple speaker labels, or merge speakers in noisy sections. For this reason, the app asks the user to confirm the speaker mapping before generating final documents.

The minutes generated by Gemini should also be reviewed before being used officially.

## Current limitations

- The Colab code is included as `.py` scripts, not yet as `.ipynb` notebooks.
- The app currently expects JSON files produced by the included Colab pipeline.
- The draft minutes are optimized for Portuguese meeting workflows.
- The current minutes-generation prompt still needs improvement to produce formal institutional minutes by default.
- Real meeting files are intentionally excluded for privacy.

## Planned improvements

The next development steps are:

- convert the Colab `.py` scripts into full `.ipynb` notebooks;
- create a formal institutional minutes mode;
- separate formal minutes from internal validation reports;
- add a dedicated field for the official agenda;
- avoid converting additional topics into agenda items;
- keep decisions/deliberations inside each agenda point;
- generate a separate validation report for uncertain or missing information;
- add download buttons for generated files;
- improve Gemini model fallback and temporary service-limit handling;
- add sanitized sample input/output files;
- improve project structure into modules instead of a single `app.py`.

## Development notes

This project was developed incrementally from a real workflow:

1. first, a Colab transcription notebook was stabilized;
2. then a separate Colab diarization notebook was created;
3. the diarization notebook was changed to a one-click subprocess-based design to avoid Colab dependency conflicts;
4. local speaker-name assignment was implemented in Streamlit;
5. Gemini-based draft minutes generation was added;
6. model fallback and retry behavior were added after temporary API overload errors;
7. the GitHub repository was created with sensitive data excluded;
8. the actual Colab code was added to the repository under `colab/`.

## License

No license has been selected yet.
