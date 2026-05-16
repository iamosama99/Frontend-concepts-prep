# Atomic Design Principles

## Quick Reference

| Level | Definition | Composes | Example |
|-------|-----------|----------|---------|
| Atom | Smallest indivisible UI unit | HTML primitive + tokens | Button, Input, Icon, Label |
| Molecule | Functional grouping of atoms | Atoms | SearchBar, FormField, TagList |
| Organism | Self-contained section of UI | Molecules (+ atoms) | Header, ProductCard, CommentThread |
| Template | Page structure without real content | Organisms | BlogPostLayout, DashboardShell |
| Page | Real content in a template | Templates + real data | BlogPost (populated), Dashboard |

---

## What Is This?

Atomic Design is a mental model and naming methodology introduced by Brad Frost in 2013 for thinking about UI components as a compositional hierarchy. The metaphor comes from chemistry: atoms combine into molecules, molecules combine into organisms, and so on upward. Applied to UI, it gives teams a shared vocabulary for describing component granularity and composition.

The key word is *methodology*, not *folder structure*. Atomic Design tells you how to think about and name components — it does not mandate that you have a directory called `atoms/` or that a `Button` file must live in a specific path. Treating it as a rigid filing system is the most common misapplication.

> **Check yourself:** What is the distinction between an atom and a molecule? What property makes something an atom rather than a molecule?

---

## Why Does It Exist?

### The problem before Atomic Design

Before systematic component thinking, frontend codebases accumulated components without agreed-upon granularity. Teams would have:

- A `Header` component that contained its own search input, nav links, notification bell, and user avatar — all directly, with no intermediate abstractions
- A separate `MobileHeader` that duplicated most of that logic for mobile
- A `Card` component in one feature that was a 400-line file handling three unrelated layout scenarios
- A `Sidebar` in another feature that embedded a completely different version of the same nav links

When a designer changed how links should look, there were six places to update. When a product manager asked for a new page that needed "the same header but without search," the answer was either "copy the header" or "add another prop to the 12-prop Header component."

The core problem was **no shared concept of granularity**. Was a "component" a single HTML element with a class, a functional group of inputs, a full page section, or anything in between? Different developers made different calls, and over time the codebase became a grab-bag of arbitrary abstractions at incompatible scales.

### What Atomic Design solved

Atomic Design gave teams a vocabulary and a decision rule: if you can break a component into smaller independently reusable pieces, you should. Those pieces are lower-level. A component that groups lower-level pieces into a functional unit is at the next level up. The hierarchy creates a consistent, predictable abstraction ladder that any team member can reason about without knowing the entire codebase.

The metaphor also makes composition explicit. Molecules are *composed of* atoms. Organisms are *composed of* molecules. The directional dependency is always upward — lower levels never import from higher levels. This enforced direction is what makes the system maintainable.

---

## How It Works

### Atoms — HTML Primitives + Design Tokens

An atom is the smallest meaningful UI element that cannot be broken down into smaller components without losing its function. In web UI, atoms map closely to native HTML elements enhanced with design tokens and basic behavioral props.

```js
// Button atom — an HTML element wrapped with design system behavior
function Button({ variant = 'primary', size = 'md', disabled, onClick, children }) {
  return (
    <button
      className={`btn btn--${variant} btn--${size}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}

// Icon atom — pure presentational, wraps an SVG sprite or icon system
function Icon({ name, size = 16, label }) {
  return (
    <svg
      aria-label={label}
      aria-hidden={!label}
      width={size}
      height={size}
    >
      <use href={`/icons.svg#${name}`} />
    </svg>
  );
}

// Label atom — semantic label with associated control
function Label({ htmlFor, required, children }) {
  return (
    <label htmlFor={htmlFor}>
      {children}
      {required && <span aria-hidden="true"> *</span>}
    </label>
  );
}
```

Atoms know nothing about layout. A `Button` does not know whether it is in a header, a form, or a card. It takes only the props required for its own function and delegates all positioning to its parent.

Design tokens are the atoms' foundation — `btn--primary` applies `var(--color-action-primary)`, `btn--md` applies `var(--space-2) var(--space-4)`. The connection to Phase 5, File 3 is direct: atoms are where tokens are consumed and where the visual system becomes concrete.

### Molecules — Functional Units

A molecule is a group of atoms that forms a functional unit. The defining characteristic is that the group has a coherent, singular purpose that its individual atoms lack. A `SearchBar` is a molecule: an `Input` atom and a `Button` atom combine to produce a thing that searches. Neither atom alone searches.

```js
// SearchBar molecule — composes Input + Button into a functional unit
function SearchBar({ onSearch, placeholder = 'Search...' }) {
  const [query, setQuery] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    onSearch(query);
  }

  return (
    <form onSubmit={handleSubmit} className="search-bar">
      <Input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder={placeholder}
        aria-label="Search query"
      />
      <Button type="submit" variant="primary">Search</Button>
    </form>
  );
}

