# PaintProxy Technical Specification

**Figure 1:** Example of a hand-painted Magic: The Gathering proxy card. PaintProxy.com will help users find low-cost cards to create similar playful proxies, while keeping the app's interface clean and modern.

## Overview and Goals

PaintProxy is a web application that helps Magic: The Gathering players find inexpensive "proxy" cards to stand in for expensive ones in their deck. The user pastes a plaintext decklist (e.g. copied from Moxfield or another deck builder), and the app suggests low-cost alternative cards with similar characteristics that the user can paint or use as proxies. There is no login or account system – the experience is paste-and-go. The app's aesthetic should evoke the fun, homemade vibe of painted proxies (as in Figure 1) but with a clean, modern UI reminiscent of Vercel or ChatGPT. It will be built with Next.js (React) for the front end, styled with Tailwind CSS and Shadcn UI components for a polished look. The backend will rely exclusively on the Scryfall API for card data and prices.

## Key Features and Requirements

- **Decklist Input:** Users can input a plaintext decklist (e.g. copy-pasted text). The app will parse card names and quantities from the text. No file uploads or complex input – just a text area for pasting the list. No authentication is required; users access the tool freely without accounts.

- **Scryfall-Powered Search:** For each card in the decklist, the app finds potential proxy substitutes using the Scryfall API. The search is based on relaxed matching criteria:
  - **Mana Cost Similarity:** Recommend cards with the same or very close mana value (converted mana cost) to the original.
  - **Type Match:** Ensure the suggested proxies share the primary card type (creature, artifact, enchantment, sorcery, etc.) with the original card.
  - **Color Identity for Lands:** If the original card is a land, determine its color identity (what colors of mana it can produce) and find lands that produce similar colors. Using Scryfall's color identity filters is especially useful for lands (e.g. `identity:ur type:land` finds all lands usable in a blue-red deck)[1].
  - **Relaxed Text Match:** By default the search does not require matching card text, but an optional "Strict Text Match" filter can be enabled to require that proxy suggestions contain certain keywords from the original card's rules text (e.g. "flying", "draw", etc.) for closer functional similarity.

- **Filter Controls:** Users can refine the suggestions with several filters:
  - **Maximum Price:** A slider or input to set the maximum price for proxy suggestions (default $5 USD or less). The app filters out any suggestions above this price. For example, a query can find only cards under $5 by including `usd<5` in the Scryfall search[2].
  - **Card Type/Rarity Restriction:** Options to restrict suggestions to certain card types or rarities. For instance, a user might choose to only see proxy suggestions that are common/uncommon to ensure low cost (Scryfall allows filtering by rarity, e.g. `rarity<rare` finds commons and uncommons[2]). By default, the proxy search already matches the original card type, but this filter could let users broaden or narrow results (e.g. include only creatures, or exclude lands, etc., depending on their preferences).
  - **Stricter Text Matching:** A toggle to enforce the optional text keyword match described above. When off, suggestions only match by type/cost (broader results); when on, the search query includes key terms from the original card's text (`oracle:` filters on Scryfall) to find more functionally similar cards.

- **Batch Search Efficiency:** The app will utilize Scryfall's API efficiently by batching requests. On input submission, the app will fetch data for all original deck cards in as few calls as possible (using Scryfall's `/cards/collection` endpoint, which allows up to 75 cards per request[3]). Then, for each original card, it will perform a Scryfall search query for proxies (applying the criteria and filters). The searches can be done in parallel up to Scryfall's rate limit (Scryfall asks clients to stay around 10 requests per second)[4] to keep the app responsive without overloading the API.

