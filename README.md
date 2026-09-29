# UniGuard — SIH26145 Prototype

A technically defensible local prototype for **AI-Based Detection of Cyber Threats in Unidirectional IP Traffic**.

The runnable path uses Python's standard-library `http.server` for the web/API server and a real scikit-learn `RandomForestClassifier` trained from the generated labelled flow scenarios. FastAPI is not required for the runnable prototype.

## What is implemented

- Standard-library HTTP service with read-only flow ingestion.
- A trained RandomForest model with held-out replay metrics and per-alert confidence.
- JSONL replay through the same ingest path used by live flow input.
- Deterministic rules for DDoS, reconnaissance, DNS tunnelling/DGA, beaconing, encrypted-session metadata and exfiltration.
- Novelty/anomaly path for suspicious metadata.
- Standardized evidence-backed alert records.
- Hash-chained alert integrity fields.
- Durable SQLite evidence ledger with WAL mode and full chain restoration after restart.
- Analyst feedback endpoint with no response/mitigation operation.
- Live dashboard polling the API.
- Corrected evaluation totals: test rows and confusion-matrix support are both 1,330.
- Explicit benign false-positive-rate metric.
- Metadata-only handling: no TLS/QUIC decryption and no payload storage.
- Offline-capable deterministic replay.

This is a prototype detector and integration harness. It is not a production SOC sensor and should not be connected to a real network without sponsor-approved controls and capture-separated validation.

## Run locally

```bash
cd /home/ubuntu/uniguard-sih26145
python3 -m pip install -r requirements.txt
python3 generate_demo_data.py
PORT=8787 python3 server_stdlib.py
```

Open http://127.0.0.1:8787 in a browser.

Click **Start replay** to stream the JSONL scenarios through the stdlib server's `/api/ingest` path. Click **Inject test flow** to send a synthetic high-rate flow through the RandomForest inference API.

## One-click launchers

Do not double-click `index.html`; the dashboard needs the local HTTP backend. Use the launcher for your operating system instead.

### Windows

Double-click `launch_uniguard.bat`. It creates `.venv`, installs scikit-learn and NumPy, generates replay data, trains the RandomForest at startup, starts the stdlib HTTP API and opens `http://127.0.0.1:8787` in the default browser.

If Windows blocks the script, right-click it, choose **Properties**, select **Unblock** if shown, and click **Apply**. Python 3.10+ must be installed.

### macOS/Linux

Run once from Terminal:

```bash
chmod +x launch_uniguard.sh
./launch_uniguard.sh
```

The script creates `.venv`, installs dependencies, generates replay data, trains the RandomForest, starts the stdlib HTTP API and opens the dashboard. Press `Ctrl+C` to stop it.

## API smoke test

```bash
curl http://127.0.0.1:8787/api/health
curl http://127.0.0.1:8787/api/state
curl -X POST http://127.0.0.1:8787/api/ingest \
  -H 'content-type: application/json' \
  -d '{"flow_id":"smoke-ddos","packets":3000,"bytes":500000,"duration_ms":500,"syn_count":120}'
```

There is deliberately no `/block`, `/quarantine`, `/probe`, `/shell` or response-control endpoint.

## Persistent evidence store

Chain records are persisted in `data/uniguard_evidence.sqlite3` using SQLite WAL mode and `synchronous=FULL`. Each suspicious flow creates one append-only row containing the alert metadata, evidence, model version, `prev_hash`, `record_hash` and creation timestamp. On restart, the server restores the latest 5,000 alerts into memory and restores the total chain count and latest hash from SQLite.

The durable ledger is available through:

```text
GET /api/evidence?limit=100
```

The in-memory alert window is only a display/cache limit. The SQLite record count is the durable chain count.

## One-way demo boundary

`docker-compose.yml` documents the intended offline topology: the replay/generator writes a JSONL file into a shared inbox, while the detector service consumes the file. The generator has no network attachment. For a production-like deployment, replace the shared file drop with a sponsor-approved data diode, one-way mirror or IPFIX exporter.

The current local process is intentionally simple and inspectable. The next production increment is to add Zeek `conn.log`, `dns.log` and `ssl.log` adapters or IPFIX/NetFlow/sFlow ingestion, Redis Streams/Kafka, rolling windows and a real trained model artifact.

## Defensible evaluation notes

The embedded evaluation record is explicitly labelled **held-out synthetic-lab flow windows**. Before claiming generalization, run capture-separated tests across CIC-IDS2017, CSE-CIC-IDS2018, UNSW-NB15 and project-generated private-lab captures. Report macro-F1, per-class recall, false alerts/hour, p50/p95 latency, throughput, dropped records and unknown-class detection.