// FormField molecule — composes Label + Input + ErrorMessage into a form control unit
function FormField({ id, label, error, required, ...inputProps }) {
  return (
    <div className="form-field">
      <Label htmlFor={id} required={required}>{label}</Label>
      <Input id={id} aria-describedby={error ? `${id}-error` : undefined} {...inputProps} />
      {error && (
        <span id={`${id}-error`} role="alert" className="form-field__error">
          {error}
        </span>
      )}
    </div>
  );
}
```

The molecule is the right level to handle inter-atom coordination: `FormField` wires `htmlFor`/`id`, `aria-describedby`, and the error announcement — concerns that span atoms and that neither atom should own individually.

Molecules are still generic. `SearchBar` does not know it is in a `Header`. `FormField` does not know it is in a registration form. They accept props and delegate data to their parents.

### Organisms — Self-Contained Sections

An organism is a self-contained, relatively complex section of UI that composes molecules (and possibly atoms) into a distinct section of an interface. Organisms are the first level that can be meaningfully placed on a page as a standalone unit.

```js
// Header organism — composes Logo (atom), Nav (molecule), SearchBar (molecule), UserMenu (molecule)
function Header({ user, navItems, onSearch }) {
  return (
    <header className="site-header">
      <Logo />
      <Nav items={navItems} />
      <SearchBar onSearch={onSearch} />
      <UserMenu user={user} />
    </header>
  );
}

// ProductCard organism — a card section with its own coherent visual identity
function ProductCard({ product, onAddToCart }) {
  return (
    <article className="product-card">
      <img src={product.imageUrl} alt={product.name} className="product-card__image" />
      <div className="product-card__body">
        <h3 className="product-card__title">{product.name}</h3>
        <Price amount={product.price} currency={product.currency} />
        <Rating value={product.rating} count={product.reviewCount} />
        <Button variant="primary" onClick={() => onAddToCart(product)}>
          Add to cart
        </Button>
      </div>
    </article>
  );
}
```

The organism difference from molecules: organisms have visual identity at the section level. A `Header` looks like a header even without knowing what page it is on. A `ProductCard` could appear in a grid or a carousel — its visual completeness is self-contained.

Organisms can own local state (is the mobile nav open?), but they should not own data fetching. They receive data via props. Data fetching belongs at the page level or in a container that wraps the organism.

### Templates — Page Structure Without Real Content

A template is the page layout skeleton: the arrangement of organisms in the space, the grid or column structure, the placement of major content zones — without real content. Templates reveal the layout through placeholder content (wireframe-like) and define the structural relationship between sections.

```js
// BlogPostTemplate — layout skeleton for a blog post page
function BlogPostTemplate({ header, content, sidebar, footer }) {
  return (
    <div className="blog-layout">
      <div className="blog-layout__header">{header}</div>
      <div className="blog-layout__body">
        <main className="blog-layout__content">{content}</main>
        <aside className="blog-layout__sidebar">{sidebar}</aside>
      </div>
      <div className="blog-layout__footer">{footer}</div>
    </div>
  );
}
```

Templates are layout components. They do not know what data goes in `content` or `sidebar` — they only define the spatial relationship between slots. This is the level where responsive layout decisions live: how does the sidebar collapse on mobile? What is the max-width of the content column? These are layout questions, not content questions.

Templates are rarely discussed in interviews but are important in practice — they are where you prevent layout logic from bleeding into organism components.

### Pages — Real Content in Templates

A page is a specific instance of a template, populated with real content. Pages are where data fetching, routing, and real user data connect to the template structure.

```js
// BlogPost page — populates BlogPostTemplate with real data
function BlogPostPage({ params }) {
  const post = usePost(params.slug);
  const relatedPosts = useRelatedPosts(params.slug);

  if (post.isLoading) return <BlogPostTemplate header={<Header />} content={<PostSkeleton />} sidebar={null} footer={<Footer />} />;

  return (
    <BlogPostTemplate
      header={<Header />}
      content={<PostContent post={post.data} />}
      sidebar={<RelatedPosts posts={relatedPosts.data} />}
      footer={<Footer />}
    />
  );
}
```

Pages are the level where:
- Route parameters are read
- Data fetching is initiated (or query results from a server component are consumed)
- Authentication checks happen
- Page-level metadata (document title, og:tags) is set
- The template receives its real slot content

Pages call organisms; organisms do not reach back up to access page-level data. The data flows down. This connects directly to unidirectional data flow from Phase 4.

> **Check yourself:** At which level of the hierarchy does data fetching belong, and why does putting it in organisms create problems?

---

## Real-World Adaptation: The Three-Level Model

Most product teams find that five levels of abstraction introduce overhead without proportional benefit. The naming and the concepts are useful, but a `templates` directory alongside a `pages` directory alongside an `organisms` directory is four or five places to look for any given component.

The pragmatic adaptation most mature teams converge on is a **three-level model**:

| Three-level term | Maps to Atomic | Notes |
|-----------------|----------------|-------|
| Primitives / UI | Atoms + simple molecules | Button, Input, Icon, FormField, Badge |
| Components | Molecules + organisms | ProductCard, Header, SearchBar, Sidebar |
| Features / Views | Templates + pages | ProductListPage, CheckoutFlow, DashboardView |

This gives the conceptual benefits — thinking in levels of abstraction, enforcing upward-only composition — without the friction of managing five named levels in a folder tree.

**When to use all five levels:** When you are building a published design system consumed by many teams, the full five-level vocabulary is useful because it is explicit documentation of component intent. A component in `organisms/` signals "this is a self-contained section; it composes molecules." That signal helps consumers of the library use components correctly.

**When to use three levels:** When you are building a single product, the full five levels add directory depth without clarity. Three levels cover 95% of real decisions.

---

## Colocation Principle

Atomic Design, regardless of which level model you use, works best when all artifacts related to a component live alongside the component file:

```
components/
  Button/
    Button.js         # component implementation
    Button.test.js    # unit and integration tests
    Button.stories.js # Storybook stories
    Button.module.css # scoped styles (if using CSS Modules)
    index.js          # public export
