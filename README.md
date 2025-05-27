# HSN-ASSESSMENT
hsn-validator-agent/
│
├── app/
│   ├── __init__.py
│   ├── validator.py         # Core validation logic
│   ├── loader.py            # Excel data loader
│   └── config.py            # Configuration constants
│
├── hsn_master/              
│   └── HSN_SAC.xlsx         # Your uploaded dataset
│
├── main.py                  # Flask app entry point
├── requirements.txt         # Dependencies
├── README.md                # Project documentation
└── webhook_sample.json      # Dialogflow/ADK webhook request example

 requirements.txt
txt
Copy
Edit
flask
pandas
openpyxl
 app/config.py
python
Copy
Edit
EXCEL_PATH = "hsn_master/HSN_SAC.xlsx"
 app/loader.py
python
Copy
Edit
import pandas as pd
from app.config import EXCEL_PATH

def load_hsn_data():
    df = pd.read_excel(EXCEL_PATH, engine='openpyxl')
    df.dropna(subset=['HSNCode'], inplace=True)
    df['HSNCode'] = df['HSNCode'].astype(str).str.strip()
    df['Description'] = df['Description'].astype(str).str.strip()
    return df
 app/validator.py
python
Copy
Edit
def is_valid_format(code):
    return code.isdigit() and len(code) in {2, 4, 6, 8}

def exists_in_master(code, df):
    return code in df['HSNCode'].values

def get_hierarchy(code):
    return [code[:i] for i in [2, 4, 6] if len(code) > i]

def validate_code(code, df):
    code = str(code).strip()
    
    if not is_valid_format(code):
        return {
            "HSNCode": code,
            "Status": "Invalid",
            "Reason": "Invalid format: must be numeric and 2–8 digits long."
        }
    
    result = {
        "HSNCode": code,
        "Status": "Valid" if exists_in_master(code, df) else "Invalid",
    }

    if result["Status"] == "Valid":
        result["Description"] = df[df['HSNCode'] == code]['Description'].values[0]
        # Check for parent hierarchy
        missing = [p for p in get_hierarchy(code) if p not in df['HSNCode'].values]
        if missing:
            result["Status"] = "Partially Valid"
            result["MissingParents"] = missing
    else:
        result["Reason"] = "HSN code not found in master dataset."
    
    return result
📄 main.py
python
Copy
Edit
from flask import Flask, request, jsonify
from app.loader import load_hsn_data
from app.validator import validate_code

app = Flask(__name__)
hsn_df = load_hsn_data()

@app.route("/")
def home():
    return "HSN Code Validation Agent is running."

@app.route("/validate", methods=["POST"])
def validate():
    data = request.get_json()
    codes = data.get("hsn_codes", [])
    
    if not codes:
        return jsonify({"error": "No HSN codes provided."}), 400

    results = [validate_code(code, hsn_df) for code in codes]
    return jsonify({"results": results}), 200

if __name__ == "__main__":
    app.run(debug=True)
 webhook_sample.json
json
Copy
Edit
{
  "hsn_codes": ["01", "0101", "01011010", "12345678", "9999", "A101"]
}
 README.md
md
Copy
Edit
# HSN Code Validation Agent

A Python-based webhook service to validate HSN codes using the Google ADK or Dialogflow integration.

## Features
- Checks for valid HSN format (2, 4, 6, 8 digits)
- Validates existence from master Excel
- Optional hierarchy parent checking
- Fast response using pre-loaded dataset

## How to Run

1. Clone this repository
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
Start the Flask server:

bash
Copy
Edit
python main.py
Send POST requests to /validate endpoint with JSON payload:

json
Copy
Edit
{
  "hsn_codes": ["0101", "9999"]
}
Dataset
Place HSN_SAC.xlsx inside the hsn_master/ folder.

Example Response
json
Copy
Edit
{
  "results": [
    {
      "HSNCode": "0101",
      "Status": "Valid",
      "Description": "LIVE HORSES, ASSES, MULES AND HINNIES"
    },
    {
      "HSNCode": "9999",
      "Status": "Invalid",
      "Reason": "HSN code not found in master dataset."
    }
  ]
}
yaml
Copy
Edit

---



