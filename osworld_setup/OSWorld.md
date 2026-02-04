# OSWorld Installation & Deploying `Muscle_mem_agent`

> Goal: Install **OSWorld** (Python **3.12** via **uv**), deploy **Muscle_mem_agent**, switch the AWS AMI, and run evaluation jobs on the **AWS** provider.

---

## 1) Install OSWorld (uv + Python 3.12)

Enter the OSWorld repo directory:

```bash
cd OSWorld
```

Sync dependencies and create the virtual environment (increase timeout if network is slow):

```bash
UV_HTTP_TIMEOUT=300 uv sync
```

If your Python installation does not include `pip`, bootstrap it:

```bash
source OSWorld/.venv/bin/activate
python -m ensurepip --upgrade
```

> `uv sync` typically creates the venv at `OSWorld/.venv/`.

---

## 2) Deploy `Muscle_mem_agent` into OSWorld

### Step 1: Activate OSWorld virtual environment

Run from anywhere, but ensure the path points to OSWorld’s `.venv`:

```bash
source OSWorld/.venv/bin/activate
```

> Recommendation: keep installs/runs inside the same environment to avoid dependency mismatch.

### Step 2: Install `Muscle_mem_agent` (editable)

Enter your agent repo/dir and install:

```bash
cd Muscle_mem_agent
python -m pip install -e .
```

---

## 3) Copy run scripts into OSWorld

Copy the run entry scripts from `osworld_setup/` into OSWorld (adjust paths to your layout):

```bash
cp osworld_setup/*.py ../OSWorld/
```

> Core intent: place `run_muscle_mem_agent.py` somewhere OSWorld can execute directly.

---

## 4) Switch the AWS AMI

Edit:

- `desktop_env/providers/aws/manager.py`

Set the AMI to:

```python
"ami-0b505e9d0d99ba88c"
```

> Tip: search for `ami-` in the file to locate the relevant field quickly.

---

## 5) Run (from OSWorld directory)

Enter OSWorld:

```bash
cd OSWorld
```

Run:

```bash
uv run python run_muscle_mem_agent.py \
  --provider_name "aws" \
  --headless \
  --num_envs 4 \
  --max_steps 100 \
  --domain "all" \
  --test_all_meta_path evaluation_examples/test_nogdrive.json \
  --result_dir "results_nogdrive_last_v1" \
  --model_provider "anthropic" \
  --model_url "xxx" \
  --model_api_key "xxx" \
  --model "claude-4-5-opus" \
  --model_temperature 1.0 \
  --ground_provider "openai" \
  --ground_url "xxx" \
  --ground_model "qwen3-vl-plus" \
  --ground_api_key "xxx" \
  --grounding_width 1000 \
  --grounding_height 1000 \
  --image_ground_model "doubao-seed-1-8-251228" \
  --image_ground_provider "openai" \
  --image_ground_url "xxx" \
  --image_ground_api_key "xxx" \
  --tavily-api-key "xxx" \
  --jina-api-key "xxx"
```