```

The colocation principle is not specific to Atomic Design but it is reinforced by it. When you understand that a `Button` is an atom — a self-contained, independently usable primitive — it becomes obvious that everything about the Button should travel with it. Separating `Button.test.js` into a root-level `__tests__/` directory means the test file has no relationship to the component in the file system, making it harder to find, easier to forget, and harder to delete cleanly.

Co-locate: styles, tests, stories, documentation (if you have component-level docs), and type definitions. Export publicly through the `index.js` barrel.

---

## Gotcha: Atomic Design Is a Methodology, Not a Folder Structure

This is the most important practical gotcha with Atomic Design.

The mistake is treating atom/molecule/organism/template/page as literal directory names and enforcing that every component file lives in exactly the right directory. This creates several problems:

**Ambiguity causes paralysis.** Is a `Dropdown` an atom or a molecule? It composes a `Button` and a list of `Option` atoms — but it feels atomic in its usage. Teams spend time debating folder placement rather than building.

**Refactoring changes folder location.** If a component that was an atom gains internal composition and is refactored into a molecule, it has to move directories, which changes the import path, which breaks imports across the codebase, which requires a grep-and-replace across potentially dozens of files.

**"Atom" and "molecule" directories become false taxonomies.** The directory name implies the component is at that level of abstraction — but abstraction level is a spectrum, not a binary, and it evolves as the codebase does.

**The right use:** Use atom/molecule/organism as vocabulary in communication and code review. "This organism is doing too much — the search functionality should be a molecule it composes, not logic embedded in the organism itself." That conversation is useful. The directory that enforces it is not.

Use feature-based or domain-based folder structure for organization. Use Atomic vocabulary for communication.

```
// Prefer this folder structure
src/
  ui/             # shared primitives (atoms + simple molecules)
  components/     # shared composed components (molecules + organisms)
  features/
    checkout/     # feature-scoped components, hooks, pages
    catalog/
  pages/          # route-level components

// Avoid this
src/
  atoms/
  molecules/
  organisms/
  templates/
  pages/
