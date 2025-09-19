# VS-Joint-Classification
A semi-automatic system to classify timber joint topologies

Note:
I tried to do all the grouping first, then go through the analysis process. But the system needed to go through one by one.

import os
import time
from pathlib import Path

def _is_break(s: str) -> bool:
    return isinstance(s, str) and s.strip().lower() in {"break", "q", "quit", "exit"}

def pick_obj_folder() -> Path:
    try:
        import tkinter as tk
        from tkinter import filedialog
        tk.Tk().withdraw()
        chosen = filedialog.askdirectory(title="Select folder containing OBJ files")
        if chosen:
            p = Path(chosen)
            if p.is_dir():
                return p
    except Exception:
        pass
    while True:
        raw = input('Enter path to folder containing OBJ files (or type "break" to cancel): ').strip().strip('"')
        if _is_break(raw):
            raise KeyboardInterrupt("User cancelled folder selection.")
        p = Path(raw)
        if p.is_dir():
            return p
        print("❗ Not a valid folder. Please try again.")

def import_obj_files(obj_dir):
    obj_files = sorted([f for f in os.listdir(obj_dir) if f.lower().endswith('.obj')], key=str.lower)
    topologies = []
    for i, obj_file in enumerate(obj_files):
        obj_path = os.path.join(obj_dir, obj_file)
        topology = Topology.ByOBJPath(
            objPath=obj_path,
            defaultColor=[255, 255, 255],
            defaultOpacity=0.5,
            transposeAxes=True,
            removeCoplanarFaces=False,
            selfMerge=False,
            mantissa=6,
            tolerance=0.0001
        )
        topologies.append(topology)
        globals()[f"model_{i+1}"] = topology
        print(f"[{i+1:02}] Imported: {obj_file} as model_{i+1}")
    return topologies

def watch_obj_directory(obj_dir, interval=5):
    seen_files = set()
    while True:
        current_files = set(f for f in os.listdir(obj_dir) if f.lower().endswith('.obj'))
        new_files = current_files - seen_files
        if new_files:
            print("New OBJ files detected. Importing...")
            import_obj_files(obj_dir)
            seen_files = current_files
        elif current_files == seen_files and len(current_files) > 0:
            print("All OBJ files have been uploaded. Stopping watcher.")
            break
        time.sleep(interval)

def display_models(obj_dir):
    obj_files = [f for f in os.listdir(obj_dir) if f.lower().endswith(".obj")]
    model_vars = sorted(
        [n for n in globals() if n.startswith("model_") and n.split("_")[1].isdigit()],
        key=lambda n: int(n.split("_")[1]),
    )
    model_count = max(len(model_vars), len(obj_files))
    for i in range(1, model_count + 1):
        fname = obj_files[i - 1] if i - 1 < len(obj_files) else "(no corresponding OBJ file)"
        print(f"[{i:02d}] model_{i} = {fname}")
    return model_count

def group_models(model_count):
    while True:
        selected = input("Enter the model numbers to group (comma-separated, or 'break' to stop): ")
        if _is_break(selected):
            print("Grouping session ended.")
            break
        try:
            idx = sorted({int(s) for s in selected.replace(" ", "").split(",") if s.isdigit() and 1 <= int(s) <= model_count})
            if not idx:
                print("⚠️ No valid model numbers found. Try again.")
                continue
            names = [f"model_{i}" for i in idx]
            grouped = [globals()[n] for n in names if n in globals()]
            missing = [n for n in names if n not in globals()]
            group_name = input("Enter a name for this group: ").strip()
            if not group_name:
                print("⚠️ Group name cannot be empty.")
                continue
            groups = globals().get("groups", {})
            groups[group_name] = names
            globals()["groups"] = groups
            globals()[group_name] = grouped
            print(f"✅ Grouped models as '{group_name}': {names}")
            if missing:
                print("Note: These variables were not found and were skipped:", ", ".join(missing))
        except Exception as e:
            print(f"⚠️ Error during grouping: {e}")

if __name__ == "__main__":
    OBJ_DIR = pick_obj_folder()
    print(f"[Folder] Using OBJ directory: {OBJ_DIR}")
    print("Watching OBJ directory for new files...")
    watch_obj_directory(OBJ_DIR)
    print("Imported OBJ models:")
    model_count = display_models(OBJ_DIR)
    group_models(model_count)
