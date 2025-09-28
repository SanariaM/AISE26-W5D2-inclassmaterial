AISE26 — W5D2 In-Class Material

Domain-Driven Design + Hexagonal (Ports & Adapters) — FastAPI

Prerequisites:
Python 3.10+ (3.11 recommended)
pip (bundled with Python)
(Optional) Git and VS Code

Quickstart — Mac/Linux (bash/zsh)

# 1) (Optional) open project folder
cd /path/to/AISE26-W5D2-inclassmaterial

# 2) Create & activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 3) Install dependencies
pip install -r requirements.txt

# 4) Run unit tests
pytest -q

# 5) Start the API (FastAPI + Uvicorn)
uvicorn app:app --reload
# → http://127.0.0.1:8000/docs  (Swagger UI)
Quickstart — Windows (PowerShell)
# 1) Go to the project
cd C:\path\to\AISE26-W5D2-inclassmaterial

# 2) Create & activate virtual environment
python -m venv .venv
.venv\Scripts\Activate    # (note the capital A in PowerShell)

# 3) Install dependencies
pip install -r requirements.txt

# 4) Run unit tests
pytest -q

# 5) Start the API
uvicorn app:app --reload
# → http://127.0.0.1:8000/docs


Windows (Command Prompt alternative):

cd C:\path\to\AISE26-W5D2-inclassmaterial
python -m venv .venv
.venv\Scripts\activate.bat
pip install -r requirements.txt
pytest -q
uvicorn app:app --reload

If uvicorn isn’t recognized on Windows/Mac, run:
python -m uvicorn app:app --reload
