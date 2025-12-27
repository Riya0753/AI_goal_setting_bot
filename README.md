#### [NOTE: This public repository only explains the code, draws a clear insight about the architecture, and mentions output screenshots/video.]

***

## AI‑Enhanced Telegram Goal Manager Bot

This project is a Telegram bot that helps you manage goals with AI assistance. It uses Google Gemini `gemini-2.5-flash` to refine user goals, assign priorities automatically, and generate short advice responses. The bot runs fully asynchronously, stores data in a lightweight JSON file, and keeps itself online using a small Flask keep‑alive server.

***

## Architecture Diagram

The following mind map visualizes the system architecture and component relationships:

- `BOT-Mind-Map.jpg`
<img width="2981" height="4937" alt="BOT Mind Map" src="https://github.com/user-attachments/assets/e838f83e-715d-471f-9a13-f5c01806a848" />

It shows how:

- Configuration (environment variables and JSON storage),  
- AI features (goal analysis, priority assignment, advice generation),  
- Core commands,  
- Technical components (Flask keep‑alive, asynchronous task handling, JSON persistence), and  
- Goal properties (ID, refined text, priority, status)  

work together inside the AI‑enhanced Telegram Goal Manager Bot.

***

## How the Code Works

### 1. Configuration and Environment

The bot reads configuration from environment variables:

- `TELEGRAM_TOKEN` – Telegram Bot API token  
- `GEMINI_API_KEY` – Google Gemini API key  
- `DATA_FILE` – Path to the JSON database file (default: `user_data.json`)

Priority levels are mapped both numerically and visually:

- `PRIORITY_MAP = {"high": 3, "medium": 2, "low": 1}`  
- `EMOJI_MAP = {"high": "🔴", "medium": "🟡", "low": "🟢"}`  

The Gemini client is configured using:

```python
genai.configure(api_key=GEMINI_API_KEY)
model = genai.GenerativeModel("gemini-2.5-flash")
```

If the API key is missing or configuration fails, the bot gracefully disables AI features and returns a friendly warning message.  

***

### 2. Keep‑Alive Flask Server

A minimal Flask app is embedded to keep the bot instance alive on platforms like Render:

- `app_web = Flask(__name__)` exposes a root `/` route returning `"Bot is running!"`.  
- `run_web()` starts Flask on `host='0.0.0.0'` and on the platform’s assigned `PORT`.  
- `keep_alive()` runs Flask in a separate `Thread`, so the Telegram polling loop can run in parallel.

This pattern allows external uptime monitors (like UptimeRobot) or the hosting provider itself to ping the HTTP endpoint and avoid the process being put to sleep.

---

### 3. Persistence Layer (JSON “Database”)

User data is stored in a simple JSON file defined by `DATA_FILE`:

- `load_data()`  
  - Safely reads the JSON file.  
  - On first run or parse errors, returns a default structure:  
    ```json
    { "goals": {}, "quotes": [] }
    ```
- `save_data(data)`  
  - Writes the in‑memory data back to disk with indentation for readability.  

Goals are kept in a dictionary keyed by a short UUID, e.g.:

```json
"AB12": {
  "task": "Refined, AI‑rewritten goal text",
  "status": "Pending",
  "priority": "high"
}
```

This makes it easy to update the status or priority of a single goal without scanning large arrays.

***

### 4. AI Utilities (Gemini Integration)

Two core helper functions integrate Google Gemini:

#### `ask_gemini(prompt)`

- Sends a free‑form prompt to Gemini using `model.generate_content(prompt)`.  
- Returns the response text or a handled error message.  
- Used by the `/advice` command to generate short, topic‑based tips.

#### `analyze_goal_with_ai(user_text)`

- Accepts the raw goal the user typed.  
- Builds a strict instruction prompt:

  > Analyze this goal, rewrite it in max 10 words, and assign a priority (High/Medium/Low).  
  > STRICT Output format: `Refined Text | Priority`.

- Calls `model.generate_content(prompt)` and parses the response by splitting on `"|"`.  
- Returns:
  - `refined_task`: concise, AI‑rewritten version of the goal.  
  - `priority`: normalized to lowercase (`"high"`, `"medium"`, or `"low"`).  

