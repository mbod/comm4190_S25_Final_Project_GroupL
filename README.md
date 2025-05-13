## COMM4190 (Spring 2025) Final Project

### Project Group L

----

# Image Recognition for Receipt Scanning and a Digital Pantry Assistance App

## Introduction  
**Pang** is your personal _Digital Pantry Assistance App_—a two-part system that  
1. **Scans** grocery-receipt images (using GPT-powered OCR + context reasoning)  
2. **Turns** them into a structured, queryable “digital pantry” you can ask questions of via a simple web UI.

---

## Project Overview  
- **Receipt Scanner**: A Python pipeline that  
  1. Loads and base64-encodes a receipt image  
  2. Sends it to GPT-4o-mini with a structured prompt  
  3. Parses the raw text tuples `(item, quantity=1, expiration, category, location, cost)`  
  4. Consolidates duplicates into quantities  
  5. Dumps the result to JSON  

- **Pantry Database & UI**:  
  1. Loads JSON into a local SQLite DB  
  2. Exposes `ask_pantry(question)` that  
     - Pulls all items  
     - Builds a new GPT prompt with the data + your question  
     - Returns a natural-language answer  
  3. Wraps `ask_pantry` in a Gradio app so you can ask things like “What snacks do I have?”  

---

## File Structure & Descriptions  

├── main.ipynb # Your core receipt-scanning notebook, contains: encode_image, prompt & API call, parsing, consolidation
├── gradio_interface.ipynb # DB setup + Gradio UI notebook
├── readme.md # An abundance of information (all the admin work about project idea, user scenarios, prompt development, app development, testing, etc.)
├── pang.png: # The Pang logo/icon that I designed
├── pantry.db: # sqlite3 db to hold json data for Gradio setup
└── receipt_output.json/ # the output for latest call in main
├── costco_test.png
├── target_test.png
├── notes_test.jng
├── cat_test.jpg
└── human_test.jpg


**test_receipts notes**  
- **human_test.jpg**  
  - High-resolution photo → very slow GPT ingestion  
  - Consider adding an image-size check or early process kill
  - Slowly returns `INVALID` → 🎉 correct behavior for non-receipt  
- **cat_test.jpg**
  - Cat photo!
  - Quickly returns `INVALID` → 🎉 correct behavior for non-receipt
- **notes_test.jpg**  
  - Picture of scribbled notes, text, though not receipt.
  - Quickly returns `INVALID` → 🎉 correct behavior for non-receipt  
- **costco_test.png**  
  - Parses all line items correctly  
  - Consolidation now working as expected  
- **target_test.png**  
  - Minor consolidation hiccups (some savings/non-food lines mis-grouped)  
  - UX mitigation: allow user to mark lines as “not groceries” in the UI  

---

## 5 User Scenarios  

Below are five common **Grocery Workflows** and how Pang handles them:

1. **Busy Parent Tracking Expiration**  
   1. Snap a photo of today’s grocery receipt.  
   2. Run the Receipt Scanner in `main.ipynb`.  
   3. Scan → raw tuples → consolidate → `receipt_output.json`.  
   4. Launch the Gradio UI and ask “What’s expiring this week?”  
   5. Pang returns a list of soon-to-expire items, letting you plan meals.

2. **Meal-Prep Enthusiast Budgeting**  
   1. Load last week’s receipts into the scanner.  
   2. Consolidated JSON shows itemized costs.  
   3. Ask the UI “How much did I spend on produce?”  
   4. Pang sums all items with category `F` or `V` and returns the total.

3. **Quick Inventory Check Before Shopping**  
   1. Review digital pantry via `ask_pantry("What snacks do I have?")`.  
   2. Get a bullet list of all category `S` items.  
   3. Know what to add to your shopping list—no surprises.

4. **Diet Tracking**  
   1. Run scanner on weekly receipts.  
   2. Ask “List all dairy items I bought this month.”  
   3. Pang enumerates category `D` with quantities—easy macro tracking.

5. **Receipt-Based Food Donation Prep**  
   1. Scan receipts from multiple trips.  
   2. Ask “Which items do I have duplicates of?”  
   3. Pang returns any item with quantity > 1 so you can bundle extras for donation.

---

## Prompt Development  

We evolved our GPT prompt in three stages:

1. **Original Prompt**  
   Today's date is {current_date}. You are given a receipt image.
   For each item:
     1. Guess the cleaned-up name.
     2. Provide the quantity.
     3. Guess an expiration date.
     4. Classify from {F, V, G, D, M, S, O}.
     5. Provide storage {P, F, Z}.
     6. Provide cost as a float.
   If not a valid receipt, return INVALID.
   Format strictly: {(item, qty, exp, cat, loc, cost), ...}

The goal behind this prompt is pretty straight forward.

2. **“Quantity = 1” Tweak**
* Changed step 2 to: '''Provide quantity **always as 1** for each line-item—do not combine duplicates. We’ll consolidate later.'''

3. **Templated Prompt**
* Templated Prompt
* Injected dynamic {category_keys} and {location_keys} from small dicts,
* Made the entire block easy to version, diff, and extend.

**Potential Gradio Prompt**
> You are a kitchen assistant.  
> Here is my pantry data:
> [{ "item":"Milk", "quantity":2, … }, …]
> User question: {user_question}  
> Answer concisely based on the data above.

---

## Workflows

Below is the core function-call flow, with a high-level description of each step:

encode_image ▶ get_enhanced_receipt_items_gpt4o  
   (image → base64 → GPT prompt → raw text)

raw text ▶ parse_enhanced_gpt_output  
   (regex → list of dicts each with quantity = 1)

parsed ▶ consolidate_items  
   (group by item/expiration/category/location/cost → sum quantities)

consolidated ▶ write_json_to_file  
   ("receipt_output.json")

receipt_output.json ▶ Cell #1 (db_frontend.ipynb)  
   (load into SQLite table `items`)

ask_pantry(question) ▶ query_pantry_db  
   (SQL SELECT * → Python list of dicts)

items + question ▶ LLM prompt ▶ GPT answer

Gradio UI ▶ calls ask_pantry ▶ displays answer

---

## Evaluation & Testing

- **User Scenario Tests**  
  - Verified each of the 5 workflows end-to-end with sample receipts.

- **Adversarial (“Invalid”) Tests**  
  - **cat_test.jpg** → quickly returns `INVALID`.  
  - **notes_test.jpg** → quickly returns `INVALID`.  
  - **human_test.jpg** → slow, but eventually returns `INVALID`.  
    - *Future*: add preflight image-size check or timeout.

- **Edge Cases**  
  - Receipts with coupons/savings lines → currently treated as costed items.  
    - UI could let you deselect non-food lines.  
  - Multi-page receipts → future work: stitch multiple images together before scanning.

---

## UI

We use **Gradio** for a lightweight front end. I have access to a local app for iOS devices that implements my functionality, feel free to email me if you'd like to a seen a screen recording of that.

- **Input**: A single textbox for your natural-language question.  
- **Backend**: Calls `ask_pantry(question)` under the hood.  
- **Output**: GPT’s concise, data-backed response.  
- **Launch**: In the gradio_interface.ipynb file's last code cell.




