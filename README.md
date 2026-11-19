# LLM Pipeline Explorer

> **"How text becomes tokens, and then becomes points in 3D space."**

LLM Pipeline Explorer is an interactive, single-page educational web application designed to demonstrate the fundamental pipeline of Large Language Models (LLMs):

$$\text{TEXT} \longrightarrow \text{TOKENIZATION} \longrightarrow \text{TOKEN IDs} \longrightarrow \text{EMBEDDINGS} \longrightarrow \text{3D VISUALIZATION} \longrightarrow \text{SIMILARITY SEARCH}$$

Built with an **Apple-inspired light interface**, the application runs entirely client-side in any modern web browser without requiring build systems, package managers, or backend services.

---

## Project Structure

The repository strictly contains only two deliverable files:

```
llm-tokeniser/
├── index.html   # Complete application (HTML5, CSS3, JavaScript ES6+, Three.js)
└── README.md    # Documentation, architecture, and technical reference
```

---

## Key Features

- **Apple-Inspired Light Research Interface** — Clean soft off-white canvas (`#f8fafc`), pure white cards (`#ffffff`), crisp dark charcoal typography (`#0f172a`), subtle grey borders, and electric blue/violet accents.
- **Corpus-Derived Vocabulary Builder** — Learns vocabulary entries, root stems, punctuation marks, and continuation subwords (`##`) directly from the training corpus.
  - **Zero hardcoded stems or continuation tokens.**
  - **Explicit Metric Distinction** — Transparently differentiates **Corpus Frequency** (actual standalone occurrences in training text) from **Derived Stem Support** (number of accepted decompositions supporting a stem).
- **Educational WordPiece-Inspired Greedy Subword Tokenizer**:
  - Matches exact words first.
  - Recursively splits unseen words using greedy longest-match prefix matching followed by continuation subwords (`##`).
  - **Unique Token IDs** — Continuation tokens (e.g. `##s`, `##ed`, `##ing`) receive distinct vocabulary IDs (never sharing IDs with root tokens).
  - **Single `[UNK]` Fallback** — If an entire word cannot be segmented using vocabulary pieces, it returns exactly one `[UNK]` token for the whole word.
  - **Discrete Punctuation Preservation** — Punctuation marks are preserved as separate tokens with distinct IDs.
- **Color-Coded Token Breakdown & Interactive Inspector** — Visual token chips with distinct treatments for Corpus Words, Derived Stems, `## Continuation Subwords`, `Punctuation`, and `[UNK]` tokens. Clicking any chip displays a comprehensive inspection card showing token ID, category, subword status, corpus frequency, stem support, and 32D vector summary.
- **Deterministic 32D Hash Embeddings** — Seeded FNV-1a hash mixer generating deterministic numerical vectors in $[-1, 1]$ combined across tokens via mean pooling.
- **Approximate PCA 3D Projection (Three.js)** — Projects 32-dimensional sentence vectors into an interactive 3D WebGL scene using empirical covariance estimation, power iteration, and Hotelling's matrix deflation.
  - **Numerical Safeguards** — Gracefully handles identical sentences (zero-variance warning), degenerate dimensions (anchoring unvaried axes strictly to $0$ without inventing artificial axes), and finiteness validation.
  - **Distinct Visual Styling for Custom Additions** — User-added sentences render with an enlarged radius, vibrant amber/orange styling, and an outer halo ring.
- **Transactional State & WebGL Resource Management** — 3D scene points and labels are never cleared until input validation, embedding generation, and PCA calculations succeed. On recalculation, old Three.js geometries, materials, child meshes, and DOM elements are cleanly disposed of (`.dispose()`) to prevent memory leaks.
- **Vocabulary Rebuild State Invalidation** — Rebuilding the vocabulary automatically invalidates previous embeddings, PCA bases, custom points, and closest-pair rankings, disabling custom sentence additions until a fresh 3D projection is calculated.
- **Ranked Closest Pairs Table** — Displays pairwise cosine similarity scores with click-to-highlight animations in the 3D scene.
- **Dynamic "Try Your Own" Input** — Tokenize, embed, and project custom sentences live into the active 3D coordinate space.

---

## Technology Stack