If AI is unavailable or parsing fails, the function falls back to the original goal text and a `"medium"` priority, ensuring the bot always remains usable.

---

### 5. Telegram Bot Commands

All commands are implemented as `async` handlers and registered via `python-telegram-bot`’s `ApplicationBuilder`.

#### `/start`

- Sends a quick help message listing all available commands:
  - `/add`, `/list`, `/done`, `/advice`, `/quote`, `/savequote`.

#### `/add [goal]` – Add a Smart Goal

Flow:

1. Extracts the raw goal text from `context.args`.  
2. Sends a provisional message: `"🧠 Analyzing '…'..."`.  
3. Calls `analyze_goal_with_ai()` to:
   - Refine the goal text.
   - Auto‑assign a priority (High/Medium/Low).  
4. Generates a short `goal_id` using a truncated UUID.  
5. Loads existing data with `load_data()`.  
   - If the stored format is an old list, converts it to a dict.  
6. Saves the new goal with fields:
   - `"task"`, `"status": "Pending"`, `"priority"`.  
7. Edits the provisional message to show:
   - The generated ID,
   - Priority icon,
   - Refined goal text.

Example final message:

> 🔴 **Added (ID: ab12):**  
> Schedule annual health checkup  
> Priority: High  

#### `/list` – View Goals

- Loads all goals from the JSON file.  
- Handles empty or legacy formats safely.  
- Sorts goals using a two‑level key:
  1. Status: `Pending` goals first, then `Done`.  
  2. Priority: High → Medium → Low, via `PRIORITY_MAP`.  
- Renders each row as:

  ```text
  `ID` 🔴 Task text
  ```

  with strikethrough applied for completed (`Done`) tasks.

This provides a quick, priority‑oriented snapshot of a user’s workload.

#### `/done [ID]` – Mark a Goal Complete

- Reads the target `goal_id` from arguments.  
- If found in the data store:
  - Updates `status` to `"Done"`.  
  - Saves the updated JSON.  
  - Confirms with a celebratory message.  
- If not found:
  - Responds with an error and suggests checking `/list`.

#### `/advice [topic]` – Get AI Advice

- Accepts any short topic (e.g., “time management”, “fitness plan”).  
- Sends a temporary `"🤔 Thinking..."` message.  
- Calls `ask_gemini()` with a prompt asking for a short response and explanation.  
- Edits the original message to show formatted advice under a `💡` header.

#### `/savequote [text]` and `/quote`

- `/savequote` appends a new quote string to the `quotes` list in `user_data.json`.  
- `/quote` randomly selects and returns one saved quote, or informs the user if no quotes exist yet.

***

### 6. Main Execution Flow

The `if __name__ == '__main__':` section coordinates everything:

1. **Start Keep‑Alive Server**  
   - Calls `keep_alive()` to spin up the Flask app on a background thread.

2. **Validate Telegram Token**  
   - If `TELEGRAM_TOKEN` is missing, logs an error and skips bot startup.  

3. **Build and Run Telegram Bot**  
   - Creates the application with `ApplicationBuilder().token(TELEGRAM_TOKEN).build()`.  
   - Registers all command handlers:
     - `start`, `add`, `list`, `done`, `advice`, `quote`, `savequote`.  
   - Starts long‑polling via `app.run_polling()`, which listens for user commands and dispatches them to the async handlers.

This single file thus runs two cooperating processes: a lightweight HTTP server and the Telegram bot’s polling loop.

***

## Output Screenshots / Demo

For clarity and quick understanding, the repository includes:

- Screenshots of key interactions:
  - `/start` help message.
  - `/add` with AI‑refined goal text and assigned priority.  
  - `/list` showing sorted goals with priority icons and completed tasks.  
  - `/advice` returning AI‑generated tips.
 <img width="1330" height="923" alt="image" src="https://github.com/user-attachments/assets/0b6afaea-aaa6-41e1-8ea6-a4f9f870e1c7" />

- A short demo video:
  - Showing a user adding goals, viewing them in sorted order, completing one, requesting advice and saving a quote.
