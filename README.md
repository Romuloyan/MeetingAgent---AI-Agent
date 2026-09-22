# MeetingAgent — AI Agent

MeetingAgent is a local AI-assisted workflow for turning meeting recordings into speaker-labelled transcripts, structured meeting notes, draft minutes, and action-item reports.

The project was built as a practical MVP for Portuguese meeting workflows, especially meetings where the final minutes need to be written in formal European Portuguese. The repository documentation is written in English for portfolio and project presentation purposes, while the generated meeting-minutes output is currently optimized for Portuguese.

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

---

## Google Colab code developed for this project

The project includes two Colab notebooks as part of the working pipeline, even though the `.ipynb` notebook files have not yet been committed to this repository.

The notebooks were not theoretical placeholders. They were built, debugged, and tested as working code blocks during development.

### Colab Notebook 1 — Transcription code

The first Colab notebook performs automatic transcription of audio or video files stored in Google Drive.

Its main responsibilities are:

- mount Google Drive;
- scan the meeting folder for audio/video files;
- skip recordings that were already processed;
- run `faster-whisper` using GPU when available;
- export several transcript formats;
- create a complete ZIP package per recording.

### Core configuration

```python
PASTA_BASE = Path("/content/drive/MyDrive/Transcrições e atas")
PASTA_RESULTADOS = PASTA_BASE / "_resultados_transcricao"

MODELO = "large-v3"
IDIOMA = "pt"

EXTENSOES_AUDIO = {
    ".m4a", ".mp3", ".wav", ".aac", ".flac",
    ".mp4", ".mov", ".mkv", ".webm", ".ogg"
}
```

### GPU/CPU selection logic

The transcription notebook automatically checks whether CUDA is available and selects the best execution mode:

```python
if torch.cuda.is_available():
    DEVICE = "cuda"
    COMPUTE_TYPE = "float16"
else:
    DEVICE = "cpu"
    COMPUTE_TYPE = "int8"
```

This allows the same notebook to run on a Colab T4 GPU or, more slowly, on CPU.

### Whisper transcription call

The working transcription code uses `faster-whisper` with VAD filtering and Portuguese language selection:

```python
segments, info = modelo.transcribe(
    str(audio_path),
    language=IDIOMA,
    beam_size=5,
    vad_filter=True,
    vad_parameters={"min_silence_duration_ms": 500},
    condition_on_previous_text=True,
)
```

### Transcription outputs

For each recording, the notebook exports:

```text
<meeting_name>_transcricao_timestamp.txt
<meeting_name>_transcricao_corrida.txt
<meeting_name>_legendas.srt
<meeting_name>_segmentos.json
<meeting_name>_transcricao.docx
<meeting_name>_TRANSCRICAO_COMPLETA.zip
```

The key output for the next stage is:

```text
<meeting_name>_segmentos.json
```

That file stores the transcript as timestamped JSON segments and is used by the diarization notebook.

---

## Colab Notebook 2 — Diarization code

The second Colab notebook adds speaker diarization to the transcript generated by Notebook 1.

Its main responsibilities are:

- mount Google Drive;
- locate `*_segmentos.json` files;
- locate the original audio file;
- convert the audio to clean WAV format;
- run `pyannote.audio` diarization;
- map speaker intervals back to transcript segments;
- export speaker-labelled transcript files.

### Why the diarization notebook uses an external runner

A major implementation issue appeared during development: installing `pyannote.audio` inside Colab can change package versions used by the active Python kernel, especially around NumPy, Torch, Numba, and pyannote dependencies.

The final working design avoids importing `pyannote` directly inside the notebook kernel after installation.

Instead, the notebook:

1. installs the required packages;
2. writes a separate Python file, for example `/content/diarizacao_runner_one_click.py`;
3. launches that file in a new Python subprocess;
4. imports and runs `pyannote.audio` only inside the subprocess.

This made the diarization notebook usable as a one-click Colab workflow.

### Simplified notebook-side runner launch

```python
runner = Path("/content/diarizacao_runner_one_click.py")

runner.write_text(r'''
# external diarization runner code goes here
''', encoding="utf-8")

processo = subprocess.Popen(
    [sys.executable, str(runner)],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True,
    bufsize=1,
    env=os.environ.copy(),
)

for linha in processo.stdout:
    print(linha, end="", flush=True)

codigo = processo.wait()

if codigo != 0:
    raise RuntimeError(f"Diarization failed with code {codigo}")
```

### Hugging Face token handling

The diarization notebook uses a Hugging Face token stored in Colab Secrets.

During development, the secret name was:

```text
TOKEN
```

The token is read inside Colab and passed to the subprocess through an environment variable:

```python
TOKEN_HF = userdata.get("TOKEN")
os.environ["TOKEN_HF"] = TOKEN_HF.strip()
```

The token itself is not stored in the repository.

### Diarization model

The pyannote model used during development was:

```text
pyannote/speaker-diarization-community-1
```

The runner loads it with:

