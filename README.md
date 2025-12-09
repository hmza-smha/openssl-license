# How to Use OpenSSL on Windows

### 1. Download OpenSSL for Windows
OpenSSL isn’t included by default in Windows. You can:
- Get an installer from [Shining Light Productions](https://slproweb.com/products/Win32OpenSSL.html).
  - Choose **Win64 OpenSSL** if you have a 64-bit Windows.
  - Download the **.exe installer**.

### 2. Install OpenSSL
- Run the installer.
- During installation, choose to add OpenSSL to the Windows system `PATH` (if prompted).
  - If not, you’ll need to manually add the installation folder (e.g., `C:\Program Files\OpenSSL-Win64\bin`) to your system `PATH` afterwards.

### 3. Add OpenSSL to your PATH (if necessary)
- Press **Win+R**, type `sysdm.cpl`, and press Enter.
- Go to **Advanced tab > Environment Variables**.
- In “System variables,” find and edit `Path`, adding your OpenSSL `bin` folder (e.g., `C:\Program Files\OpenSSL-Win64\bin`).
- Click OK.

### 4. Open a New Command Prompt Window

Close any old windows. Open a new **Command Prompt** and try:

```shell
openssl version
```
- You should see OpenSSL’s version, confirming installation.
- 
---

# 1) Keys (OpenSSL)

```bash
# generate RSA 4096-bit private key (keep private.pem secret)
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:4096

# extract public key (distribute public.pem)
openssl rsa -pubout -in private.pem -out public.pem
```

# 2) License file format (`license.json`)

Use UTC ISO timestamps.

```json
{
  "serial": "LIC-2025-0001",
  "issuedAt": "2025-12-08T00:00:00Z",
  "expiry": "2026-03-08T00:00:00Z",
  "features": { "pro": true, "maxUsers": 5 },
  "kid": "key-v1"
}
```

# 3) Sign (produces `license.sig` binary)

```bash
# sign JSON bytes with private key (SHA256)
openssl dgst -sha256 -sign private.pem -out license.sig license.json

# optional: create base64 text signature for transport
base64 license.sig > license.sig.b64
```

# 4) Verify (OpenSSL quick test)

```bash
openssl dgst -sha256 -verify public.pem -signature license.sig license.json
# prints "Verified OK" if valid
```

# 5) ASP.NET Core — verify signature & expiry (minimal)

```csharp
// LicensePayload.cs
public class LicensePayload {
  public string Serial { get; set; }
  public DateTime IssuedAt { get; set; }
  public DateTime Expiry { get; set; }
  public JsonElement Features { get; set; }
  public string Kid { get; set; }
}
```

```csharp
// LicenseUtils.cs
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

public static class LicenseUtils {
  public static RSA LoadPublicFromPem(string pem) {
    var rsa = RSA.Create();
    rsa.ImportFromPem(pem.ToCharArray());
    return rsa;
  }

  public static bool VerifySignature(string payloadJson, byte[] signature, RSA publicRsa) {
    var data = Encoding.UTF8.GetBytes(payloadJson);
    return publicRsa.VerifyData(data, signature, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
  }

  public static (bool ok, bool expired, LicensePayload payload) Validate(string licenseJsonPath, string signaturePath, RSA publicRsa) {
    var payloadJson = File.ReadAllText(licenseJsonPath);
    var sig = File.ReadAllBytes(signaturePath);
    if (!VerifySignature(payloadJson, sig, publicRsa)) return (false, true, null);
    var payload = JsonSerializer.Deserialize<LicensePayload>(payloadJson, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
    var expired = DateTime.UtcNow > payload.Expiry;
    return (true, expired, payload);
  }
}
```

```csharp
// Program.cs (middleware)
var publicPem = File.ReadAllText("public.pem");
using var publicRsa = LicenseUtils.LoadPublicFromPem(publicPem);

app.Use(async (ctx, next) => {
  const string licJson = "license.json";
  const string licSig = "license.sig";
  if (!File.Exists(licJson) || !File.Exists(licSig)) {
    ctx.Response.StatusCode = 403; await ctx.Response.WriteAsync("License missing."); return;
  }
  var (ok, expired, payload) = LicenseUtils.Validate(licJson, licSig, publicRsa);
  if (!ok) { ctx.Response.StatusCode = 403; await ctx.Response.WriteAsync("Invalid license."); return; }
  if (expired) { ctx.Response.StatusCode = 403; await ctx.Response.WriteAsync($"License expired {payload?.Expiry:O} (UTC)."); return; }
  ctx.Items["License"] = payload;
  await next();
});
```

# 6) Regenerate license (licensing server)

* Create new `license.json` with new `expiry`.
* Sign with `private.pem`: `openssl dgst -sha256 -sign private.pem -out license.sig license.json`
* Deliver both files to customer ```license.json``` & ```license.sig``` (replace old ones), and they should have the ```public.pem```.

# 7) Key rotation & revocation (brief)

* Add `kid` in `license.json`. App may map `kid` → public key file and choose correct public key for verification.
* Maintain server-side revocation DB of `serial` values; app can check online revocation endpoint periodically if you need revocation (offline apps rely solely on expiry).

# 8) Security notes (short)

* Never ship `private.pem`. Protect signing endpoint.
* Use UTC times and `DateTime.UtcNow`.
* For stronger signatures use RSASSA-PSS (must match OpenSSL sign/verify options).
* For offline use set reasonable expiry and require periodic renewal to detect revocations.

---

You **cannot fully prevent** a determined user from modifying IL code,
**but you can make it extremely difficult + impossible to run without your server**.

Below is the real-world way companies protect .NET licensing.

---

# ✅ 1) Use .NET Obfuscation (MUST)

Use a commercial obfuscator that supports **control-flow**, **string-encryption**, **method encryption**, **anti-tamper**:

* **ConfuserEx (free, OK)**
* **Babel Obfuscator**
* **Eazfuscator.NET**
* **.NET Reactor (very strong)**
* **Agile.NET (commercial)**

This makes removing middleware **non-trivial**.

---

# ✅ 2) Put license logic in an internal DLL wrapped with anti-tamper

Example structure:

```
WebApp.dll   → calls LicenseRuntime.dll → verifies signature
LicenseRuntime.dll → obfuscated + anti-tamper
```

**Do NOT put verification directly in Program.cs.**
Put it inside the protected DLL so they cannot patch Program.cs easily.

---

# ...
---

# 🔥 Real Answer (Industry Standard)

You can **never** stop a determined hacker from modifying IL.
But you can:

* Make patching extremely time-consuming
* Make reverse engineering very expensive
* Make the app rely on server-side validation
* Require periodic online renewal
* Bind license to device
* Use strong obfuscation + anti-tamper

This is how commercial .NET software survives.

---

