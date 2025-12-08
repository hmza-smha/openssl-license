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
* Deliver both files to customer ```license.json``` & ```license.sig``` (replace old ones).

# 7) Key rotation & revocation (brief)

* Add `kid` in `license.json`. App may map `kid` → public key file and choose correct public key for verification.
* Maintain server-side revocation DB of `serial` values; app can check online revocation endpoint periodically if you need revocation (offline apps rely solely on expiry).

# 8) Security notes (short)

* Never ship `private.pem`. Protect signing endpoint.
* Use UTC times and `DateTime.UtcNow`.
* For stronger signatures use RSASSA-PSS (must match OpenSSL sign/verify options).
* For offline use set reasonable expiry and require periodic renewal to detect revocations.

---
