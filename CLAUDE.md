# Dark Store Order Picker — project notes for Claude Code

## What we are building
A hackathon project (Nebius x NVIDIA Global AI Hackathon, Physical AI track, deadline 30 Oct 2026).
A **mobile robot** (mecanum base + SO-100 arm on Jetson AGX) picks and packs online grocery orders.
**NVIDIA Nemotron** (via Nebius Token Factory) is the "brain": it reads the order, plans the picks,
tells the robot what to do, handles problems, and writes the bill.
**NVIDIA GR00T** runs the pick policy on the Jetson. Nemotron gives GR00T a short text instruction,
for example "pick up the soup can".

Flow: customer order (touchscreen) -> queued in SQLite -> worker picks up order ->
Nemotron agent: navigate to shelf -> pick, scan, place each item -> navigate to customer ->
navigate home -> bill + summary in the app.

## Team (4 people)
- App flow (this repo, `app/`): web app, Nemotron agent, inventory, talking to the robot.
- Imitation learning (1 person): GR00T pick policy that runs inside the robot server `/pick`.
- Hardware (2 people): mobile robot (mecanum base + SO-100 arm), cameras, Jetson AGX.

## Architecture
```
                    Streamlit multipage app (3080 PC)
                   /                                  \
    [Customer page]                              [Owner dashboard]
    (Raspberry Pi browser)                       (laptop browser)
         |                                              |
         +-----------------+---------------------------+
                           |
                     [SQLite: data/orders.db]
                           |
                     [Background worker]
                           |
                     [Agent (Nemotron)]
                           |
                     [robot_client.py]
                           |
                      HTTP :8000
                           |
                     [Robot server on Jetson AGX]
                     - ROS 2 Nav2 (navigation between stations)
                     - GR00T policy (pick skill)
                     - Camera feed
```

### Pages
1. **Customer page** (runs on Raspberry Pi touchscreen):
   - Pick products, cart, FAKE payment button, order status.
2. **Owner dashboard** (runs on laptop browser):
   - All orders, live stock, live robot task and station, live camera feed.

### Data
- Both pages share data through SQLite file (`data/orders.db`).
- Order fields: id, created_at, items (JSON), status, step_log (JSON),
  bill (JSON), problems (JSON), current_task, station.
- Status flow: "queued" -> "packing" -> "done" or "failed".

### Robot
- Mobile: mecanum base + SO-100 arm on Jetson AGX.
- Navigation: ROS 2 Nav2 between fixed stations: "home", "shelf_A", "shelf_B", "shelf_C", "customer".
- Pick skill: GR00T policy. Nemotron gives it a text instruction.

## Robot API contract (v2)
| Method | Path | Body | Response |
|---|---|---|---|
| GET | /health | - | {"status": "ok"} |
| GET | /status | - | {"busy", "holding", "station", "pose", "battery", "speed", "nav_state", "obstacle", "last_error"} |
| GET | /map | - | {"stations": {"home": {"x", "y", "theta"}, ...}} |
| POST | /navigate | {"station": "home" \| "shelf_A" \| "shelf_B" \| "shelf_C" \| "customer"} | {"success", "message"} |
| POST | /pick | {"item": "soup_can", "shelf": "shelf_A", "instruction": "..."} | {"success", "message", "item"} |
| POST | /scan | - | {"success", "barcode", "product"} |
| POST | /place | {"target": "basket"} or {"target": "return"} | {"success", "message"} |
| POST | /stop | - | {"success", "message"} — emergency stop |
| POST | /resume | - | {"success", "message"} — resume after stop |
| GET | /camera | - | MJPEG stream (optional, for dashboard) |

- `/status` returns full state: pose {x, y, theta}, battery %, speed, nav_state, obstacle flag, last_error.
- `/map` returns station positions for the store map in the dashboard.
- `/pick` uses "shelf" (not "bin"). The agent navigates to the shelf first.
- `/place` target is "basket" (not "bag") or "return".
- `/stop` immediately halts all movement (emergency stop).
- One command at a time; busy robot returns `success: false, message: "busy"`.
- The "instruction" field in `/pick` is optional. Nemotron writes it for GR00T.

### Agent Route Planning
The agent groups order items by shelf and visits each shelf once (in order A, B, C).
At each shelf, it picks all items (heavy first) before moving to the next shelf.
This minimizes travel and puts heavy items at the bottom of the basket.

