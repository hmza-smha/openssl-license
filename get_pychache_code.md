1. Compile your FastAPI project (excluding folders you don't want)
2. Build a clean deploy folder
3. Deploy to Ubuntu (Nginx + Uvicorn) exactly as usual

This is the method you want.

---

# ✅ **1. Correct compile command with folder/file exclusions**

`compileall` **cannot exclude paths directly**, so the clean solution is:

### **Step 1 — Create a clean `deploy/` folder with only wanted folders**

Run this from PowerShell in your project root:

```powershell
python -m compileall -b .
```

This produces `.pyc` files **next to** your `.py` files.

### **Step 3 — Remove all `.py` files**

```powershell
Get-ChildItem .\deploy -Recurse -Filter *.py | Remove-Item
```
---

# ✅ **2. How to deploy to Ubuntu Nginx**

Exactly the same as before — nothing changes in your deployment process.