- **Proxy Suggestions Display:** For each card in the input deck, the UI will display a set of suggested proxy replacements:
  - The original card's name (and possibly a small image or icon) will be shown for reference.
  - A horizontal carousel of proxy options will be displayed, each showing the candidate card's image, name, set, rarity, and price. Users can scroll within the carousel to view all suggestions.
  - Each proxy option will show its price (using Scryfall's USD price for the cheapest printing) and the set/edition (since the cheapest version often comes from a specific set). For clarity, the app may label each option with the set code or name (e.g., "Dominaria (DOM) – $2.50") under the card image.
  - The carousel allows a quick side-by-side comparison of options. The design will use Shadcn/UI Carousel or Tabs components for smooth interaction.

- **Proxy Selection:** Users can click or tap on one of the suggested proxy cards to select it as the replacement for the original. Selection can be indicated with a highlight or checkmark on the chosen card. Only one proxy per original card can be selected. (If the decklist had multiple copies of a card, selecting a proxy implies using that same proxy card for all copies of that card in the deck.)

- **Summary View:** Once proxies are selected for all (or some) cards, the app provides a summary panel:
  - A list or table of all chosen proxy cards, including each card's name, chosen printing set, and individual price. If a card was in the deck multiple times, the summary may indicate the quantity (e.g. "4× Goblin Raider (M20) – $0.10 each").
  - The total price of all selected proxy cards (sum of prices times quantities).
  - The total estimated price of the original decklist. This is computed by summing the price of each original card (for the same quantity) using Scryfall's price data. We will use each original card's lowest available printing price for this calculation by default.
  - The total savings = original deck cost – proxy deck cost, showing how much money the user would save by proxying those cards.
  - The summary should be presented in a clear, easy-to-read format (possibly as a table or list with bold totals).
  - An "Export" feature will output the selected proxies as a plaintext list, with each card's name and set code, suitable for copying. For example, the export might list:
    ```
    4 Goblin Raider (M20)
    2 Evolving Wilds (M21)
    ...
    ```
    This contains the card names and set identifiers for easy reference. No images or PDFs are generated – it's a simple text output by design.

- **Modern UI Design:** The interface will be clean and minimalistic:
  - Uses Tailwind CSS for utility-first responsive styling.
  - Employs Shadcn/UI components for polished, accessible UI elements (buttons, inputs, carousels, toggles, etc.). Shadcn provides pre-built React components styled with Tailwind, ensuring a consistent modern look[5].
  - The overall layout is two-column on desktop: the left side for input and filters, and the right side for results and summary. On mobile, the layout will stack vertically (input section on top, results below) for a responsive experience.
  - The design will use plenty of whitespace and clear typography (e.g., a clean sans-serif font) similar to Vercel's or ChatGPT's interface. Color scheme will likely be neutral or light theme with a dark text, possibly with a bright accent color (for buttons or highlights) to add a touch of playfulness (inspired by colorful paint markers) without cluttering the UI.
  - No heavy graphics or ornate backgrounds – keep it simple to let users focus on the card results. Any decorative elements should be subtle (for example, a faint textured background or proxy-themed illustration could be used in empty states, but nothing overpowering).

## Architecture Overview

### Tech Stack

The app is built with Next.js (a React framework). It will use Next.js 13+ App Router for building the UI as a single-page application experience. This provides server-side rendering for initial load and client-side interactivity for the deck processing. Tailwind CSS is used for styling, configured with a custom theme if needed. Shadcn/UI (built on Radix UI + Tailwind) will be integrated for ready-made UI components, which are copied into the project for full control[5]. The app is entirely front-end oriented aside from calling external APIs; no database or persistent server storage is required (all data is fetched live from Scryfall and stored in memory/state).

### Data Flow

The processing of a deck goes through these steps:

1. **Input Parsing:** When the user submits a decklist (by clicking a "Find Proxies" button or similar), the app parses the input text. This parsing logic will read each line, extract the quantity (if present) and card name. It will ignore any unrecognized lines (e.g. section headers like "Sideboard:" or comments). For example, a line like "4 Lightning Bolt" would be parsed as Lightning Bolt ×4. If no quantity is given, it's assumed to be 1. The result of parsing is an array of deck entries, e.g. `[{ name: "Lightning Bolt", quantity: 4 }, ...]`. This array is stored in state for further processing.

2. **Original Card Data Fetch:** The app will retrieve details for each unique card in the deck from Scryfall. To minimize API calls, it will use the `/cards/collection` endpoint by sending up to 75 card identifiers in one request[3]. If the deck has more than 75 unique cards (rare, but possible in large Commander decks), multiple batch requests will be made. The collection query will return data for each card, including fields like mana cost (CMC), type line, oracle text, and prices. The app will extract the relevant info for proxy searching:
   - Mana Value (converted mana cost).
   - Card Type (supertype and subtype if needed, but primarily we care if it's Creature, Sorcery, Land, etc.).
   - Oracle Text (to identify keywords or color identity).
   - Color identity (from the card's data – Scryfall provides a `color_identity` array for each card).
   - Price (the `usd` price of the card's cheapest printing or the printing returned – this is used for calculating original deck cost).

3. **Proxy Search Queries:** For each original card, the app constructs a Scryfall search query string based on the criteria and user-selected filters:
   - **General structure:** The query will include `type:` filter matching the card's type, mana value (or `cmc`) filter matching the card's mana cost (or a range around it if "similar" allows flexibility), and a price filter `usd<=X` (X being the max price from user filter, default 5). For example, if the original card is a creature with mana value 4, the query might be: `t:creature manavalue=4 usd<=5`.
   - **Color considerations:** If it's not a land, we may include a color identity filter to keep suggestions in the same color or colors. For instance, if the original card has color identity {U}{R} (blue/red), adding `identity:ur` will restrict results to cards that could be used in a blue-red deck (i.e., cards that don't have colors outside U/R)[6][7]. This ensures we don't suggest off-color proxies that wouldn't fit in the deck.
   - **Land specific:** If the original is a land, instead of mana cost, we focus on color identity and land type. For example, an original land that produces green and white mana (identity GW) will be proxied by searching `t:land identity:gw usd<=5`. This finds lands that can be used in a green-white deck, i.e., lands that produce green/white mana, which is exactly what we need for functional land proxies[1].
   - **Rarity filter:** If the user has restricted rarity (say only want commons & uncommons), add `rarity<=uncommon` to the query. Scryfall supports rarity filtering and ordering; rarities have an order so comparisons work (common < uncommon < rare < mythic)[2].
   - **Oracle text filter:** If "Strict Text Match" is enabled and the original card's oracle text contains a distinctive keyword or mechanic, include an `o:keyword` term in the query to require that keyword. For example, if the original card text has "Flying", the query can include `o:flying`. We will choose one or two relevant terms (too many could over-constrain the search). This filter will narrow results to cards that likely have similar abilities.
   - **Sorting and uniqueness:** To get the cheapest viable suggestions, the query will sort results by price ascending. Scryfall allows sorting by USD price (`order:usd`), which will list cheapest cards first. When sorting by price without specifying a particular printing, Scryfall's engine returns the cheapest printing of each card[2]. We can also use `unique:cards` to avoid duplicate printings in results. This way, each suggested card in our results is a distinct card (with its cheapest printing's info).
   - **Result capping:** We will retrieve, say, the top 5–10 results for each query. This is accomplished by using Scryfall's page size or simply fetching one page (which by default returns up to 175 cards, but we will not display that many). We'll take the first few results from each to show in the carousel.

**Example:** Original card "Lightning Bolt" (a Red instant with mana value 1, price ~$2). The search query (assuming default filters) might be:

```javascript
q = "t:instant manavalue=1 identity:r usd<=5"
```

This would find cheap red instants costing 1 mana under $5. Scryfall might return options like "Shock" (cost 1R in text but 1 mana total, effectively similar), "Burst Lightning", etc., each with price info. The app then takes the top suggestions and prepares them for display.

4. **API Rate Management:** All Scryfall queries will be performed with respect to their rate limit guideline (roughly 10 requests per second)[4]. The app should batch and throttle requests accordingly:
   - The initial `/collection` request for original cards is just 1-2 calls (not an issue).
   - The search queries for proxies could be numerous (one per card). We will implement a queue or promise chain to ensure we don't exceed ~10 concurrent calls per second. For example, if the deck has 20 unique cards, we can fire them in batches of 5 with a short delay between batches, or use a small concurrency limit in code.
   - This can be done on the server side (via Next.js API routes or server-side functions) to avoid blocking the UI and to secure the API calls. Alternatively, calls can be from the client since Scryfall supports CORS; in that case, we still throttle in the client code. For simplicity and performance, a serverless API route (e.g. `pages/api/proxies` or an app route handler) could accept the list of parsed cards and perform all Scryfall fetches, then return the aggregated results.

5. **Response Handling:** Each Scryfall search response will be parsed for the needed data:
   - For each suggested card result, we gather its name, set code, image URL, and price. We'll use the image URI for a small card image (Scryfall typically provides a `image_uris.small` or similar for each card).
   - We might prefer the card's cheapest printing info. If using sorted results, the card object returned likely corresponds to the cheapest printing. If not, we can explicitly fetch the specific printing data (but this is likely unnecessary if the search result is already the cheapest version).
   - We ensure to exclude the original card itself if it appears in results. (If the original card is already under the price threshold, it might show up in the query results for itself – we should filter that out so we're only showing different cards as proxies.)
   - All proxy suggestions for all cards are compiled into a structured format (e.g., a dictionary mapping original card name -> list of proxy options).

6. **Updating the UI:** Once the proxy suggestions data is ready, the app updates state to display the results on the right side of the interface. The user can then interact with the carousel suggestions and make selections. Each selection updates the state (linking which proxy was chosen for which original card).

7. **Summary Calculation:** The summary panel computes totals whenever selections change. The original deck cost is already known from step 2 (sum of original cards' prices times quantity). The proxies cost is calculated by summing the chosen proxy cards' prices times the original quantity of that card. The difference gives the savings. These calculations are straightforward and happen instantly in the client. The summary view is then rendered with this information.

8. **Export:** When the user requests the export, the app will generate a plain text block listing each selected proxy. This can be shown in a modal or separate section for easy copying. No server action is needed; it's generated on the fly from current state. We may provide a "Copy to Clipboard" button for convenience.

Throughout this flow, any errors from the API (e.g. a card not found, or network error) will be handled gracefully. For instance, if a particular card name is unrecognized by Scryfall, the app can skip it and possibly notify the user that one card couldn't be found. Similarly, if Scryfall returns too many results or times out, we might show a message to try again or refine filters.

## UI Components Breakdown

The application can be broken down into the following key components and UI sections:

- **DeckInput Area (Component: `DeckInputPanel`):** This is on the left side of the screen. It includes:
  - A heading or prompt like "Paste your decklist:".
  - A multiline text `<textarea>` for the user to paste the deck list. This will be a Shadcn/UI Textarea component with proper styling[8].
  - Optionally, a small help text or example format below the textarea (to guide how to format the deck list, e.g. "Use the format 'Quantity Card Name', one per line.").
  - A submit button, e.g. a Shadcn Button[8], labeled "Find Proxies" or similar. Clicking this triggers the parsing and search process. The button could be at the bottom of the left panel or fixed at top of results area – but likely directly under the text input for clarity.
  - This panel may also include an error message area (for input parsing errors or API errors) and possibly a loading indicator (e.g., a spinner) that appears once the search is in progress.

- **Filter Panel (Component: `FilterPanel`):** Also on the left side, probably below or above the text input (could be collapsed by default or always visible). Contains filter controls:
  - **Max Price filter:** could be a number input or slider. A Shadcn Slider component[9] can be used for a nice UI. The slider range might be $0–$50, defaulting at $5. We'll display the chosen value.
  - **Card Type filters:** possibly a dropdown multi-select or a group of checkboxes for each card type. For simplicity, a multi-select (using Shadcn Select or Checkbox components) can allow users to pick which types to allow. If none selected, default means no restriction beyond matching original type.
  - **Rarity filter:** similar to type – could be a multi-select or checkboxes (Common, Uncommon, Rare, Mythic). By default all are allowed; user could uncheck some to exclude, or a toggle "Only common/uncommon proxies" as a quick option.
  - **Text match toggle:** a simple on/off switch. Use Shadcn Switch component[10] with a label like "Strict text match (match keywords)".
  - These filters should be settable before running the search. If the user changes them after getting results, we may allow re-running the search with new filters. (In the UI, changing a filter could auto-requery or enable a "Update Results" button.)
  - The filter panel should be clearly separated from the deck input (e.g., grouped in a fieldset or card). It might be visually presented as an accordion or collapsible section to avoid clutter. For instance, a "Filters" collapsible panel (using Shadcn Accordion[11] or Collapsible) that can be expanded if the user wants to refine the search.

- **Results Panel (Component: `ResultsPanel`):** This occupies the right side of the screen on desktop (or below on mobile). It will display once the user has submitted a deck and the app has proxy suggestions. Key elements inside:
  - For each original card from the deck, a sub-section listing the suggestions. We can implement each as a ProxySuggestions Card.
  - **ProxySuggestions Card (Component: `ProxySuggestionsCard`):** This component represents one original card and its proxy options:
    - It might be styled as a card or list group. At the top, show the original card's name (and perhaps mana cost or a tiny image of it). Also show the quantity from the deck (e.g., "4× Lightning Bolt") and possibly the original's price (like "($2.00 each)" for context).
    - Below that, the horizontal carousel of proxy options. We can utilize Shadcn's Carousel component[12] if available, or implement a scrollable container (Tailwind can make a horizontally scrolling div).
    - Each item in the carousel is a ProxyOption (described next).
    - The carousel should have navigation controls (left/right arrows) or scrollbar for usability. On mobile, it will likely be scrollable by swipe.
    - If the list of proxy options is short (<=3), we could just show them side by side; if longer, carousel makes sense. We should ensure it's clear that it's scrollable (maybe partially show the next card or provide arrows).
    - This component also handles user selection: when a user clicks one of the proxy options, that selection state is updated (e.g., highlight the selected card). It may also immediately reflect in the summary.
    - Consider accessibility: ensure carousel and selections are keyboard-navigable (Shadcn's components and Radix should handle ARIA roles, focus, etc. automatically in many cases).
  - **ProxyOption (Component, could just be a reusable fragment):** Represents a single card suggestion in the carousel:
    - Likely implemented as a small card or simply an image with text. It might use a Shadcn Card component[13] for consistent styling.
    - Contains the card image (using Next.js `<Image>` or just an `<img>` tag with the Scryfall image URL). We might constrain the size (e.g., 100px width).
    - Beneath the image (or overlaying it on hover) display the card name and set code, and the price.
    - Example: an option might show the card art for "Shock", and below it "Shock – M10 – $0.15".
    - The whole option should be clickable/selectable. We can style the selected state (e.g., border and background highlight).
    - This component will receive props like `card {name, set, price, image}` and an `isSelected` flag.

- **Summary Panel (Component: `SummaryPanel`):** Shown after or alongside the list of suggestions. Placement could be:
  - At the bottom of the results column, after all ProxySuggestionCards.
  - Or as a fixed panel at the bottom of the screen that updates (but fixed might be tricky on mobile).
  - In either case, it should be easily noticeable once selections are made. We might include a "View Summary" button or auto-scroll to it when ready.
  - The summary includes:
    - Count of total proxy cards selected (and maybe how many original cards were proxied).
    - Total proxy cost and original cost and savings, clearly labeled. Possibly use a larger or bold font for these totals.
    - A line like "Original deck cost: $123.45" and "Proxy deck cost: $15.20" and "You save: $108.25 (approximate)".
    - The list of selected proxies, each on its own line or table row. For each selected proxy: "{Quantity} {Card Name} ({Set Code}) – ${price each}". If quantity >1, show total for that line as well (or just the each price since total will be reflected in overall sum).
    - We can format this as a small table with columns for quantity, name (with set), each price, and line total. Or a simple list with inline info.
    - An export button or link: clicking it could open a modal or dropdown that contains the plaintext list ready to copy. We might also allow one-click copying to clipboard (using a small script or library).

- **Export Modal/Dialog (Component: `ExportDialog`):** When triggered, shows a text area or a preformatted text block that the user can copy. Alternatively, it could instantly copy to clipboard and show a toast notification using Shadcn's Toast component[14][10]. To keep it simple, we might have a modal with the text. This modal can reuse Shadcn Dialog or AlertDialog for a styled popup[11].

- **Loading and Feedback UI:** Various states should be conveyed to the user:
  - While searches are in progress, show a spinner or loading indicator. Possibly overlay the results panel with a "Loading proxies…" message or use a Shadcn Spinner component[15] in each card section that's waiting for data.
  - If no suggestions are found for a particular card (which can happen if the original is already very cheap or filters are too strict), show a message in that card's section like "No cheap alternatives found under the current filters."
  - If an error occurred (API failure), display a non-intrusive error message (perhaps a toast notification or an error banner in the results area).
  - After initial load, the results panel might display a placeholder illustration or text ("Paste a deck list to see proxy suggestions") to guide the user.

- **Responsive Behavior:** Use Tailwind's responsive classes to adjust layout:
  - On small screens, the DeckInputPanel and ResultsPanel stack vertically. Likely the input (and filters) remain at top, and results below.
  - The carousel components should scroll by touch drag on mobile.
  - Long card names or multiple filters might need wrapping or collapsing to fit narrow screens.
  - Use relative units and flexbox/grid to ensure the two panels split appropriately on larger screens (e.g., left panel maybe 35% width, right 65%, or any comfortable ratio).

All components will be implemented in TypeScript and JSX. The styling will rely on Tailwind classes (augmented by Shadcn UI component classes). We will keep components small and focused (e.g., separate out the ProxyOption card to simplify the carousel code).

## Scryfall API Integration Strategy

Interfacing with Scryfall is central to the app's functionality. Below is how we will handle API usage:

- **Card Data Retrieval:** Using the Scryfall `/cards/collection` endpoint to fetch original card details in bulk[3]:
  - We will send a POST request with a JSON body like `{"identifiers": [ { "name": "Card Name 1" }, { "name": "Card Name 2" }, ... ]}`. We'll ensure to use exact card names from the deck list for accuracy. (If needed, we could use the fuzzy search in `/cards/named` for misspelled names, but that would be one-by-one; ideally deck exports have correct names.)
  - The response will contain an array of card objects in the same order. We will map these to our internal deck list entries by name. We have to be mindful that some cards have multiple printings or versions; the collection API usually returns a default printing. We will use the data primarily for mana cost, type, and oracle text. The `prices.usd` field from each object gives a price – we'll treat that as the approximate price for that card's cheapest printing (though it may actually be price of the returned printing).
  - If a card is not found, Scryfall returns an error or an `not_found` entry. Our logic will catch this and possibly remove that card from consideration (and notify the user).
  - This call is made once per deck submission. Caching: If the same deck is submitted again, or if user modifies the deck slightly, we might cache previous results in memory to avoid re-fetching card details, but for an MVP, caching is optional.

- **Search Queries for Proxies:** We will construct dynamic queries as described in the earlier section. To call Scryfall's search API:
  - We perform GET requests to the endpoint `/cards/search?q=<query>` (URL-encoded query string). We will include parameters for sorting and uniqueness, e.g. `&order=usd&dir=asc&unique=cards` to sort by price ascending and get unique cards.
  - Example query URL (encoded): `https://api.scryfall.com/cards/search?q=t%3Aland+identity%3Aug+usd%3C%3D5&order=usd&dir=asc&unique=cards`
  - The response comes paginated if results are large. By default it includes up to 175 cards and a `has_more` flag with a next page URL if there are more. We don't need to fetch more than the first page since we only display a few top suggestions.
  - We will parse the JSON response (`data` field is an array of card objects).
  - From each card object, gather:
    - `name`
    - `set_name` or set code (the code like "M20")
    - `prices.usd` (the price in USD; if null, it might mean no price available, perhaps skip those or treat as 0 if it's a free card like basic land).
    - `image_uris.small` (a small image link) or if card is double-faced, use `card_faces[0].image_uris.small`.
    - We might also check rarity if needed to filter by rarity on our side (though if we included it in query, it's already filtered).
  - We will likely limit ourselves to the first 5 results in `data`. If none in `data`, then as mentioned, mark "no suggestions".
  - These queries will be executed for each original card concurrently up to our throttle limit. Implementation-wise, using `Promise.all` with a concurrency cap or a task queue system (since Next.js might not have a built-in queue, we can use a small utility or just manually throttle with `setTimeout`).
  - All search requests and the collection request will include a small User-Agent identifier as recommended by Scryfall API policy (e.g., identifying the app name).

- **Rate Limiting & Error Handling:** As noted, we respect ~10 requests/second[4]. Given a typical 60-card deck with maybe ~40 unique cards, our plan of batching 5-10 at a time will stay within that. We will implement retry logic for transient failures: if Scryfall returns a 503 or rate-limit error, we can wait a moment and retry that specific query. Any hard errors (404, etc.) we handle gracefully per card.
  - In case Scryfall is completely unreachable (network down), the app will show an error banner and not crash.
  - The app does not store any Scryfall data long-term, so no need to manage data updates – every query is live. (Scryfall's data is updated daily on their end.)

- **Bulk Data (Optional Optimization):** Scryfall provides bulk data dumps (e.g., a big JSON of all cards with prices). We are not planning to use these for this project because it would require downloading a huge dataset and hosting our own search logic. Instead, we rely on Scryfall's live search for accurate and up-to-date info. The mention of bulk search in requirements is addressed by using the batch collection and search APIs rather than truly one-by-one calls.

By structuring queries with Scryfall's powerful search syntax, we minimize post-processing on our side. We let Scryfall filter by price, type, color, etc., returning exactly the subset of cards we want (e.g., "commons under $5 with similar attributes"[2]). This keeps our logic simpler and leverages Scryfall's well-maintained database for correctness.

## State Management

The application state will be handled primarily on the client side using React state and possibly context for global states. Here's an outline of the state and how it flows through the app:

- **Deck List State:** Once parsed, the array of deck entries (name + quantity + original card data) is stored in a state variable, say `deckList`. This might be in a context or lifting state up to the top-level page component, since both the input panel and results need it. It will contain additional info after the Scryfall fetch of original cards, e.g. each entry could be `{ name, quantity, details, price }` where `details` includes mana cost, type, etc., and `price` is the original card price.

- **Filter State:** Each filter control (`maxPrice`, `allowedTypes`, `allowedRarities`, `strictText`) has its own piece of state. These can live in a Filters context or in the same parent component that triggers the search. When the user modifies filters before search, it just updates these state values. If filters are changed after initial results, we might either re-run the proxy search or filter the already fetched suggestions:
  - For MVP, it's simpler to require the user to click an "Update Results" button which re-fetches suggestions with the new filters.
  - Alternatively, we could fetch a superset (e.g. under a higher price) and then filter client-side for minor changes, but since price and other criteria can drastically change the set of results, re-query is safer to ensure accuracy.
  - The filter state is also used in constructing queries (`maxPrice` goes into `usd<=`, rarities into `rarity:` filter, etc.).

- **Loading/Progress State:** A boolean or state machine indicates when a search is in progress. For example, `isLoading` can be `true` while contacting Scryfall. This triggers UI indicators (like disabling the submit button, showing spinners). We may also maintain progress info (e.g., number of cards processed out of total) if we want to show a progress bar. However, given the short wait times (a few seconds), a simple spinner is likely enough.

- **Proxy Suggestions State:** This could be structured as a mapping from original card name (or an ID) to an array of proxy options. For example:
  ```javascript
  suggestions = {
    "Lightning Bolt": [ {card: "Shock", set: "M10", price: 0.15, img: "...", rarity: "Common"}, {...}, ... ],
    "Tundra": [ {card: "Glacial Fortress", set: "M19", price: 3.50, img: "...", rarity: "Rare"}, ... ],
    ...
  }
  ```
  We will populate this after fetching results. It might be stored in a React state hook or in a context if deep component tree. Likely, the main results panel component will hold it and pass down relevant slice to each ProxySuggestionsCard.

- **Selection State:** For tracking which proxy option the user chose for each card, we can have a dictionary or Map: `selectedProxy = { [cardName]: indexOrIdOfSelection }`. Alternatively, we can store the full selected card object for each original:
  ```javascript
  selected = {
    "Lightning Bolt": {card: "Shock", set: "M10", price: 0.15, ...},
    "Tundra": {card: "Glacial Fortress", ...},
    ...
  }
  ```
  This state is updated when the user clicks a ProxyOption. The ProxySuggestionsCard can call a handler (passed via props or context) like `onSelect(cardName, proxyOption)`. That updates the state and triggers a re-render so the UI can highlight the selection and update summary.

- **Summary State:** We might not need separate state for the summary; it can be computed on the fly from `selectedProxy` and the known prices. However, to optimize or simplify, we might derive some values:
  - `originalTotalPrice` – computed once after fetching original card data (sum of all `deckList[i].price * quantity`).
  - `proxyTotalPrice` – computed whenever `selectedProxy` changes (sum of each selected card's price * original quantity).
  - We can use a React `useMemo` to compute totals whenever dependencies change, rather than storing them as independent state (to avoid inconsistency).
  - The summary list of selected proxies can be generated by mapping over `selectedProxy`.

- **Context vs Prop Drilling:** Given the app's scale, using React Context for `deckList`, `suggestions`, and `selectedProxy` could be convenient, especially if deeply nested components need access (e.g., an ExportDialog might just consume selected proxies to produce text). We will likely create a context provider at a high level (e.g., `DeckContext`) that provides:
  - the parsed deck entries,
  - the suggestions map,
  - the selection map,
  - and maybe functions to update these (like a function to start the search, or to select a proxy).
  - This avoids prop drilling through each component.
  - The state hooks (`useState` or `useReducer`) themselves can live in the context provider component.

- **Using Reducer:** We might use a `useReducer` for managing the complex state transitions (from input to loaded suggestions to selections). For example, an action type `"SET_SUGGESTIONS"` updates the suggestions state, `"SELECT_PROXY"` updates the selection state. This is an internal implementation detail – not strictly necessary, but can help organize state updates.

- **Resetting State:** If the user clears or replaces the input and submits a new deck, the state should reset accordingly:
  - Clear out old suggestions and selections.
  - Possibly keep filters (or reset them to defaults, UX decision).
  - Ensure the UI returns to initial state (aside from maybe keeping the last deck's summary until a new search starts, but better to clear to avoid confusion).

- **Local Storage (optional):** We are not required to persist anything, but it could be nice to remember the last filters the user set (e.g., if they always want commons only). This could be done via browser local storage. However, it's an enhancement not specified, so we can omit persistence for now.

In summary, React's state will handle transient data for this app. The flow is:
- User input triggers state update (`deckList`, `loading`).
- After API calls, state updates (`suggestions` ready, `loading` false).
- User selections update state (`selectedProxy`).
- State changes cause UI re-renders, which update the displayed suggestions and summary totals.

This unidirectional data flow makes the app predictable and easier to maintain or expand.

## Project Structure (Next.js + Tailwind)

Below is the suggested file and directory structure for the Next.js project implementing PaintProxy:

```
paintproxy/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── globals.css
│   └── api/
│       └── proxySuggestions/route.ts
├── components/
│   ├── DeckInputPanel.tsx
│   ├── FilterPanel.tsx
│   ├── ResultsPanel.tsx
│   ├── ProxySuggestionsCard.tsx
│   ├── ProxyOptionCard.tsx
│   ├── SummaryPanel.tsx
│   └── ExportDialog.tsx
├── components/ui/   (Shadcn UI components)
│   ├── button.tsx
│   ├── input.tsx
│   ├── textarea.tsx
│   ├── slider.tsx       (...etc for any needed Shadcn components)
│   └── ... (Shadcn internal dependencies like provider if needed)
├── lib/
│   ├── parseDeck.ts
│   ├── scryfall.ts
│   └── types.ts
├── public/
│   └── ...(any static images or icons if needed)
├── tailwind.config.js
├── package.json
└── README.md
```

### Explanation

- **`app/` directory:** Using Next.js App Router.
  - `layout.tsx` defines the root layout (could include a header with the app name "PaintProxy", and applies global styles and fonts).
  - `page.tsx` is the homepage which will render the main interface. This can be a React client component since we handle interactive state. It will likely import the DeckInputPanel, FilterPanel, ResultsPanel, etc., composing them into the two-column layout.
  - `globals.css` imports Tailwind's base, components, and utilities, plus any custom CSS. For example, Tailwind is configured here with `@tailwind base; @tailwind components; @tailwind utilities;`. It may also set some global styles (like body font, scroll behavior).
  - `api/proxySuggestions/route.ts`: This is an API route (using Next.js 13's Route Handler) that can handle the heavy lifting of contacting Scryfall. For instance, the client could POST the parsed deck and filters to this endpoint, and this serverless function would then perform the Scryfall API calls (using fetch on the server side), and return the suggestions JSON. This keeps the client lightweight and avoids CORS issues. If we choose to do all fetches on the client, this file might not be needed. But having it gives flexibility to offload work to the server and hide API details from client. In our implementation, we can start without an API route (direct client fetch), and introduce it if needed for performance.

- **`components/`:** This folder contains our custom React components (as identified in the breakdown). Each is likely a functional component with its own props:
  - `DeckInputPanel.tsx` – contains the textarea and submit button, and perhaps incorporates FilterPanel or renders it conditionally.
  - `FilterPanel.tsx` – contains filter controls. Might be used inside DeckInputPanel or above it.
  - `ResultsPanel.tsx` – container for the list of ProxySuggestionsCards and the SummaryPanel. This component receives the deck list and suggestions from context or props, and maps over them to render each ProxySuggestionsCard.
  - `ProxySuggestionsCard.tsx` – as described, shows one original card's suggestions. It likely takes props like the original card info and the list of proxy options, as well as any selection state and onSelect handler.
  - `ProxyOptionCard.tsx` – a subcomponent for the card options (could be defined within ProxySuggestionsCard, but separating keeps it modular). Handles displaying a single option.
  - `SummaryPanel.tsx` – displays the summary of selections. It gets the totals and selections from context/props and renders the output. Could also include the Export button.
  - `ExportDialog.tsx` – a modal dialog that shows the export text and a copy button. (This might internally use Shadcn's Dialog component to render, or could be a simple conditional render of a `<pre>` block. Using a Dialog is cleaner for accessibility and styling.)
  - Each of these components will be styled with Tailwind classes and can use Shadcn UI elements internally (for example, FilterPanel might import a Shadcn Slider or Switch).

- **`components/ui/`:** This directory is created by Shadcn's CLI when adding components. It will contain the source for any Shadcn UI components we use. For example, after running `npx shadcn add button input textarea slider switch dialog`, etc., those files appear here (like `button.tsx`, which exports a styled `<Button>` component). We will use these in our components (e.g. `import { Button } from "@/components/ui/button"`). These are fully customizable – if design tweaks are needed, we can edit these files. The presence of these files means our app doesn't rely on an external component library at runtime; it's all compiled in.

- **`lib/`:** Utility modules.
  - `parseDeck.ts` will contain the function that takes the raw input string and outputs the parsed list of `{name, quantity}`.
  - `scryfall.ts` can hold functions for calling Scryfall API. For example, `fetchCardDetails(names: string[])` for the collection query, and `searchProxiesForCard(card, filters)` for constructing the query string and fetching results. Abstracting these out keeps the component code cleaner.
  - We can also define types here in `types.ts` for clarity:
    - `type DeckEntry = { name: string; quantity: number; price?: number; details?: any }`
    - `type ProxyOption = { name: string; set: string; price: number; image: string; rarity: string }`
    - etc., to use across the app. This ensures consistency and helps Cursor/AI codegen to understand the data structures.
  - We might include a `useDeck` custom hook in this lib or in context file, encapsulating state logic for the deck and suggestions (if not using context, not needed).

- **`public/`:** Static assets. Not many are expected for this app since we fetch card images from Scryfall. But if we want to add a logo or an icon (perhaps a favicon or a small logo for the site), it would go here. Also, if we wanted a default image placeholder or any theme images, they'd be here.

- **`tailwind.config.js`:** Tailwind configuration. We need to ensure it includes Shadcn presets if required (Shadcn might require adding `shadcn/plugin` for their typography or aspects). Also, configure the content paths to include our component files (e.g., `"./app/**/*.{js,ts,jsx,tsx}"`, `"./components/**/*.{js,ts,jsx,tsx}"`). We might also set a custom color or font here if desired for the accent/theme. Shadcn's default style uses CSS variables for theming (it has a default theme we can keep or adjust).

- **`package.json`:** Will include dependencies like `"next"`, `"react"`, `"tailwindcss"`, and Shadcn's CLI as a dev dependency if needed. It might also include something like `"@tanstack/react-query"` if we use it (but likely not necessary for straightforward data fetching).

- **`README.md`:** Documentation for developers on how to run and develop the project (out of scope for spec, but will mention how to start Next dev server, etc.)

### Styling Considerations

We will enable both light and dark mode in Tailwind (Shadcn supports dark mode toggling easily[16]). Possibly default to light theme, but ensuring that using `dark:` classes we can have an alternate dark style if needed (ChatGPT UI is dark by default; we could mimic that). This can be decided during implementation.

### Development Approach

The spec will be fed into Cursor (an AI codegen tool), so the code generation will proceed component by component as specified. We should implement and test the parsing, API calls, and UI incrementally. Because the spec is detailed, the codegen should produce the desired components. After generation, we will integrate the pieces, test with some sample deck input (like a small deck), adjust any styling issues (especially for the carousel behavior and responsiveness), and then deploy (Vercel can host a Next.js app easily, and Scryfall API usage is client-friendly).

Finally, we will ensure to credit Scryfall per their terms (e.g., "Card data and images © Scryfall" can be noted in the footer or about section) since we are using their API.

---

## References

[1] [2] [6] [7] [Searching with Scryfall: Magic at your Fingertips — Lucky Paper](https://luckypaper.co/articles/searching-with-scryfall-magic-at-your-fingertips/)

[3] [cards/collection · API Documentation - Scryfall](https://scryfall.com/docs/api/cards/collection)

[4] [Using Scryfall API on Google Sheets : r/magicTCG](https://www.reddit.com/r/magicTCG/comments/ff5ho9/using_scryfall_api_on_google_sheets/)

[5] [16] [Free Open Source shadcn/ui Templates & Components - Beautiful React UI Templates | shadcn.io](https://www.shadcn.io/template)

[8] [9] [10] [11] [12] [13] [14] [15] [Next.js - shadcn/ui](https://ui.shadcn.com/docs/installation/next)
