

```
dfs["brocade"]["switchshow"]
dfs["brocade"]["fabricshow"]
```

---

# 📁 Estructura final

```
v1.py
dvl/
    ├── brocadeHelper.py
    ├── powermaxHelper.py
    ├── functionHelper.py
    ├── reportHelper.py   <-- modificado para devolver dfs por comando
    ├── logHelper.py
inventory.json
```

---

# 1️⃣ **v1.py — script principal ejecutando collectors en paralelo**

```python
import json
import concurrent.futures

from dvl.brocadeHelper import brocadeCollector
from dvl.powermaxHelper import powermaxCollector
from dvl.purestorageHelper import pureCollector
from dvl.netappHelper import netappCollector
from dvl.ecsHelper import ecsCollector
from dvl.datadomainHelper import ddCollector
from dvl.netbackupHelper import nbCollector

from dvl.reportHelper import build_reports
from dvl.logHelper import init_logging


def run_all_collectors():
    init_logging()

    # Cargar inventario JSON
    with open("inventory.json") as f:
        inventory = json.load(f)

    # Mapear collectors
    collectors = {
        "brocade": lambda: brocadeCollector(inventory["brocade"]),
        "powermax": lambda: powermaxCollector(inventory["powermax"]),
        "pure": lambda: pureCollector(inventory["pure"]),
        "netapp": lambda: netappCollector(inventory["netapp"]),
        "ecs": lambda: ecsCollector(inventory["ecs"]),
        "datadomain": lambda: ddCollector(inventory["datadomain"]),
        "netbackup": lambda: nbCollector(inventory["netbackup"]),
    }

    results = {}

    print("\n[INFO] Ejecutando todos los collectors en paralelo...\n")

    # 1️⃣ Ejecutar todos los collectors en paralelo
    with concurrent.futures.ThreadPoolExecutor(max_workers=len(collectors)) as executor:
        future_map = {
            executor.submit(func): vendor
            for vendor, func in collectors.items()
        }

        for future in concurrent.futures.as_completed(future_map):
            vendor = future_map[future]
            try:
                results[vendor] = future.result()
            except Exception as e:
                print(f"[ERROR] Collector {vendor}: {e}")
                results[vendor] = None

    print("\n[INFO] Consolidando datos y generando DataFrames...\n")

    # 2️⃣ Reportes — ahora devuelve diccionario por comando
    dfs = build_reports(results)

    return dfs


if __name__ == "__main__":
    dfs = run_all_collectors()

    # Ejemplo de acceso:
    df_switchshow = dfs["brocade"]["switchshow"]
    df_fabricshow = dfs["brocade"]["fabricshow"]

    print(df_switchshow.head())
```

---

# 2️⃣ **brocadeHelper.py — usando inventario y generando salida por sistema**

Este es el *collector base*, el resto de fabricantes usarán el mismo patrón.

```python
import concurrent.futures
from dvl.functionHelper import run_switchshow, run_fabricshow

def run_all_functions_for_switch(system_name, data):
    """
    Ejecuta secuencialmente todos los comandos para 1 switch.
    `data` proviene del inventario.
    """
    results = {}

    results["switchshow"] = run_switchshow(data)
    results["fabricshow"] = run_fabricshow(data)

    return system_name, results


def brocadeCollector(inventory_brocade):
    """
    inventory_brocade = {
        "SWITCH1": {...},
        "SWITCH2": {...},
        ...
    }
    """
    output = []

    with concurrent.futures.ThreadPoolExecutor(max_workers=len(inventory_brocade)) as executor:
        futures = {
            executor.submit(run_all_functions_for_switch, system_name, system_data): system_name
            for system_name, system_data in inventory_brocade.items()
        }

        for future in concurrent.futures.as_completed(futures):
            name = futures[future]

            try:
                system_name, data = future.result()
                output.append({"system": system_name, "data": data})
            except Exception as e:
                output.append({"system": name, "error": str(e)})

    return output
```

Los demás helpers: `powermaxHelper.py`, `pureHelper.py`, etc. usarán este mismo formato.

---

# 3️⃣ **functionHelper.py — ejemplo realista**

```python
import paramiko

def ssh_run(host, user, password, command):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(host, username=user, password=password)
    stdin, stdout, stderr = client.exec_command(command)
    out = stdout.read().decode()
    err = stderr.read().decode()
    client.close()
    return out, err


def run_switchshow(system):
    out, err = ssh_run(system["fqdn"], system["username"], system["password"], "switchshow")
    return parse_switchshow(out)

def run_fabricshow(system):
    out, err = ssh_run(system["fqdn"], system["username"], system["password"], "fabricshow")
    return parse_fabricshow(out)


def parse_switchshow(output):
    rows = []
    for line in output.splitlines():
        rows.append({"raw": line})
    return rows

def parse_fabricshow(output):
    rows = []
    for line in output.splitlines():
        rows.append({"raw": line})
    return rows
```

---

# 4️⃣ **reportHelper.py — retorna df por fabricante y por comando**

Este es el módulo clave, ahora modificado para que puedas hacer:

```
dfs["brocade"]["switchshow"]
dfs["brocade"]["fabricshow"]
```

```python
import pandas as pd

def build_reports(results):
    """
    results = {
        "brocade": [
            { "system": "SW1", "data": { "switchshow": [...], "fabricshow": [...] }},
            { "system": "SW2", "data": {...}},
        ],
        "powermax": [...]
    }

    Devuelve:
    {
        "brocade": {
            "switchshow": DataFrame,
            "fabricshow": DataFrame,
        }
    }
    """
    vendor_dfs = {}

    for vendor, vendor_results in results.items():
        if vendor_results is None:
            continue

        bucket = {}  # por comando

        # Agrupar resultados por comando
        for item in vendor_results:
            system = item.get("system")
            data = item.get("data", {})

            for command, rows in data.items():
                if command not in bucket:
                    bucket[command] = []

                for row in rows:
                    row["_system"] = system
                    bucket[command].append(row)

        # Convertir bucket → dict de dataframes
        dfs_por_comando = {
            command: pd.DataFrame(rows)
            for command, rows in bucket.items()
        }

        vendor_dfs[vendor] = dfs_por_comando

    return vendor_dfs
```

---

# 5️⃣ **Acceso a los dataframes**

Una vez que `run_all_collectors()` termina:

```python
dfs = run_all_collectors()

df_switchshow = dfs["brocade"]["switchshow"]
df_fabricshow = dfs["brocade"]["fabricshow"]

print(df_switchshow)
```

---

# ✔️ Si quieres, puedo agregar:

### 🔸 Export a PostgreSQL por comando

### 🔸 Manejo de errores por sistema + retry

### 🔸 Timeouts por comando

### 🔸 Uso de asyncio + asyncssh para máxima velocidad

### 🔸 Pool de conexiones SSH reutilizable

Solo dime.
