# Shopify & FlutterFlow: Production-Ready Search Page Architecture

A guide to building a high-converting, mobile-optimized search and discovery page in FlutterFlow powered directly by the Shopify Storefront GraphQL API.

---

## 1. Benefits & Key Features

* **Zero-Latency Search History:** Stores up to 5 recent keywords locally on the customer's phone using persistent state, delivering an instant load without extra database overhead or cloud costs.
* **Instant Discovery (No Empty Screens):** When the search field is idle, the screen automatically populates with recent search icons, curated popular keyword chips, and a live feed of your store's top 20 best-selling products.
* **Shopify Predictive Search:** Returns real-time, fuzzy-matched catalog search suggestions and products directly from Shopify as the user submits queries.
* **Crash-Resistant Responsive Grid:** Built using a 2-column layout that avoids Flutter's unbounded height viewport errors inside scrollable screens.
* **Dynamic View Switching:** Automatically toggles between the discovery/history view and the active search results grid without navigating across multiple pages.

---

## 2. Prerequisites & API Configuration

Ensure your Shopify Storefront API access is active. All calls use your store domain and public Storefront Access Token.

### API Call 1: Predictive Search (`ShopifyPredictiveSearch`)

* **Method:** `POST`
* **URL:** `https://<YOUR-STORE-DOMAIN>/api/2024-07/graphql.json`
* **Headers:**
* `Content-Type`: `application/json`
* `X-Shopify-Storefront-Access-Token`: `<YOUR_PUBLIC_STOREFRONT_TOKEN>`


* **Variables:**
* `searchQuery` (`String`, default: `"shirt"`)


* **Body (JSON):**
```json
{
  "query": "query SearchProducts($query: String!) { predictiveSearch(query: $query, types: [PRODUCT], limit: 10) { products { id title handle vendor priceRange { minVariantPrice { amount currencyCode } } featuredImage { url altText } } } }",
  "variables": {
    "query": [searchQuery]
  }
}

```


* **Output Path to Save:** `$.data.predictiveSearch.products` (name it `productsList`).

---

### API Call 2: Best-Selling Products (`GetBestSellingProducts`)

* **Method:** `POST`
* **URL:** `https://<YOUR-STORE-DOMAIN>/api/2024-07/graphql.json`
* **Headers:** Same as above.
* **Body (JSON):**
```json
{
  "query": "query GetBestSellingProducts { products(first: 20, sortKey: BEST_SELLING) { edges { node { id title handle featuredImage { url altText } priceRange { minVariantPrice { amount currencyCode } } } } } }"
}

```


* **Output Path to Save:** `$.data.products.edges` (name it `bestSellingEdges`).

---

## 3. FlutterFlow State & Custom Functions

### Persistent App State

Create a persistent App State variable to preserve search keywords locally across app relaunches:

* **Field Name:** `recentSearches`
* **Data Type:** `List <String>`
* **Persisted:** `True`

### Page State Variables

In your Search Page, create a local state variable:

* **Field Name:** `searchResults`
* **Data Type:** `List <JSON>` (or `List <dynamic>`)

### Custom Function: Clean Keyword History

Create a Custom Function in **Custom Code > Functions** named `addToRecentSearches`:

* **Return Type:** `List <String>` (Nullable: `True`)
* **Arguments:**
* `currentSearches`: `List <String>` (Nullable: `True`)
* `newKeyword`: `String` (Nullable: `True`)



```dart
List<String>? addToRecentSearches(
  List<String>? currentSearches,
  String? newKeyword,
) {
  final cleanKeyword = newKeyword?.trim() ?? '';
  if (cleanKeyword.isEmpty) {
    return currentSearches ?? [];
  }

  List<String> updated = List<String>.from(currentSearches ?? []);

  // Remove duplicate so the keyword bubbles up to the first slot
  updated.removeWhere(
    (item) => item.toLowerCase() == cleanKeyword.toLowerCase(),
  );

  // Insert newest search at index 0
  updated.insert(0, cleanKeyword);

  // Cap history at 5 items
  if (updated.length > 5) {
    return updated.sublist(0, 5);
  }

  return updated;
}

```

---

## 4. Widget Tree Structure

