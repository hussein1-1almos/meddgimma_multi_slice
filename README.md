# MedGemma multi-slice DICOM case review (experimental)

This package extends the single-image prototype with a folder-upload interface and a `POST /v1/localize-series` endpoint. It is intended for software testing/research only. **It is not a medical device and must not be used for diagnosis, triage, or treatment decisions. All outputs require qualified clinical review.**

## Important behavior and limits

- Select a folder containing DICOM files in the Streamlit interface.
- The API orders files by `SeriesInstanceUID`, projected `ImagePositionPatient` when orientation/position are available, then `InstanceNumber` and filename.
- Each slice is analyzed independently by MedGemma as a 2D image. It does **not** perform native 3D volume reasoning, lesion tracking across slices, or a validated whole-study interpretation.
- Output boxes use `original_image_pixels` on the decoded 2D image for each slice.
- Up to 300 files per request; max 64 MiB per file by default and 512 MiB total per series request. Larger studies should be split into batches.
- The UI's progress indicator cannot show actual per-slice inference progress while the single HTTP request is running. Large CPU-only studies may take a long time; set a sufficiently long timeout.
- Folder upload support depends on a recent desktop Streamlit/browser. If directory selection is unavailable, ZIP upload is not implemented in this version; use a supported browser or ask to add ZIP support.
- DICOM privacy: this app uploads the original DICOM files, including metadata, to the configured server. Use only authorized data and an appropriately secured network; do not expose the API publicly without authentication, TLS, access controls, and privacy review.

## Server setup (Ubuntu)

Use the Python version compatible with the installed PyTorch/Transformers versions on your machine. Run these commands from the `server` directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env
```

If the model is gated on Hugging Face, accept its terms and authenticate on the server before startup. Configure a local model path in `.env` if the model has already been downloaded. Then run:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 1
```

Check `http://SERVER_IP:8000/health` and API documentation at `http://SERVER_IP:8000/docs`. Ensure the firewall allows port 8000 only from trusted clients. The model is CPU-only in this implementation; RAM and processing time can be substantial.

## Client setup (Windows)

Run from the `client` directory in PowerShell:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
streamlit run app.py
```

Open the local Streamlit URL (usually `http://localhost:8501`). In the sidebar, set the Ubuntu server URL, for example `http://192.168.1.20:8000`. Select a folder containing DICOM slices, then choose **Upload case and analyze slices**.

## API

`POST /v1/localize-series` uses multipart field `files` (repeat once per slice) and form field `target`. The response includes a sorted `slices` array, one object per input file, with `slice_number`, `filename`, `series_instance_uid`, `instance_number`, image dimensions, `findings`, status, and optional error. Boxes are per-slice pixel coordinates.

The existing `POST /v1/localize` endpoint remains available for one image.

## Validate before use

Test only on non-identifiable sample data first. Verify ordering against the source viewer, compare pixel dimensions and orientation, test compressed transfer syntaxes, measure false positives and false negatives on an appropriately labeled dataset, and obtain radiologist review. An empty findings list is not evidence that a study is normal.
