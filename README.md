# Twilio Outbound Dialer

A FastAPI-based automated outbound calling system for a system in Australia to contact pet owners about failed payments.  
The system dials contacts from an uploaded CSV, plays a personalized greeting + pre-recorded message using ElevenLabs TTS, records callers who ask for a payment link by SMS (press 1) into a dashboard work list for manual sending, and transfers callers who want to speak with the team (press 2) to a human agent.

## Features

- Upload daily contact list via CSV (columns: `Client`, `Name`, `Phone`)
- Sequential outbound calling with rate limiting (one call at a time)
- Personalized "Hi [Name]" greeting generated via ElevenLabs TTS
- Failed-payment script played as pre-generated MP3
- Speech & DTMF input detection:
  - Press 1 (or say "text me" / "SMS" / "send the link") → caller is added to the **SMS Requests panel** (nothing is sent automatically — the team texts the payment link manually)
  - Press 2 (or say "transfer" / "agent" / "speak to someone") → transfer to human agent
- SMS Requests panel on the dashboard (`sms_requests.json`, served by `/sms-requests`) listing Client, Name, Phone and request time; cleared on each new CSV upload
- Call outcome tracking: `no_answer`, `busy`, `voicemail`, `sms_requested`, `completed_no_transfer`, `successfully_transferred`
- Automatic CSV result generation with `Response` (outcome), `Selection` (which option they chose: "SMS payment link (1)" / "Speak with team (2)") and `Input` (how they chose it: "Pressed 1" / "Said: text me the link") columns
- Simple HTML frontend for upload & start

## Tech Stack

- **Backend**: FastAPI
- **Telephony**: Twilio (outbound calls + TwiML)
- **Text-to-Speech**: ElevenLabs
- **Frontend**: Basic HTML + JavaScript
- **Storage**: CSV (input), JSON (temp results), MP3 (audio files)

## Prerequisites

- Python 3.9+
- Twilio account (Account SID, Auth Token, Twilio phone number)
- ElevenLabs account & API key + Voice ID
- ngrok or public server (for Twilio webhooks)


## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/vetpay-outbound-dialer.git
   cd vetpay-outbound-dialer
   ```

2. Create virtual environment & install dependencies:
```Bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install fastapi uvicorn twilio python-multipart requests
```

3. Create config.py in the root directory:
```Python
TWILIO_ACCOUNT_SID = "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
TWILIO_AUTH_TOKEN  = "auth_token"
TWILIO_PHONE_NUMBER = "+614xxxxxxxx"      # Twilio number
BASE_URL = "https://your-ngrok-url.ngrok.io"   # Must be https
ELEVENLABS_API_KEY = "your_elevenlabs_key"
VOICE_ID = "your_voice_id"                # e.g. "Rachel", "Adam", custom voice
```

4. Create folders (automatically created on startup, but you can do it manually):
```text
audio/
output_results/
```

## Usage

1. Start the server
```Bash
uvicorn main:app --reload --port 8000
```

2. Expose locally with ngrok (recommended for testing)
```Bash
ngrok http 8000
```
Copy the https URL and update BASE_URL in config.py.

3. Open the web interface
```text
http://127.0.0.1:8000/index.html
```
Or use the ngrok URL.

4. Workflow
Upload a CSV file with columns: Client, Name, Phone
Click Start Calls
System dials contacts one by one
Results are saved in output_results/call_results_YYYYMMDD_HHMMSS.csv


## Example CSV format (contacts.csv)
```csv
Client,Name,Phone
C001,John Doe,+614123456__
C002,Emma Smith,+614876543__
C003,Alex Tan,+88017123456__
```


## Project Structure
```text
vetpay-outbound-dialer/
├── main.py             # FastAPI application
├── config.py           # (create yourself) credentials
├── index.html          # Simple frontend
├── contacts.csv        # (uploaded)
├── call_results.json   # (temporary results)
├── audio/              # TTS audio files
│   ├── common_message.mp3
│   ├── hello_<client>.mp3
│   ├── please_hold.mp3
│   ├── goodbye.mp3
│   └── thank_you_goodbye.mp3
└── output_results/     # Final CSVs with "Response", "Selection" and "Input" columns
```


## Important Notes

- Phone normalization: Handles BD (+880) and AU (+61) formats, strips spaces, etc.
- Rate limiting: Calls are made sequentially with a queue to avoid Twilio rate limits.
- Audio generation: Common message is generated once per script version (`*_v4.mp3`). Short "Hi [name]" is generated per contact if missing. Changing `COMMON_MESSAGE_TEXT` requires bumping the audio version suffix (or deleting the cached MP3) so it regenerates.
- SMS payment link: NOT sent automatically. Pressing 1 records the caller in `sms_requests.json` and confirms by voice that a link will be texted shortly — the team sends it manually from the dashboard list.
- Twilio status callbacks: Only completed events are processed.
- No duplicate outcomes: Final outcomes (`successfully_transferred`, `sms_requested`) are never overwritten by later status callbacks. SMS requests are keyed by phone, so retried calls can't duplicate rows.
- Future Improvements
- Add real-time progress dashboard
- Support retry for busy/no-answer calls
- Add call recording
- Better error handling & logging
- Docker support
- Authentication on API endpoints