Construct the page layout inside a `Scaffold` using this hierarchy:

```text
Scaffold
└── SafeArea
    └── Column (Main, stretch)
        ├── Top Bar Container (Fixed header)
        │   └── Row
        │       ├── Back IconButton
        │       ├── Search Icon
        │       └── Searchbar (TextFormField / Autocomplete)
        └── Expanded (Flex: 1)
            └── Stack
                ├── Active Results View (Conditional: Searchbar is NOT empty)
                │   └── SingleChildScrollView
                │       └── GridView.builder (2 Columns, ShrinkWrap: True, Physics: NeverScrollable)
                │           └── ProductCard Component
                │
                └── Discovery View (Conditional: Searchbar IS empty)
                    └── SingleChildScrollView
                        └── Column
                            ├── Recent Searches Header Row (Conditional: recentSearches is NOT empty)
                            │   ├── Text ("Recent Searches")
                            │   └── Text ("Clear All") -> Actions: Clear recentSearches
                            │
                            ├── Recent Searches Horizontal ListView (Conditional: recentSearches is NOT empty)
                            │   └── Item: Column (Width: 70px)
                            │       ├── Circular Container with Icon(Icons.history_rounded)
                            │       └── Text (dynamic: recentSearchItem, MaxLines: 2)
                            │
                            ├── Popular Searches Text ("Popular Searches...")
                            ├── Popular Searches ChoiceChips (Preset tags: "T-Shirt", "Shoes", "Mug")
                            │
                            ├── Trending Products Header Text ("Trending & Best Sellers")
                            └── Trending GridView / Container (Backend Query: GetBestSellingProducts)
                                └── GridView.builder (2 Columns, ShrinkWrap: True, Physics: NeverScrollable)
                                    └── ProductCard Component

```

---

## 5. Action Flows

### Flow A: Searchbar Form Submit (Keyboard "Enter")

1. **Update App State:**
* Field: `recentSearches`
* Update Type: `Set Value`
* Value: `addToRecentSearches(currentSearches: recentSearches, newKeyword: Searchbar.text)`


2. **Backend Call (API):**
* Call: `ShopifyPredictiveSearch`
* `searchQuery`: `Searchbar.text`
* Action Output: `searchResponse`


3. **Update Page State:**
* Field: `searchResults`
* Update Type: `Set Value`
* Value: `searchResponse -> JSON Body -> $.data.predictiveSearch.products`


4. **Rebuild Page** (Toggles view to results grid automatically).

---

### Flow B: Tapping a Recent Search or Popular Keyword

1. **Set Form Field:**
* Select `Searchbar`.
* Set Value: Selected string (from the tapped recent item or chip).


2. **Update App State:**
* Run `addToRecentSearches` so the tapped item becomes the most recent search.


3. **Backend Call (API):**
* Run `ShopifyPredictiveSearch` with the tapped keyword.
* Action Output: `searchResponse`


4. **Update Page State:**
* Assign `searchResponse -> JSON Body -> $.data.predictiveSearch.products` to `searchResults`.


5. **Rebuild Page**.

---

### Flow C: Clearing Search (Searchbar Reset / "Clear All")

* **To clear the search input:** Provide a trailing `IconButton` (an "X" icon) on the search bar. On tap:
1. Clear `Searchbar` field.
2. Set `searchResults = []`.
3. Rebuild Page (instantly resets to the discovery screen).


* **To clear search history:** On tapping "Clear All":
1. Update App State `recentSearches` $\to$ `Clear Value`.
2. Rebuild Page (hides the Recent Searches row and horizontal list).



---

## 6. Critical Layout Settings (Preventing Crashes)

* **Preventing Viewport Errors:** Inside a `SingleChildScrollView`, every nested `GridView` or `ListView` **must** have:
* `Shrink Wrap = True`
* `Primary = False` (or Physics set to `NeverScrollableScrollPhysics`).


* **Aspect Ratio Setting:** On the 2-column GridViews, set `Child Aspect Ratio` to `0.62`–`0.68` (width divided by height). If your product images or multi-line titles clip, decrease the aspect ratio (e.g., to `0.60`) to grant additional vertical space.