```python
pipeline = Pipeline.from_pretrained(
    "pyannote/speaker-diarization-community-1",
    token=TOKEN_HF,
)

pipeline.to(device)
```

### Audio conversion before diarization

The final diarization code converts audio files to WAV before sending them to pyannote:

```python
cmd = [
    "ffmpeg",
    "-y",
    "-i", str(audio_path),
    "-vn",
    "-ac", "1",
    "-ar", "16000",
    "-sample_fmt", "s16",
    "-fflags", "+genpts",
    "-avoid_negative_ts", "make_zero",
    str(wav_path),
]
```

This step was added to avoid sample mismatch errors when using compressed audio formats such as `.m4a`.

### Pyannote output handling

Different versions of `pyannote.audio` may return different wrapper objects. The working code handles the returned diarization object safely:

```python
if hasattr(resultado, "exclusive_speaker_diarization"):
    anotacao = resultado.exclusive_speaker_diarization
elif hasattr(resultado, "speaker_diarization"):
    anotacao = resultado.speaker_diarization
else:
    anotacao = resultado

for turno, _, speaker in anotacao.itertracks(yield_label=True):
    diarizacao.append({
        "inicio": float(turno.start),
        "fim": float(turno.end),
        "orador": str(speaker),
    })
```

### Mapping diarization to transcript segments

The diarization runner assigns each transcript segment to the speaker with the highest time overlap:

```python
def sobreposicao(a_inicio, a_fim, b_inicio, b_fim):
    return max(0.0, min(a_fim, b_fim) - max(a_inicio, b_inicio))


def atribuir_orador(seg, diarizacao):
    pontos = defaultdict(float)

    for d in diarizacao:
        ov = sobreposicao(seg["inicio"], seg["fim"], d["inicio"], d["fim"])
        if ov > 0:
            pontos[d["orador"]] += ov

    if not pontos:
        return "ORADOR_DESCONHECIDO"

    melhor, valor = max(pontos.items(), key=lambda x: x[1])

    if valor < 0.05:
        return "ORADOR_DESCONHECIDO"

    return melhor
```

### Diarization outputs

For each meeting, the diarization notebook exports:

```text
<meeting_name>_segmentos_oradores.json
<meeting_name>_transcricao_oradores.txt
<meeting_name>_transcricao_oradores.docx
<meeting_name>_legendas_oradores.srt
<meeting_name>_amostras_oradores.txt
<meeting_name>_COM_ORADORES.zip
```

The key file for the local Streamlit application is:

```text
<meeting_name>_segmentos_oradores.json
```

The file:

```text
<meeting_name>_amostras_oradores.txt
```

contains representative phrases for each detected speaker and helps the user identify who each speaker is.

### Tested diarization result during development

A tested meeting recording produced:

```text
3559 transcript segments
3417 detected speech intervals
5 detected speaker labels + ORADOR_DESCONHECIDO
```

The diarization stage successfully exported all expected JSON, TXT, DOCX, SRT, sample, and ZIP files.

---

## Local MeetingAgent app

After transcription and diarization in Colab, the local Streamlit app handles the human-in-the-loop part of the workflow.

### What the app does

The local app:

1. searches a local/synced folder for `*_segmentos_oradores.json` files;
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

After speaker identification, the app creates:

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

### Gemini outputs

After generating meeting minutes, the app exports:

```text
<meeting_name>_ata_preliminar_gemini.txt
<meeting_name>_ata_preliminar_gemini.docx
<meeting_name>_analise_intermedia_gemini.txt
<meeting_name>_tarefas_pendentes_gemini.txt
```

The minutes are currently generated in European Portuguese.

## Project structure

```text
MeetingAgent---AI-Agent/
├── app.py
├── requirements.txt
├── README.md
└── .gitignore
```

### Main files

- `app.py` — main Streamlit application.
- `requirements.txt` — Python dependencies for the local app.
- `README.md` — project documentation.
- `.gitignore` — excludes virtual environments, secrets, local transcripts, meeting files, generated outputs, and media files.

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

- The Colab notebooks are part of the broader workflow but are not yet included as `.ipynb` files in this repository.
- The app currently expects JSON files produced by the existing Colab pipeline.
- The draft minutes are optimized for Portuguese meeting workflows.
- The current minutes-generation prompt still needs improvement to produce more formal institutional minutes by default.
- Real meeting files are intentionally excluded for privacy.

## Planned improvements

The next development steps are:

- add the two Colab notebooks to the repository as documented examples;
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

## Development notes

This project was developed incrementally from a real workflow:

1. first, a Colab transcription notebook was stabilized;
2. then a separate Colab diarization notebook was created;
3. the diarization notebook was changed to a one-click subprocess-based design to avoid Colab dependency conflicts;
4. local speaker-name assignment was implemented in Streamlit;
5. Gemini-based draft minutes generation was added;
6. model fallback and retry behavior were added after temporary API overload errors;
7. the GitHub repository was created with sensitive data excluded.

## License

No license has been selected yet.
