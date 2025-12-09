# 🚀 **How to use Cython for your project**

## These steps are for Windows, you may need other steps on Linux or run these steps on Linux environemnt.

### **Step 1 — Install Cython**

From your project root:

```powershell
pip install cython wheel setuptools
```

---

### **Step 2 — Create a `setup.py` file**

Create this file:

### **setup.py**

```python
from setuptools import setup, Extension
from Cython.Build import cythonize
from pathlib import Path

root = Path(__file__).parent

extensions = []

# 1) Top-level Python files
top_level_files = ["main.py", "encryptor.py"]
for file_name in top_level_files:
    module_name = file_name.replace(".py", "")  # main, encryptor
    extensions.append(Extension(
        module_name,
        [str(root / file_name)]
    ))

# 2) All python files inside src/ (recursive)
src_dir = root / "src"
for py_file in src_dir.rglob("*.py"):
    # convert "src/utils/helper.py" → "src.utils.helper"
    rel_path = py_file.relative_to(root).with_suffix("")
    module_name = ".".join(rel_path.parts)

    extensions.append(Extension(
        module_name,
        [str(py_file)]
    ))

setup(
    name="fastapi_app_cython",
    ext_modules=cythonize(
        extensions,
        compiler_directives={"language_level": "3"},
        annotate=False
    ),
)
```
---

# ✅ STEP 3 — Build with this improved setup

Run from inside:

```powershell
(.venv) PS> python setup.py build_ext --inplace
```

You will see `.pyd` files placed inside the correct folders.

---

# ✅ STEP 4 — Remove original un-wanted files (you only need the .pyd (Windows) files)

Safe for production:

Here are **safe Windows PowerShell commands** to delete all **.py**, **.c**, and other unneeded files while keeping only the compiled **.pyd** files and your project assets.

---

## ✅ **1. Delete all `.c` files**

```powershell
Get-ChildItem -Recurse -Filter *.c | Remove-Item -Force
```

---

## ✅ **2. Delete all `.py` files but KEEP `__init__.py` (required for packages)**

If you want to **keep `__init__.py`**, use:

```powershell
Get-ChildItem -Recurse -Filter *.py | Where-Object { $_.Name -ne "__init__.py" } | Remove-Item -Force
```

If you want to **delete ALL `.py` files including `__init__.py`** (only do this if your imports still work):

```powershell
Get-ChildItem -Recurse -Filter *.py | Remove-Item -Force
```

---

## ✅ **3. Delete build folders created by Cython**

If you followed the usual steps:

```powershell
Remove-Item -Recurse -Force build
Remove-Item -Recurse -Force dist
Remove-Item -Recurse -Force *.egg-info
```

---

## ✅ **4. Delete all `.pyc` and `__pycache__`**

(These are useless after compilation)

```powershell
Get-ChildItem -Recurse -Filter *.pyc | Remove-Item -Force
Get-ChildItem -Recurse -Directory -Filter "__pycache__" | Remove-Item -Recurse -Force
```

---

## ✅ **5. Delete setup files if you used Cython**

```powershell
Remove-Item setup.py -Force
```

---

# 📌 **Summary of what you KEEP**

✔ `.pyd` files
✔ Static files (HTML, CSS, JS, etc.)
✔ Config files
✔ `__init__.py` (optional but recommended)

---


# ✅ STEP 5 — Run FastAPI normally

- Cython compiled modules load using the same import name.
- You may need to perform some fixes to the code

```powershell
uvicorn main:app
```


---
