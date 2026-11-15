# Vision Redactor

A local privacy service that blurs faces and sensitive text in screenshots before they are sent to an AI model.

Built by Varun Kumar Kodali as the privacy layer of **Project Vision**, a browser-automation agent from our Smart India Hackathon 2026 entry (ISRO, problem statement 26171). The full team project is at [X-netrunner/Project-Vision](https://github.com/X-netrunner/Project-Vision).

## What it does

A browser extension captures a screenshot of the active tab and sends it to this service over a WebSocket. The service returns the same image with faces and personal data irreversibly blurred, so the vision model that plans the next click never sees them.

Two jobs run in parallel on every image:

- **Faces:** YuNet (OpenCV) finds faces and avatars, and a padded box is blurred around each.
- **Text:** RapidOCR reads every text block, and each block is checked three ways:
  - Presidio (spaCy) and regex look for emails, phone numbers, card and bank numbers, IP addresses and locations.
  - Inline fields such as `Name: value` blur the value after the label.
  - Form labels such as "Shipping Address" blur the stacked lines of text beneath them.

Everything flagged gets a double Gaussian blur, and the result is re-encoded as JPEG.

Because it works on pixels, it catches content that DOM-based masking cannot, such as faces, images, and PII sitting in plain page text.

## Running it

```bash
python main.py
```

On first start it installs the packages in `requirements.txt`, downloads the YuNet face model, loads the OCR and NLP models, and pre-warms them with a dummy image. Python 3.10 or newer is recommended.

The server listens on `ws://127.0.0.1:8000/ws`. Host, port, thresholds and blur strength are all in `config.py`.

## Message format

Request:

```json
{
  "type": "RAW_SCREENSHOT",
  "request_id": "req_1",
  "tab_id": 123,
  "step_index": 0,
  "image": "data:image/jpeg;base64,...",
  "action_result": null
}
```

Response:

```json
{
  "type": "REDACTED_SCREENSHOT",
  "request_id": "req_1",
  "tab_id": 123,
  "step_index": 0,
  "image": "data:image/jpeg;base64,...",
  "processing_time_ms": 0,
  "action_result": null
}
```

Errors come back as `{"type": "ERROR", "request_id": ..., "error": ...}`.

## Testing

`tester.py` is a small desktop app. Start `main.py`, run `python tester.py`, then pick a JPEG or capture the screen to see the redacted result.

## Design notes

- **Thread safety:** OCR and face-detector calls are serialised with locks, because both wrap shared model objects.
- **Privacy:** screenshot content is never written to the logs, only message sizes and timings.
- **Startup:** the models are pre-warmed so the first real request is not slow.

## Files

| File | Purpose |
|---|---|
| `main.py` | WebSocket server, startup checks, logging |
| `redact.py` | Pipeline: decode, detect, blur, encode |
| `face_detect.py` | YuNet face detection |
| `ocr_engine.py` | RapidOCR text extraction |
| `pii_engine.py` | Presidio and regex PII detection |
| `blur_utils.py` | Blur function |
| `config.py` | All settings |
| `tester.py` | Desktop test client |

## Status

Built for a hackathon and not production-hardened. The WebSocket has no authentication, so it is meant to run on localhost only.
