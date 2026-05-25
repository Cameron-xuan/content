---
title: 'Run WhisperX Transcription With Sapat'
description:
  'Build a reproducible Daytona workflow for local WhisperX transcription,
  alignment, and optional diarization with Sapat.'
date: 2026-05-25
author: 'Cameron Xuan'
tags: ['daytona', 'sapat', 'whisperx', 'transcription', 'ai']
---

# Run WhisperX Transcription With Sapat

# Introduction

AI engineers often need more than a plain transcript. A meeting recording,
lecture, product demo, or research interview becomes much easier to review when
the transcript stays connected to the source audio. That is where
[word-level speech alignment](/definitions/20260525_definition_word_level_speech_alignment.md)
helps: it lets the workflow preserve timing data so humans can audit confusing
phrases, build subtitle files, or jump from a quote back to the original media.

[Sapat](https://github.com/nibzard/sapat) is a compact Python command-line tool
that converts video files with `ffmpeg`, sends the audio to a selected
transcription backend, and writes a `.txt` transcript next to each source file.
The companion Sapat pull request adds a local `--api whisperx` backend, so the
same workflow can run through [WhisperX](https://github.com/m-bain/whisperX)
inside a reproducible [Daytona](https://github.com/daytonaio/daytona)
workspace.

## TL;DR

- Use Daytona to create a clean Python workspace for Sapat.
- Configure Sapat with the new `--api whisperx` provider.
- Keep WhisperX settings in local `WHISPERX_*` environment variables.
- Run local transcription for a file or directory without committing secrets,
  model files, or private media.
- Enable diarization only when you have a Hugging Face token and have accepted
  the required model terms.

## Why Use WhisperX With Sapat?

OpenAI, Groq, and Azure OpenAI are convenient when you want hosted speech
recognition. WhisperX is a better fit when you want local execution, tighter
control over model/device settings, alignment-focused output, and an optional
speaker diarization pass for review workflows.

That makes the WhisperX-backed Sapat flow useful for:

- AI teams reviewing recorded demos before turning them into issues.
- Developer relations teams creating drafts for tutorials or release videos.
- Researchers who need a local-first workflow for sensitive recordings.
- Product teams that want transcript text and source-audio review in one loop.

The key idea is simple: Daytona gives the project a repeatable workspace, Sapat
keeps the video-to-transcript command small, and WhisperX handles the
transcription and alignment work.

![WhisperX transcription workflow in Daytona](/guides/assets/20260525_run_whisperx_transcription_with_sapat_in_daytona.svg)

## Step 1: Create the Daytona Workspace

Start from a machine with the Daytona CLI installed and authenticated. Create a
workspace from the Sapat repository:

```bash
daytona create https://github.com/nibzard/sapat --code
```

Open a terminal inside the workspace and confirm Python and `ffmpeg` are
available:

```bash
python --version
ffmpeg -version
```

Sapat already uses `ffmpeg` to convert input video files to MP3 before sending
the audio to a provider. If your base workspace does not include `ffmpeg`, add
it to the workspace image or install it through the package manager used by
your Daytona environment.

## Step 2: Prepare Python Dependencies

Create an isolated Python environment for Sapat:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install .
```

Install WhisperX in the same environment or expose it through another command
runner such as `uvx`:

```bash
python -m pip install whisperx
whisperx --help
```

WhisperX is a heavier local dependency than a hosted API client because it uses
speech models and PyTorch. If you are running on CPU, start with a smaller model
and `int8` compute. If you have GPU access, move to a larger model once the
workflow is validated.

## Step 3: Configure the WhisperX Provider

Create a local `.env` file in the Sapat workspace. Do not commit this file.

```bash
WHISPERX_BINARY=whisperx
WHISPERX_MODEL=small
WHISPERX_DEVICE=cpu
WHISPERX_COMPUTE_TYPE=int8
WHISPERX_BATCH_SIZE=4
WHISPERX_ALIGN_MODEL=
WHISPERX_HF_TOKEN=
WHISPERX_DIARIZE=false
WHISPERX_MIN_SPEAKERS=
WHISPERX_MAX_SPEAKERS=
WHISPERX_EXTRA_ARGS=
```

These values map to the WhisperX command that Sapat will run:

- `WHISPERX_BINARY` can be `whisperx`, an absolute path, or a wrapper such as
  `uvx whisperx`.
- `WHISPERX_MODEL` selects the Whisper model name, such as `small` or
  `large-v2`.
- `WHISPERX_DEVICE` selects `cpu` or `cuda`.
- `WHISPERX_COMPUTE_TYPE` controls the inference type, with `int8` being a
  practical CPU default.
- `WHISPERX_BATCH_SIZE` controls inference batching.
- `WHISPERX_ALIGN_MODEL` lets you override the automatic alignment model.
- `WHISPERX_EXTRA_ARGS` gives advanced users a controlled escape hatch for
  additional WhisperX CLI flags.

## Step 4: Run a Single Transcription

Put a short test video in the workspace, then run Sapat with the WhisperX
provider:

```bash
sapat ./recordings/demo.mp4 --api whisperx --language en --quality M
```

Sapat will:

- Convert `demo.mp4` to a temporary `demo.mp3`.
- Run the configured WhisperX command with `--output_format txt`.
- Read the generated WhisperX text output.
- Write `demo.txt` next to the source video.
- Remove the temporary MP3 file.

Use the `--prompt` option when the recording contains product names, acronyms,
or domain-specific vocabulary:

```bash
sapat ./recordings/demo.mp4 \
  --api whisperx \
  --language en \
  --prompt "Daytona, Sapat, WhisperX, dev containers"
```

The provider passes that value to WhisperX as `--initial_prompt`, which helps
the first decoding window recognize important terms.

## Step 5: Process a Directory

Sapat also accepts a directory and processes each `.mp4` file inside it:

```bash
sapat ./recordings --api whisperx --language en --quality M
```

Use this mode for repeatable review batches. For example, an AI engineering
team can drop several demo recordings into `recordings/`, run one command, and
then review the generated `.txt` files before turning the findings into issues,
release notes, or support articles.

## Step 6: Enable Optional Diarization

Speaker diarization can be valuable when the transcript needs to distinguish
between participants. WhisperX supports diarization through pyannote models on
Hugging Face. Before enabling it, create a read token, accept the required model
terms, and keep the token only in your local environment.

```bash
WHISPERX_HF_TOKEN=hf_your_read_token
WHISPERX_DIARIZE=true
WHISPERX_MIN_SPEAKERS=2
WHISPERX_MAX_SPEAKERS=4
```

Then run the same Sapat command:

```bash
sapat ./recordings/customer-call.mp4 --api whisperx --language en
```

For private or regulated recordings, validate your data-handling policy before
using diarization models that require external model downloads or gated model
access.

## Step 7: Review the Output

After transcription, treat the `.txt` file as a draft. A practical QA pass
usually checks:

- Product names and technical acronyms.
- Numbers, currencies, dates, and version strings.
- Names of speakers, customers, repositories, and pull requests.
- Sections where overlapping speech may have reduced accuracy.
- Whether the transcript should be edited before being shared outside the team.

Keep private media, generated MP3 files, `.env` values, and downloaded model
weights out of Git. Commit only the code, documentation, or transcript excerpts
that are safe to publish.

## Common Issues and Troubleshooting

**Problem:** `WhisperX executable was not found.`

**Solution:** Install WhisperX in the active environment or set
`WHISPERX_BINARY` to the command you want Sapat to run. For example:

```bash
WHISPERX_BINARY=whisperx
```

or:

```bash
WHISPERX_BINARY="uvx whisperx"
```

**Problem:** CPU transcription is slow.

**Solution:** Start with `WHISPERX_MODEL=small`, `WHISPERX_COMPUTE_TYPE=int8`,
and a smaller `WHISPERX_BATCH_SIZE`. Move to `cuda` and a larger model only
after the workflow works on a short sample.

**Problem:** Diarization fails while loading the model.

**Solution:** Check that `WHISPERX_HF_TOKEN` is present, that the token has read
access, and that the required Hugging Face model terms have been accepted by
the account associated with the token.

**Problem:** The transcript misses important project-specific terms.

**Solution:** Pass a focused `--prompt` with product names, acronyms, and
repository names. Keep it short; the prompt should guide recognition, not become
a replacement transcript.

## Conclusion

The WhisperX provider turns Sapat into a local-first transcription workflow for
AI engineers who care about reproducibility, timing-aware review, and optional
speaker separation. Daytona keeps the environment isolated, Sapat keeps the
command surface small, and WhisperX supplies the alignment-oriented speech
pipeline.

Use this pattern when you need a repeatable transcript workflow for demos,
meetings, lectures, and research recordings, especially when you want to keep
provider configuration and secrets outside the repository.

## References

- [Sapat repository](https://github.com/nibzard/sapat)
- [WhisperX repository and CLI examples](https://github.com/m-bain/whisperX)
- [Daytona repository](https://github.com/daytonaio/daytona)
- [Companion Sapat PR](https://github.com/nibzard/sapat/pull/50)