```

The first structure organizes by *relationship to the product*. The second organizes by *abstract classification*. When a new engineer asks "where does the ProductCard go?", the second structure produces a debate. The first produces an obvious answer: `components/ProductCard/`.

> **Check yourself:** A teammate proposes adding a `templates/` directory to enforce the separation between layout and content. What do you say, and what problem does that directory structure actually create?

---

## Gotchas

**Atoms that "know too much."** An atom that imports from a global store, makes a fetch call, or reads from a context that is product-specific is no longer an atom — it is tightly coupled to the application and cannot be reused or tested in isolation. Atoms should be pure in the UI sense: their output is determined entirely by their props.

**Organisms that fetch data.** Data fetching in organisms means every test for that organism requires mocking a network call or a context provider. It also means the organism cannot be used in a different data environment. Organisms should receive data via props. Lift fetching to the page or a dedicated container component.

**Molecules with too many responsibilities.** A molecule that solves two unrelated problems (a `Header` that is also a `SearchBar`) is not a molecule — it is an undifferentiated glob. If you cannot write one sentence describing what a molecule does, it needs to be split.

**Inconsistent export patterns.** If some components export as default, some as named, and some through barrel index files, consumers have no predictable pattern. Establish one convention (barrel + named export is common) and enforce it via linting.

**Storybook stories at the wrong level.** Stories for atoms and molecules are the most valuable — they document the building blocks in isolation. Stories for organisms are useful for integration. Stories for pages add limited value and are expensive to maintain (require all context providers, all mock data). Prioritize story coverage at atom and molecule levels.

---

## Interview Questions

**Q (High): A large feature file contains a 600-line component that handles layout, data fetching, and interaction logic for a product listing page. How do you decompose it using Atomic Design thinking?**

Answer: Start by reading the component for distinct visual concerns, not by counting lines. Identify atoms: the individual interactive elements (a sort button, a filter checkbox, a product image). Identify molecules: functional groupings (a `FilterGroup` that composes several filter checkboxes with a heading, a `SortBar` that composes sort buttons). Identify organisms: the `ProductGrid` that composes `ProductCard` organisms, the `FilterSidebar` that composes `FilterGroup` molecules. Extract the layout into a template: `ProductListingLayout` defines the grid + sidebar spatial relationship. Move data fetching to the page level: `ProductListingPage` fetches products and filters, passes them as props to the template's slot components. The result is six or seven small files, each independently testable, each with a single responsibility. The 600-line file becomes a page component of roughly 40 lines that orchestrates the others. The key discipline: the decomposition is not "extract to make files smaller" — it is "extract at the natural seams where responsibilities are distinct."

The trap: Candidates decompose by technical concern (hooks file, styles file, render file) rather than by UI abstraction level. That produces separated-but-coupled files rather than independently reusable components.

**Q (High): Why does putting data fetching in organism-level components create problems at scale?**

Answer: Three compounding problems. First, testability: every test for the organism must mock the data layer — either a network call, a query client, or a context provider. The component cannot be rendered in isolation without an elaborate test setup. Organisms are often the most complex components in the codebase; making them hard to test means they accumulate untested edge cases. Second, reuse: an organism that fetches its own data is coupled to a specific API shape and a specific data source. If a second team wants to use `ProductCard` with data from a different API endpoint, the fetching logic inside the organism blocks them. They fork or wrap. Third, caching and request deduplication: if two organisms on the same page independently fetch overlapping data, you get redundant network requests unless you add shared caching (React Query/SWR). But that caching infrastructure being hidden inside an organism means the page-level engineer cannot see or control it — it's a black box with network effects. The fix is to lift data fetching to the page or a dedicated container layer, pass data as props, and let organisms remain pure UI components.

The trap: Candidates say "React Query caches the request so duplicate fetches are fine." That addresses the network efficiency but misses the testability and reuse coupling problems.

**Q (High): Your team is debating whether to organize the codebase by Atomic Design levels (atoms/, molecules/, organisms/ directories) or by feature domain (features/catalog/, features/checkout/). Make the case for and against each.**

Answer: Atomic Design directories make the abstraction level explicit — a component in `atoms/` signals "pure, composable, no application coupling." They work well for design system packages where the consumer needs to understand component granularity to use them correctly. The problems: ambiguous classification (is Dropdown an atom or a molecule?), import path churn when a component's level changes during refactoring, and they do not reflect feature boundaries — so a feature engineer working on `checkout` touches five different directories for one feature's components. Feature-domain directories reflect how developers actually work: a team owning checkout owns everything under `features/checkout/`. Finding and modifying checkout components means one directory. New engineers understand the structure by understanding the product, not by learning an abstract taxonomy. Cross-cutting components go in `ui/` or `components/` at the root. The recommendation for most product teams: feature-based directories for application-specific components, a shared `ui/` package for primitives, and Atomic Design vocabulary for code review communication. Reserve Atomic directories for published design system packages where the taxonomy is itself documentation.

The trap: Candidates treat this as a binary either/or. The senior answer recognizes that Atomic vocabulary is always useful for communication; the question is whether directories are the right enforcement mechanism.

**Q (Medium): What is the purpose of the Templates level in Atomic Design, and why do many teams collapse it into Pages?**

Answer: Templates exist to separate layout decisions from content decisions. A template defines the spatial relationship between organisms — two-column layout, sticky header, max-width content container — without caring what content goes in each zone. This separation lets you reuse the same layout for different content (a blog post template used for every blog post, regardless of content) and lets you test layout without requiring real data. Teams collapse templates into pages for a practical reason: most layouts are not reused across many pages, so the template indirection adds an extra file and an extra mental model layer without proportional reuse benefit. A page that contains its own layout CSS alongside its content is one file to find instead of two. The trade-off is that if the layout does need to change for multiple pages, you are now updating it in multiple page files rather than one template. The signal to extract a template is when three or more pages share the same structural layout.

The trap: Candidates say templates are unnecessary without explaining what they solve. The level exists because layout and content coupling is a real maintenance problem at scale — the point is when that problem is significant enough to warrant the indirection.

**Q (Medium): Explain the colocation principle and how it relates to the Atomic Design hierarchy.**

Answer: Colocation means placing all artifacts that define a component — implementation, tests, stories, styles, type definitions — in the same directory as the component file, rather than in separate root-level directories (e.g., `__tests__/`, `styles/`, `stories/`). The Atomic Design connection is that colocation only works cleanly when components have clear boundaries — which the Atomic hierarchy enforces. An atom has a clear single responsibility, clear inputs (props), and clear outputs (rendered HTML). Colocating its test and story is natural because the test and story have the same scope as the component. An organism that contains too many concerns — data fetching, layout, interaction logic — makes colocation awkward because the test file has to cover too many scenarios and the story requires too much mock setup. When colocation feels painful, it is usually a signal that the component needs to be decomposed into smaller, more focused units. Colocation is therefore a feedback mechanism: difficulty colocating tests is an indicator of poor component decomposition.

The trap: Candidates describe colocation as a file organization preference. The senior insight is that colocation difficulty is a code quality signal — it surfaces hidden coupling and violation of single responsibility.

**Q (Low): What does it mean for atoms to be "pure" in the UI sense, and how does this connect to testability?**

Answer: UI purity for atoms means the rendered output is determined entirely by the component's props. No global state reads, no context subscriptions to application-specific providers, no side effects (fetch calls, subscriptions), no reliance on module-level singletons. An atom is a function from props to UI. This connects directly to testability: a pure atom can be rendered in a test with only `render(<Button variant="primary">Click me</Button>)` — no wrapping providers, no mocked modules, no fetch intercepts. The test can assert on what the user sees (the rendered output) without setting up an application environment. Impure atoms require exponentially more test setup as the application grows. They also cannot be accurately documented in Storybook without the full application context, and they cannot be published in a design system and consumed by a different application. Purity is not a nice-to-have for atoms — it is what makes the atom level of the hierarchy valuable.

The trap: Candidates conflate "pure component" in the React.memo sense (referential equality for re-renders) with UI purity. These are different concepts. A `React.memo` component can still read from global state; a UI-pure atom cannot.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Define all five Atomic Design levels and give a concrete example of each
- [ ] Explain why data fetching belongs at the page level, not inside organisms, with at least two distinct reasons
- [ ] Describe the key gotcha about treating Atomic Design as a folder structure and explain what directory structure you'd recommend instead
- [ ] Explain the colocation principle and articulate why difficulty colocating tests is a code quality signal
- [ ] Describe when a team should use all five levels vs. collapsing to three, and what signals drive that decision

---

*Next: Component Composition Patterns — with the Atomic Design hierarchy understood, the next question is how components within that hierarchy communicate and compose: HOCs, render props, hooks, and compound components are the main patterns.*
