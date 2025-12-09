Below is a **complete, practical, production-ready method** to implement license signing + verification for your FastAPI (or any Python) application using **OpenSSL** with **RSA keys**.

---

# ✅ **STEP 1 — Generate RSA Keys with OpenSSL**

Run these commands **on your machine (never share private key)**:

### **1. Create a 4096-bit private key**

```bash
openssl genrsa -out private_key.pem 4096
```

### **2. Extract the public key**

```bash
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

**You will ship only**: `public_key.pem`
**You will keep private**: `private_key.pem`

---

# ✅ **STEP 2 — Create a License File (JSON)**

Example license (license.json):

```json
{
  "customer": "ACME Corp",
  "expiry": "2025-12-09T07:20:54Z", // UTC
  "max_users": 50,
  "features": ["core", "reports"],
  "machine_id": "SERVER-1234"
}
```

The client receives:

* `license.json`
* `license.sig`
* `public_key.pem`

---

# ✅ **STEP 3 — Sign the License File Using OpenSSL**

You generate the license signature with your **private key**:

```bash
openssl dgst -sha256 -sign private_key.pem -out license.sig license.json
```

---

# ✅ **STEP 4 — Your Python App: Verify the License**

Install cryptography:

```bash
pip install cryptography
```

Verification code:

```python
import json
import uuid
from datetime import datetime, timezone
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding


LICENSE_JSON_PATH = "license.json"
LICENSE_SIG_PATH = "license.sig"
PUBLIC_KEY_PATH = "public_key.pem"


# ---------------------------------------------------------
# Helper: Load public key
# ---------------------------------------------------------
def load_public_key():
    with open(PUBLIC_KEY_PATH, "rb") as f:
        return serialization.load_pem_public_key(f.read())


# ---------------------------------------------------------
# Helper: Parse UTC date (supports "Z")
# ---------------------------------------------------------
def parse_utc(date_str):
    # Convert "Z" to "+00:00" to avoid parsing issues
    if date_str.endswith("Z"):
        date_str = date_str.replace("Z", "+00:00")
    return datetime.fromisoformat(date_str)


# ---------------------------------------------------------
# Machine ID (if you want binding)
# ---------------------------------------------------------
def get_machine_id():
    # MAC address as machine ID
    return hex(uuid.getnode())


# ---------------------------------------------------------
# Verify RSA signature against license.json
# ---------------------------------------------------------
def verify_signature():
    # Read license data
    with open(LICENSE_JSON_PATH, "rb") as f:
        data = f.read()

    # Read binary signature
    with open(LICENSE_SIG_PATH, "rb") as f:
        signature = f.read()

    # Load public key
    public_key = load_public_key()

    try:
        public_key.verify(
            signature,
            data,
            padding.PKCS1v15(),
            hashes.SHA256()
        )
        return True

    except Exception as e:
        print("Signature verification failed:", e)
        return False


# ---------------------------------------------------------
# License validation logic
# ---------------------------------------------------------
def validate_license():
    # STEP 1 — verify the signature
    if not verify_signature():
        raise RuntimeError("❌ Invalid license: signature mismatch")

    # STEP 2 — load license JSON
    with open(LICENSE_JSON_PATH, "r") as f:
        lic = json.load(f)

    # STEP 3 — expiry check using UTC
    expiry_utc = parse_utc(lic["expiry"])          # datetime object
    now_utc = datetime.now(timezone.utc)

    if now_utc > expiry_utc:
        raise RuntimeError(f"❌ License expired on {expiry_utc.isoformat()}")

    print("✔ License is valid.")
    return lic  # Return parsed license information
```

---

# ✅ **STEP 5 — Enforce the License Inside FastAPI**

In your FastAPI startup:

```python
import json
from fastapi import FastAPI
from datetime import datetime

from license_utils import validate_license

app = FastAPI()

def load_license_json():
    with open("license.json") as f:
        return json.load(f)
    
@app.on_event("startup")
def check_license_on_startup():
    validate_license()

@app.get("/")
def read_root():
    return {"message": "Hello, World!"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

If the license is invalid, the app **refuses to start**.

---

# ⭐ OPTIONAL: Bind License to Specific Machine

You can bind using:

* hostname
* MAC address
* CPU serial
* custom hardware fingerprint

Example machine ID collector:

```python
import uuid
def get_machine_id():
    return hex(uuid.getnode())
```

Then compare it with the license value.

---

### **On your machine**

* edit `license.json`
* sign it → `license.sig.b64`
* send both to customer

### **On customer’s server**

* they place:

  * `license.json`
  * `license.sig.b64`
  * `public_key.pem`
* your app verifies at startup

You never expose:

* private RSA key
* internal secrets
* your cloud credentials

---
