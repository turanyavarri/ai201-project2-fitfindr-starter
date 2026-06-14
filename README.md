# FitFindr 🛍️

A multi-tool AI agent that helps users find secondhand clothing and figure out how to wear it. FitFindr searches mock thrift listings, suggests outfit combinations based on your wardrobe, and generates a shareable fit card caption all from a single natural language query.

---

## Tool Inventory

### `search_listings(description, size, max_price)`
- **Purpose:** Searches the mock listings dataset for items matching the user's query, filtered by size and price, and ranked by keyword relevance.
- **Inputs:**
  - `description` (str): Natural language description of the item (e.g. "vintage graphic tee")
  - `size` (str | None): Size string to filter by, or None to skip size filtering
  - `max_price` (float | None): Maximum price inclusive, or None to skip price filtering
- **Output:** A list of matching listing dicts sorted by relevance score, each containing: `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand`, `platform`. Returns an empty list if nothing matches.

---

### `suggest_outfit(new_item, wardrobe)`
- **Purpose:** Uses an LLM to suggest 1–2 complete outfit combinations using the thrifted item and pieces from the user's existing wardrobe.
- **Inputs:**
  - `new_item` (dict): A listing dict returned by search_listings
  - `wardrobe` (dict): A dict with an `items` key containing a list of wardrobe item dicts. Each item has: `id`, `name`, `category`, `colors` (list), `style_tags` (list), `notes`
- **Output:** A non-empty string with outfit suggestions and styling advice. If the wardrobe is empty, returns general styling advice for the item instead.

---

### `create_fit_card(outfit, new_item)`
- **Purpose:** Generates a short, shareable Instagram/TikTok-style caption for the complete outfit.
- **Inputs:**
  - `outfit` (str): The outfit suggestion string returned by suggest_outfit
  - `new_item` (dict): The selected listing dict, used for price and platform details
- **Output:** A 2–4 sentence string in casual social media style mentioning the item, price, and platform. Returns a descriptive error string if outfit is empty rather than raising an exception.

---

## How the Planning Loop Works

The planning loop in `run_agent()` follows conditional logic, it does not call all three tools unconditionally:

1. **Parse the query** using regex to extract `max_price` (looks for `$` followed by a number) and `size` (looks for "size" followed by a size string). The full query is used as the description.
2. **Call `search_listings()`** with the parsed parameters. Store results in `session["search_results"]`.
3. **Check if results is empty.** If yes, store an error message in `session["error"]` and return the session early, `suggest_outfit` is never called with empty input.
4. **If results exist**, store `results[0]` in `session["selected_item"]` and call `suggest_outfit()` with the selected item and wardrobe.
5. Store the outfit suggestion in `session["outfit_suggestion"]` and call `create_fit_card()`.
6. Store the fit card in `session["fit_card"]` and return the completed session.

The key branch is step 3, the agent's behavior changes meaningfully based on what `search_listings` returns.

---

## State Management

All state is tracked in a single session dict initialized at the start of `run_agent()`:

| Key | What it stores | When it's set |
|-----|---------------|---------------|
| `session["query"]` | Original user query | Start of run |
| `session["parsed"]` | Extracted description, size, max_price | After query parsing |
| `session["search_results"]` | Full list of matching listings | After search_listings |
| `session["selected_item"]` | Top listing dict | After search succeeds |
| `session["wardrobe"]` | User's wardrobe dict | Start of run |
| `session["outfit_suggestion"]` | String from suggest_outfit | After suggest_outfit |
| `session["fit_card"]` | String from create_fit_card | After create_fit_card |
| `session["error"]` | Error message if something fails | When a tool fails |

Each tool receives its inputs directly from the session dict, no data is re-entered between steps. For example, `session["selected_item"]` set in step 4 is passed directly into `suggest_outfit()` in step 5, and `session["outfit_suggestion"]` set in step 5 is passed directly into `create_fit_card()` in step 6.

---

## Error Handling

### `search_listings`
**Failure mode:** No listings match the query.
**Strategy:** Returns an empty list without raising an exception. The planning loop checks for this and sets `session["error"]` to: "No listings found for your search. Try a broader description, a different size, or a higher price limit." The agent returns early and never calls `suggest_outfit`.

**Concrete example from testing:** python -c "from tools import search_listings; print(search_listings('designer ballgown', size='XXS', max_price=5))"

**Output:** []
Running the full agent with this query returned the error message in the UI's top listing panel with the other two panels empty.

---

### `suggest_outfit`
**Failure mode:** The user has no wardrobe items entered.
**Strategy:** Detects an empty `wardrobe['items']` list and switches to a general styling prompt instead of a wardrobe-specific one. Still returns a useful string.

**Concrete example from testing:** python -c "
from tools import search_listings, suggest_outfit
from utils.data_loader import get_empty_wardrobe
results = search_listings('vintage graphic tee', size=None, max_price=50)
print(suggest_outfit(results[0], get_empty_wardrobe()))
"

**Output:** General styling advice for the Y2K Baby Tee without referencing wardrobe pieces

---

### `create_fit_card`
**Failure mode:** The outfit string passed in is empty or whitespace-only.
**Strategy:** Guards at the top of the function and returns a descriptive error string rather than calling the LLM or raising an exception.

**Concrete example from testing:**
python -c "
from tools import search_listings, create_fit_card
results = search_listings('vintage graphic tee', size=None, max_price=50)
print(create_fit_card('', results[0]))
"

**Output:** "Could not generate a fit card — outfit description was missing."

---

## Spec Reflection

**One way the spec helped:** Writing out the planning loop logic in `planning.md` before coding made the implementation of `run_agent()` straightforward. Having the exact branch condition ("if results is empty → set error, return early") already written out meant there was no ambiguity when it came to coding it, the spec translated almost directly into code.

**One way implementation diverged:** The spec described query parsing as a step to "extract description, size, and max_price" but didn't specify how. During implementation, I used regex to extract price (looking for `$` followed by digits) and size (looking for "size" followed by a size string), and used the full query string as the description for keyword scoring. This worked well but wasn't detailed in the original spec, it was a decision made during implementation.

---

## AI Usage

### Instance 1 — Implementing `search_listings`
I gave Claude the Tool 1 spec from `planning.md` (inputs with types, return value description, failure mode) and the instruction to use `load_listings()` from `data_loader.py`. Claude produced a function that loaded listings, filtered by price and size, scored by keyword overlap, and returned a sorted list. Before using it, I verified it filtered by all three parameters and returned an empty list (not an exception) when nothing matched. I tested it with three queries, "vintage graphic tee under $30", "designer ballgown size XXS under $5", and "jacket", before moving on.

### Instance 2 — Implementing `run_agent`
I gave Claude the Planning Loop section and Architecture diagram from `planning.md` and asked it to implement `run_agent()` in `agent.py`. Claude produced a function with the correct session dict structure and branching logic. I reviewed it before running to verify it checked for empty results before calling `suggest_outfit`, stored values in the session dict at each step, and didn't call all three tools unconditionally. I then tested both the happy path and the no-results path using the CLI test cases already in `agent.py`.