## Project structure
```
app/
  __init__.py
  config.py          # .env values, model names, load_inventory(), get_client()
  inventory.json     # product -> shelf, bin, barcode, price (INR), weight, stock
  order_parser.py    # order text -> {"items": [...], "unknown": [...]} (Nemotron Nano)
  robot_client.py    # HTTP calls to the robot server
  tools.py           # tool schemas for Nemotron + run_tool()
  agent.py           # agent loop: navigate, pick, scan, place, navigate
  test_agent.py      # tests for agent.py
  store.py           # SQLite helper for orders and stock
  test_store.py      # tests for store.py
  worker.py          # background worker that processes queued orders
  test_worker.py     # tests for worker.py (stock goes down after an order)
  Home.py            # Streamlit entry page (start worker, nav buttons)
  pages/
    1_Customer.py    # Customer kiosk: product grid, cart, payment, order status
    2_Owner.py       # Owner dashboard: metrics, robot status, orders, stock, AI summary
  ui.py              # (legacy) standalone Streamlit UI for demo mode
data/
  orders.db          # SQLite database (auto-created)
robot_server/
  __init__.py
  mock_robot.py      # fake robot for testing (same contract v2)
  jetson_server.py   # real robot server for Jetson with Nav2 + stubs
  stations.example.yaml # station positions example (copy to stations.yaml)
  test_stations.py   # tests for loading stations.yaml
  README.md          # instructions for hardware team
.env                 # NEBIUS_API_KEY, ROBOT_URL (never commit)
```

## Nebius Token Factory
- OpenAI-compatible API, base URL `https://api.tokenfactory.nebius.com/v1/`, key in `NEBIUS_API_KEY`.
- Models available on our account:
  - `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B` — fast, simple jobs (order parsing)
  - `nvidia/nemotron-3-super-120b-a12b` — main agent brain (tool calling)
  - `nvidia/Nemotron-3-Ultra-550b-a55b` — strongest, use rarely (expensive)
  - `nvidia/Nemotron-3_5-Lightning`
- Model replies may contain `<think>...</think>`; strip it before parsing.
- Hackathon rule: the project MUST use Token Factory + an NVIDIA open model. Do not add other LLM providers.

## How to run
```bash
source venv/bin/activate

# Start the mock robot server:
python robot_server/mock_robot.py                    # terminal 1, runs on :8000

# Start the Streamlit multipage app:
streamlit run app/Home.py                            # terminal 2, open browser

# Pages:
#   - Home: choose Customer Kiosk or Owner Dashboard
#   - Customer Kiosk: place orders (touch-friendly for Raspberry Pi)
#   - Owner Dashboard: monitor orders, robot, stock, AI summary

# CLI tools for testing:
python -m app.order_parser "2 soup cans and one chips"
python -m app.agent "2 soup cans and one chips"
python -m app.test_agent
python -m app.test_store
python -m app.test_worker
python -m app.worker                                 # test the background worker
```
Always run modules with `python -m app.xxx` from the project root.

## Coding rules
- Python 3.10 compatible. Keep code simple and readable; small functions.
- Code, comments and output text in **simple English** (the lead's English is beginner level).
- Never crash on robot errors: return `{"success": false, "message": ...}` and let the agent decide.
- Never hardcode the API key. Never print it.
- Test every new file against the mock robot before saying it is done.
- Work ONE step at a time. After each step, stop, show how to test it, and wait.

## Roadmap
1. [done] Local setup, mock robot, order parser, robot tools
2. [done] Agent loop: Nemotron Super calls tools until the order is packed, then returns a bill
3. [done] Streamlit UI: order box, live step log, bill
4. [done] New architecture: SQLite store, background worker, mobile robot with navigation
   - 4.1 [done] app/store.py, app/worker.py, updated robot API, agent with navigation
   - 4.2 [done] Customer page (product grid, cart, fake payment, order status)
   - 4.3 [done] Owner dashboard (metrics, robot status, step log, stock, AI summary)
   - 4.4 [done] Contract v2: multiple shelves, route planning, pose/battery/stop, jetson_server.py
5. [next] Connect to the real Jetson robot (change ROBOT_URL)
6. GitHub repo, README, MIT license
7. Batch test (many random orders, success rate) — later on Nebius Serverless Jobs