| Layer | Technology | Details |
|---|---|---|
| **Markup & Styling** | HTML5, CSS3 | Single-file architecture, Apple light theme, CSS variables, responsive grid |
| **Logic & Algorithms** | Vanilla JavaScript (ES6+) | No frameworks, no bundlers, no build step |
| **3D Graphics** | Three.js (r128) | Loaded via CDN (`cdnjs.cloudflare.com`), OrbitControls, WebGLRenderer, Raycasting |
| **Typography** | Google Fonts | Inter & JetBrains Mono loaded via Google Fonts CDN |
| **Dependencies** | External CDN only | **An active internet connection is required on initial load** to fetch Three.js and Google Fonts from CDN |

---

## 1. Vocabulary Construction & Frequency Distinction

The vocabulary is derived dynamically from the supplied training text:

```
[Training Corpus Text]
         │
         ▼
[Text Normalization]  ──► Lowercasing, punctuation separation (.,!?;:() etc.)
         │
         ▼
[Frequency Counting]  ──► Empirical occurrence counts for words and punctuation
         │
         ▼
[Suffix Discovery]    ──► Scans candidate partitions (word = stem + suffix, stem length >= 3)
         │
         ▼
[Continuation Filter] ──► Validates suffixes that appear across multiple words
         │                or attach to known standalone corpus stems
         ▼
[Unique ID Assignment]──► ID 0 strictly reserved for [UNK]
                          IDs 1..P for punctuation
                          IDs P+1..W for standalone corpus words
                          IDs W+1..S for purely derived stems
                          IDs S+1..N for continuation subwords (##...)
```

### Clarification: Suffix-Discovery Heuristic

> The application uses a corpus-derived suffix-partition heuristic to construct a simplified WordPiece-inspired vocabulary. It is an educational approximation, not the original WordPiece training algorithm or a linguistically complete morphological analyzer.

- The heuristic examines possible word splits (`stem + suffix`) with `stem.length >= 3` and evaluates suffix occurrences across corpus words.
- Suffixes that occur across multiple words (or attach to standalone corpus stems) are accepted, while trailing sub-suffixes (e.g. `ng` when `ing` covers the matching words) and coincidental endings are filtered out.
- Zero hardcoded stems, zero hardcoded continuation tokens, and zero fabricated frequencies are used.

### Corpus Frequency vs. Derived Stem Support

To prevent learner misconceptions, the application strictly distinguishes between two different metrics:

| Metric | Definition | Example in Default Corpus |
|---|---|---|
| **Corpus Frequency** | The number of times a token appeared as a complete, independent word in the input text. | `"cat"` appeared $1$ time as a complete word (`Corpus Freq: 1`). `"rest"` never appeared independently (`Corpus Freq: 0`). |
| **Derived Stem Support** | The number of accepted corpus-derived word decompositions that support the stem. | `"rest"` was identified as the prefix in `"rested"` (`Stem Support: 1 decomp`). `"learn"` was supported by `"learning"` (`Stem Support: 1 decomp`). |

Continuation subwords (`##...`) display their empirical occurrence counts across corpus words (e.g., `##ing` with $4$ occurrences, `##s` with $3$ occurrences, `##ed` with $2$ occurrences).

---

## 2. Subword Tokenization Algorithm

The application implements an educational **WordPiece-inspired greedy subword tokenizer**:

1. **Punctuation Isolation** — Punctuation characters are isolated as discrete tokens.
2. **Exact Word Lookup** — If an input word is already in the vocabulary, it is emitted directly.
3. **Greedy Longest-Match Segmentation**:
   - Searches for the longest prefix of the word matching an existing root/stem token in the vocabulary.
   - For remaining characters, prepends `##` and searches for the longest matching continuation token.
   - Repeats until the entire word is consumed.
