# SafeShot Desktop

SafeShot is an offline desktop application for applying a local
VAE protection prototype to an uploaded image. End users install a DMG, launch
SafeShot from an icon, and use a local browser interface. They do not need
Python, Conda, a terminal, an account, or internet access.

Developers building the DMG need internet once to install Python dependencies
and download the pinned model files from their original Hugging Face repository.
The completed DMG contains those files and runs offline.

## Easy Download

macOS: Download SafeShot.dmg, double-click to mount, drag SafeShot.app to Applications. First launch: right-click → Open (to bypass Gatekeeper on the unsigned build).
Also available via link: https://github.com/603-gl/SafeShot/releases/download/v1.0.0/SafeShot.dmg

The "right-click → Open" step is required because the DMG is unsigned (no Apple Developer ID). The build script already prints a warning about this — it's expected for a dev/research tool.

## Build a DMG from a Clean Clone

Requirements:

- An Apple Silicon Mac
- macOS 12 or newer
- Python 3.10
- Internet access during setup
- Several gigabytes of free disk space

```bash
git clone https://github.com/SabrinaYyy/SafeShot_dmg.git
cd SafeShot_dmg

python3.10 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -r requirements-build.txt

chmod +x scripts/download_model.sh scripts/build_macos.sh
./scripts/download_model.sh
./scripts/build_macos.sh
```

The finished installer is:

```text
dist/dmg/SafeShot.dmg
```

The first build can take several minutes. PyInstaller packages Python, Gradio,
PyTorch, the application, and the downloaded model into one offline app.

## Model Download

`scripts/download_model.sh` downloads only the VAE and scheduler files required
by SafeShot:

```text
models/instruct-pix2pix/
  vae/config.json
  vae/diffusion_pytorch_model.safetensors
  scheduler/scheduler_config.json
```

The download is pinned to:

```text
Repository: timbrooks/instruct-pix2pix
Revision:   31519b5cb02a7fd89b906d88731cd4d6a7bbf88d
```

Pinning the revision ensures that every developer builds with the same model
files. Downloaded model files are ignored by Git and should not be committed.

The application itself uses `local_files_only=True`. Running `python app.py`
does not download a missing model. Run `./scripts/download_model.sh` first.

## Run from Source

After completing the environment and model-download steps:

```bash
python scripts/validate_bundle.py
python -m pytest
python app.py
```

The interface opens locally at `127.0.0.1`. Gradio sharing is disabled.

The main screen includes:

- Resolution choices of 128, 256, or 512 pixels
- An optimization-step slider from 20 to 100
- A Stop button that requests cancellation between optimization steps

The default is 256 pixels and 20 steps. Higher settings can take dramatically
longer. Stopping is cooperative, so the current model operation must finish
before cancellation takes effect.

## Repository Structure

```text
SafeShot/
  app.py
  SafeShot.spec
  requirements.txt
  requirements-build.txt
  SafeShot/
    protection.py
    resources.py
    ui.py
  models/
    .gitkeep
  packaging/
    macos/
    windows/
  scripts/
    download_model.sh
    build_macos.sh
    build_windows.ps1
    validate_bundle.py
  tests/
  licenses/
```

## macOS Signing

Without an Apple signing identity, `build_macos.sh` creates an unsigned
development DMG. Another Apple Silicon Mac may require the user to right-click
SafeShot and choose **Open**.

For a signed build:

```bash
export APPLE_SIGNING_IDENTITY="Developer ID Application: Organization (TEAMID)"
./scripts/build_macos.sh
```

A public release should also be notarized and stapled using an Apple Developer
account.

## Windows Build

Windows releases must be built on Windows:

```powershell
py -3.10 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements-build.txt
```

Download the same pinned model files into `models\instruct-pix2pix`, then run:

```powershell
.\scripts\build_windows.ps1
```

The PyInstaller output is `dist\SafeShot\SafeShot.exe`. Compile
`packaging\windows\SafeShot.iss` with Inno Setup to create the installer.

## Privacy and Storage

- Processing occurs on the user's computer.
- The server binds only to `127.0.0.1`.
- Public Gradio sharing is disabled.
- Generated downloads are stored in the user's application-data directory.
- Generated files older than 24 hours are removed during a later startup.

Closing the browser tab does not necessarily stop SafeShot. Quit the
application through the Dock or operating-system application controls.

## Release Checklist

1. Confirm redistribution rights for the model checkpoint and dependencies.
2. Include authoritative third-party licenses and required attributions.
3. Run `./scripts/download_model.sh`.
4. Run `python scripts/validate_bundle.py` and `python -m pytest`.
5. Build on a clean machine for each operating system and architecture.
6. Test startup, protection, cancellation, and download with networking off.
7. Sign and notarize macOS public releases.
8. Sign Windows executables and installers.
9. Publish a SHA-256 checksum for each installer.

## Product Limits

- This is the VAE latent-divergence protection path, not universal protection
  against every image-editing or face-swap system.
- Output is a center-cropped square PNG at the selected resolution.
- CPU mode is supported but can be much slower than MPS or CUDA.
- One protection job runs at a time to limit accelerator memory use.
- The installed application is large because it contains PyTorch, Gradio, and
  the model checkpoint.

## Acknowledgements and Citations
SafeShot is built from existent prototypes of EditShield and BlurGuard. The code of the model is taken from their respective repositories, and merged into SafeShot using Claude Code. Moreover, we focused on protecting against specifics attacks of IP2P and SD (Stable Diffusion).

Core references:
- Baseline MIT Prototype: Hadi Salman, Alaa Khaddaj, Guillaume Leclerc, Andrew Ilyas, and Aleksander Madry. "Raising the Cost of Malicious AI-Powered Image Editing" (PhotoGuard). 2023. https://arxiv.org/abs/2302.06588
- EditShield Paper: Ruoxi Chen, Haibo Jin, Yixin Liu, Jinyin Chen, Haohan Wang, and Lichao Sun. "EditShield: Protecting Unauthorized Image Editing by Instruction-guided Diffusion Models." ECCV 2024. https://arxiv.org/abs/2311.12066
- BlurGuard Paper: Jinsu Kim, Yunhun Nam, Minseon Kim, Sangpil Kim, and Jongheon Jeong. "BlurGuard: A Simple Approach for Robustifying Image Protection Against AI-Powered Editing." 2025. https://arxiv.org/abs/2511.00143
- IP2P: Tim Brooks, Aleksander Holynski, and Alexei A. Efros. "InstructPix2Pix: Learning to Follow Image Editing Instructions." 2022. https://arxiv.org/abs/2211.09800
- Stable Diffusion Model: https://huggingface.co/stable-diffusion-v1-5/stable-diffusion-inpainting
- Robin Rombach, Andreas Blattmann, Dominik Lorenz, Patrick Esser, and Bjorn Ommer. "High-Resolution Image Synthesis with Latent Diffusion Models." 2021. https://arxiv.org/abs/2112.10752