4. **Single `[UNK]` Fallback**:
   - If any segment of the word cannot be matched to any vocabulary piece, the entire word fails subword decomposition and emits **one single `[UNK]` token (ID #0)**.
5. **Punctuation Preservation**:
   - Punctuation characters (such as `.`, `,`, `!`) are preserved as separate tokens with their own unique IDs.

### Worked Examples

#### Example A: Subword Splitting (`"The cats learned Python coding."`)
- `"cats"` $\longrightarrow$ `["cat" (#ID), "##s" (#ID)]`
- `"learned"` $\longrightarrow$ `["learn" (#ID), "##ed" (#ID)]`
- `"Python"` $\longrightarrow$ `["python" (#ID)]` (Exact root match)
- `"coding"` $\longrightarrow$ `["coding" (#ID)]` (Exact root match)
- `"."` $\longrightarrow$ `["." (#ID)]` (Punctuation preserved)

#### Example B: Unseen / Unmatchable Word (`"unprecedented zebra spaceship"`)
- `"zebra"` is not in the vocabulary and cannot be segmented $\longrightarrow$ `["[UNK]" (#0)]` (Single `[UNK]` for the entire word).

---

## 3. 32D Deterministic Embeddings & Mean Pooling

### Token Vector Generation
Each token string is converted to a 32-dimensional vector $\mathbf{v} \in [-1, 1]^{32}$ using a deterministic FNV-1a hash mixer:

```javascript
h = 32-bit FNV-1a hash of token character codes
For dimension d = 0 to 31:
  hd   = mix(h, d * 2654435761)
  v[d] = ((hd / 0xFFFFFFFF) * 2) - 1
```

- **Deterministic**: The same token always produces the exact same 32D vector.
- **Distinct**: Root token `"ing"` and continuation token `"##ing"` yield completely different hash vectors.

### Sentence Mean Pooling
Sentence embeddings $\mathbf{S}$ are computed by averaging the 32D vectors of their constituent tokens:

$$\mathbf{S} = \frac{1}{T} \sum_{t=1}^{T} \text{tokenToVector}(t)$$

> [!NOTE]
> **Educational Simulation Disclaimer:**
> These embeddings are deterministic simulated vectors generated from token hashes to demonstrate vector geometry, mean pooling, and dimensionality reduction. They do not represent learned semantic representations from a neural network.

---

## 4. Approximate PCA & 3D WebGL Visualization

To visualize 32-dimensional sentence vectors in a 3D Three.js coordinate space, the application computes an **approximate Principal Component Analysis (PCA)** using power iteration and Hotelling's matrix deflation:

1. **Mean-Centering**:
   $$\mathbf{\mu} = \frac{1}{N} \sum_{i=1}^N \mathbf{S}_i, \quad \mathbf{X}_i = \mathbf{S}_i - \mathbf{\mu}$$
2. **Empirical Covariance Matrix**:
   $$\mathbf{C} = \frac{1}{N - 1} \mathbf{X}^T \mathbf{X} \quad (32 \times 32)$$
3. **Power Iteration & Matrix Deflation**:
   - For each principal component $c \in \{1, 2, 3\}$, power iteration (300 iterations) approximates the dominant eigenvector $\mathbf{v}_c$ and eigenvalue $\lambda_c$.
   - Hotelling's deflation removes the variance along $\mathbf{v}_c$:
     $$\mathbf{C}_{k+1} = \mathbf{C}_k - \lambda_c \mathbf{v}_c \mathbf{v}_c^T$$
4. **Projection**:
   $$P_{i, c} = \mathbf{X}_i \cdot \mathbf{v}_c$$
5. **Projection of Custom Sentences**:
   - Custom sentences added in Section 03 are projected using the **stored principal components and empirical mean vector**, ensuring that previously plotted points maintain fixed coordinates.

### Numerical Safeguards
- **Zero Variance Detection**: If input sentences yield identical embeddings, PCA halts safely with an informative error rather than dividing by zero.
- **Degenerate Dimension Handling**: If input sentences span fewer than 3 significant dimensions (e.g. only 2 distinct sentences), unvaried axes are set strictly to $0$ without inventing artificial dimensions, and a clear dimensional notice is displayed.
- **Finite Value Guards**: All calculated coordinates are validated with `Number.isFinite()` to prevent `NaN` or `Infinity` from reaching WebGL.

### Transactional Workflow & Resource Disposal
- **Transactional Update**: Old points, lines, and labels are cleared only after input validation, embedding generation, and PCA calculation succeed. If validation fails, the existing visualization remains undisturbed.
- **Resource Disposal**: On every replot, all Three.js meshes, geometries, materials, child halo rings, and DOM label elements are explicitly traversed and `.dispose()`d to eliminate WebGL memory leaks.

---

## 5. Vocabulary Rebuild & State Invalidation

When the user modifies the training corpus and rebuilds the vocabulary:

1. **Embedding Invalidation**: All previously computed sentence embeddings and PCA basis vectors (`pcaComponents`, `pcaMean`, `pcaScale`) are cleared.
2. **Visualization Reset**: The 3D scene points, custom halo meshes, and HTML point labels are disposed of and hidden.
3. **Custom Input Protection**: The custom sentence insertion button is disabled, and an informative banner is displayed in Section 02 and Section 03.
4. **Required Action**: The user must click **"Embed and Plot"** to compute fresh embeddings and a new PCA projection basis before custom sentences can be added. This guarantees that custom points are never projected into a mismatched or stale coordinate system.

---

## 6. Cosine Similarity & Closest Pairs

Pairwise cosine similarity evaluates the angular alignment between 32-dimensional sentence vectors:

$$\text{sim}(\mathbf{u}, \mathbf{v}) = \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\| \|\mathbf{v}\|} = \frac{\sum_{i=1}^{32} u_i v_i}{\sqrt{\sum_{i=1}^{32} u_i^2} \sqrt{\sum_{i=1}^{32} v_i^2}}$$

- Scores range from $-1.0$ (opposite orientation) to $+1.0$ (identical orientation).
- The **Closest Pairs** table lists the top 5 sentence pairs ranked by similarity.
- Clicking any row pulses both corresponding spheres in the 3D scene.

---

## How to Run

1. **Download or Clone** this repository:
   ```bash
   git clone <repo-url>
   cd llm-tokeniser
   ```
2. **Open the Application**:
   - Double-click `index.html` or open it directly in any modern web browser (Chrome, Safari, Firefox, Edge).
   - Alternatively, serve it locally if desired:
     ```bash
     npx serve .
     # or
     python3 -m http.server 8000
     ```
3. **Internet Requirement**:
   - An active internet connection is required on first launch to load CDN dependencies (Three.js and Google Fonts).
   - No build tools, package managers, or server installations are necessary.

---

## Educational Walkthrough

1. **Section 01 — Tokenizer**:
   - Click **"Build Vocabulary"** to extract corpus-derived root words, stems, punctuation, and continuation subwords.
   - Inspect the table to observe the difference between **Corpus Freq** and **Stem Support**.
   - Use the sample preset buttons or type `"The cats learned Python coding."` and click **"Tokenize"**.
   - Observe how `"cats"` splits into `"cat"` + `"##s"` and `"learned"` splits into `"learn"` + `"##ed"`.
   - Click any token chip to inspect its Token ID, category, corpus frequency, and stem support.
2. **Section 02 — Embeddings in 3D**:
   - Click **"Embed and Plot"** to calculate 32D embeddings, execute approximate PCA, and render the 3D WebGL scene.
   - Drag with left-click to orbit, right-click to pan, and scroll to zoom.
   - Click **"Auto-Rotate"** for automated turntable rotation, or **"Reset View"** to smoothly return to default vantage.
   - Inspect the **Closest Pairs** table and click any row to highlight the pair in 3D space.
3. **Section 03 — Try Your Own**:
   - Enter a custom sentence (e.g. `"The intelligent computer learned patterns."`) and click **"Tokenize, Embed & Add to Plot"**.
   - Observe how the custom sentence is highlighted with an orange halo in the 3D coordinate space.
4. **Vocabulary Rebuild Flow**:
   - Change the training text in Section 01 and click **"Build Vocabulary"**.
   - Notice that previous 3D points are safely cleared and Section 03 prompts you to re-run **"Embed and Plot"** to generate a fresh coordinate basis.

---

## Limitations Compared with Production LLMs

| Property | This Educational Simulator | Production LLM (e.g. GPT-4, Gemini, Llama 3) |
|---|---|---|
| **Vocabulary Construction** | Corpus-derived greedy morphological subword heuristic | Trained Byte-Pair Encoding (BPE), WordPiece, or Unigram on gigabytes of text |
| **Vocabulary Size** | ~50 to 100 tokens | 32,000 to 256,000+ tokens |
| **Embedding Dimensions** | 32 dimensions (deterministic FNV-1a hash) | 768 to 12,288+ learned dense dimensions |
| **Semantic Representation** | Lexical/character hash overlap | Deep contextual semantic geometry learned over trillions of tokens |
| **Dimensionality Reduction** | Approximate PCA (Power Iteration + Deflation) | High-dimensional transformer representations visualized via t-SNE, UMAP, or exact SVD/PCA |
| **Context & Attention** | Mean pooling across token vectors | Multi-head self-attention layers with positional encodings |

---

*Built for educational exploration of LLM internals: tokenization, vocabulary mapping, vector spaces, PCA dimensionality reduction, and cosine similarity.*
