# Lino Framework AI Coding Agent Instructions

This workspace contains **Lino**, a Django-based framework for building desktop-like web applications. It spans multiple repositories organized as independent Python packages with a **monorepo structure**.

## Quick Start for AI Agents

**Essential first steps when working in this codebase:**

1. **Monorepo structure**: Each top-level directory is an independent Python package with its own `pyproject.toml` and `tasks.py`
2. **Testing React**: `cd react/` → Run `BASE_SITE=noi npm run itest` (requires `python puppeteers/noi/manage.py prep` first)
3. **Testing Python**: `cd book/` → Run `pytest tests docs` (includes RST doctests)
4. **Demo sites**: Located in `react/puppeteers/{site}/` and `book/lino_book/projects/` - use `python manage.py prep` to initialize
5. **Key files**:
   - `react/lino_react/react/renderer.py` - Python-to-JSON serialization for React UI
   - `react/lino_react/react/components/App.jsx` - React app entry point
   - `lino/lino/core/site.py` - Site configuration and plugin loading
   - `lino/lino/core/plugin.py` - Plugin system base class
6. **Common pitfall**: Always use editable installs (`pip install -e .`) for local development to see cross-package changes immediately

## Project Overview

**Lino** is a plugin-based framework consisting of:
- **Core** (`lino/`): Base framework, renderers, models, actors, and plugin system
- **Extensions** (`lino_*` packages): Domain-specific applications (CRM, accounting, books, etc.)
  - Examples: `lino_noi` (ticketing), `lino_cosi` (accounting), `lino_avanti` (integration management), `lino_welfare` (social services)
- **Renderers**: Multiple UI options via separate packages:
  - `lino_react` (React/TypeScript, actively developed)
  - `lino.modlib.extjs` (ExtJS, legacy)
  - `lino_openui5` (OpenUI5, experimental)
- **Support packages**: 
  - `atelier` - Build task orchestration (invoke-based)
  - `rstgen`, `etgen` - Code/document generation utilities
  - `commondata.*` - Common data packages (countries, languages, etc.)
  - `xl/` - Extended library with reusable plugins

**Key principle**: Plugin-based architecture where features are optional, installable plugins loaded via Django's `INSTALLED_APPS`.

### Monorepo Structure
- Each top-level directory (`lino/`, `react/`, `book/`, `noi/`, `cosi/`, etc.) is an **independent Python package**
- Packages are interdependent: `lino` is core, `lino_react` depends on `lino`, applications depend on both
- Local development uses editable installs; each package has its own `pyproject.toml` and `tasks.py`
- Test suite primarily in `book/` package with doctest-based RST files
- Package naming convention:
  - `lino` - Core framework
  - `lino_<appname>` - Application packages (e.g., `lino_noi`, `lino_cosi`)
  - `commondata.<region>` - Regional data packages (e.g., `commondata.be`, `commondata.ee`)
  - Utility packages use standalone names (`atelier`, `etgen`, `rstgen`)


## Architecture Essentials

### Plugin System
- Every Lino feature is a `Plugin` class (see `lino/core/plugin.py`)
- Plugins are registered in `Site.get_installed_plugins()` and configured via `tasks.py`
- Plugin dependencies are declared via `needs_plugins` and `needed_by` attributes
- Renderers are registered as plugins with UI labels (e.g., `'react'`, `'extjs'`)

Example plugin registration (from `tasks.py`):
```python
from atelier.invlib import setup_from_tasks
ns = setup_from_tasks(globals(), 'lino_react', revision_control_system='git')
```

### Model & Actor Pattern
**Models** (`lino/core/model.py`): Extend Django's `Model` with Lino-specific methods:
- `get_detail_action()`, `allow_cascaded_delete`, `allow_cascaded_copy`
- `submit_insert`, `grid_post` (default actions)
- `active_fields`, `workflow_state_field` (UI configuration)

**Actors** (`lino/core/actors.py`): Represent queryable data sources (tables/grids):
- `get_handle()`: Returns UI-specific layout handle
- `get_actions()`: Returns bound actions available on the actor
- `_actions_list`: Defines available actions
- `actor_id`: Unique identifier used in URLs and caching

### Renderer Architecture
Renderers are framework-specific implementations that convert Python data to UI format:
- **Base**: `JsCacheRenderer` (JavaScript cache with py2js conversion)
- **React**: `Renderer` in `lino_react/react/renderer.py` (serializes to JSON for React frontend)
- **Key methods**: `elem2json()`, `panel2json()`, `actor2json()`, `action2json()`
- **Conversion**: `py2js()` + custom `py2js_converter()` handles Django models, choices, menus

React renderer serializes actor layouts as form panels stored in cache files (naming: `Lino_{actorId}_{usertype}_{language}.json`).

### React/TypeScript Frontend Architecture
- **Entry point**: `react/lino_react/react/components/` contains TSX/JSX components
- **Build system**: Webpack-based (see `react/package.json`), run `npm run build` for production
- **Type system**: TypeScript with comprehensive type definitions in `types.ts`
- **Action handling**: `ActionHandler.tsx` manages server communication via `runAction()` and `silentFetch()`
- **Dynamic imports**: `Base.ts` provides `ImportPool` pattern for code-splitting and lazy loading
  - Pattern: `RegisterImportPool({moduleName: import(/* webpackChunkName: "name_Component" */"module")})` 
  - Each component declares `static iPool` and `static requiredModules` for dependency resolution
  - `getExReady()` hook for function components, `DynDep` base class for class components
  - **Chunk naming**: Use descriptive names like `/* webpackChunkName: "lodash_SiteContext" */` to group module imports by component
  - Example from `SiteContext.jsx`:
    ```javascript
    const ex = {
        _: import(/* webpackChunkName: "lodash_SiteContext" */"lodash"),
        nc: import(/* webpackChunkName: "NavigationControl_SiteContext" */"./NavigationControl"),
    };
    RegisterImportPool(ex);
    ```
- **State management**: `window.App` global object stores site_data, settings, and registered panels (see `renderer.py:write_lino_js()`)
- **URL handling**: `URLParser` in `ActionHandler.tsx` converts query strings to typed objects, handles date/time/numeric sanitization

### React Context Architecture
**Navigation & State Controllers** (see `NavigationControl.js`, `SiteContext.jsx`):
- **NavigationControl.Context**: Main controller class managing URL navigation, state, and actor lifecycle
  - Created via `new Context({APP, rs, slave, next})` from `DynDep` base
  - Holds `actionHandler` (ActionHandler instance), `history` (History instance), and `dataContext` (DataContext instance)
  - Methods: `setRoot()`, `setParent()`, `build()`, `buildURLContext()`, `attachDataContext()`
  - Child contexts tracked via `children` object for slave grids/panels
- **URLContext components** (React.Context providers):
  - `RootURLContext`: Top-level context for dashboard/main view, initializes via `controller.setRoot(this)`
  - `URLContext`: Slave context for detail panels, slave grids - initialized with `inherit` or `path` prop
  - Pattern: Controller stored in `this.state.context.controller`, provides `value` to Context.Provider
- **DataContext**: Manages mutable form/grid data state
  - Tracks modifications via `modified[]` (single row) and `modifiedRows{}` (multi-row grid)
  - Methods: `set()`, `update()`, `clearMod()`, `isModified()`, `backupContext()`
  - Stores refs to all input leaves via `refStore.Leaves`, `refStore.slaveLeaves`
- **Initialization flow**:
  1. `App` creates root `NavigationControl.Context` with `ActionHandler` and `History`
  2. `RootURLContext` constructor calls `controller.setRoot(this)` to bind React component
  3. On mount, `attachDataContext(new DataContext({root, context, next}))` initializes data layer
  4. `controller.build()` fetches actor data and populates URL params
  5. Slave contexts created via `<URLContext path="..." onContextReady={...}/>` with parent reference

### Request & Action Flow
1. Client sends action request (HTTP/URL params) → `ActionRequest` created
2. `bound_action` executes action logic
3. Renderer calls `ar2js()` to serialize response to JavaScript
4. React uses `window.App.runAction()` with JSON-encoded status

## Developer Workflows

### Testing
**Python/Django Tests:**
- **Framework**: pytest with doctest support
- **Command**: `pytest tests docs` (includes RST doctests)
- **Configuration**: `pytest.ini` specifies doctest globs and forking
- **Example**: From `book/pytest.ini`:
  ```ini
  env = LINO_LOGLEVEL=INFO
  addopts = -v --forked --html=pytest_report.html --self-contained-html --doctest-glob="*.rst"
  ```
- **Forked execution**: `--forked` flag isolates each doctest in separate process (prevents state pollution)
- **HTML reports**: Tests generate `pytest_report.html` with detailed results

**React/TypeScript Tests:**
- **Framework**: Jest with Puppeteer for e2e tests, React Testing Library for unit tests
- **Configuration**: `jest.config.ts` with site-specific test matching (uses `testMatch` pattern)
- **Commands**:
  - `npm run test` - Run full Puppeteer e2e test suite (requires `prep` first)
  - `BASE_SITE=noi npm run itest` - Run tests for specific site (noi, avanti, etc.)
  - `BASE_SITE=noi BABEL=1 npm run itest` - Run React Testing Library unit tests (jsdom environment)
  - `RTL_LOG_NETWORK=1 BASE_SITE=noi BABEL=1 npm run itest` - Run RTL tests with live server logging
  - `npm run test:jsdom` - Alternative shortcut for `BASE_SITE=noi BABEL=1 jest -i`
- **Test setup**: Tests use `setupJEST.js` to spawn Django dev server and Puppeteer browser
- **Site preparation**: Run `python puppeteers/{site}/manage.py prep --noinput` before e2e tests
- **Environment variables**:
  - `BASE_SITE` - Selects demo site (noi, avanti, etc.) - REQUIRED for all test commands
  - `BABEL=1` - Switches to jsdom environment for RTL/unit tests (vs Puppeteer default)
  - `RTL_LOG_NETWORK=1` - Enables live server mode for RTL tests with MSW network logging
- **Test file naming convention**: 
  - `__tests__/{site}/*.ts` - Puppeteer tests (require full browser)
  - `__tests__/{site}/*.tsx` - RTL tests (use jsdom, activated when `BABEL=1` is set)
  - `__tests__/{site}/DOMTexts.ts` - Pure jsdom tests (no React rendering)

**Test File Patterns:**
- **Puppeteer tests** (`URLContext.ts`): Full browser automation with real backend
  - Use `page.evaluate()` to access `window.App` and execute client-side code
  - Use `page.locator()` and `page.$()` for DOM selection
  - Use `page.waitForNetworkIdle()` for async operations
  - Requires running Django dev server
  - Example: `react/lino_react/react/components/__tests__/noi/URLContext.ts`
  
- **jsdom tests** (`DOMTexts.ts`): Lightweight DOM simulation without browser
  - Use JSDOM to create virtual browser environment
  - Mock `window.App` and all backend interactions
  - Direct DOM manipulation via `document` API
  - Faster execution, no backend required
  - Example: `react/lino_react/react/components/__tests__/noi/DOMTexts.ts`
  
- **React Testing Library tests** (`functional.tsx`): Component-focused unit tests
  - Use `@testing-library/react` with `render()`, `screen`, and `userEvent`
  - Emphasize accessibility and user-centric queries (`getByRole`, `getByLabelText`)
  - Use `waitFor()` for async assertions
  - Network mocking via MSW (Mock Service Worker) - see RTL Network Testing below
  - Best practices: test behavior, not implementation; use semantic queries
  - Example: `react/lino_react/react/components/__tests__/noi/functional.tsx`

**RTL Network Testing with MSW:**
- **Purpose**: Mock network requests in RTL tests using MSW (Mock Service Worker)
- **Cache format**: NDJSON (newline-delimited JSON) for memory efficiency
  - Files: `network-testing/logs/network-log-rtl-{site}.ndjson`
  - Each line is a separate JSON object with pre-computed `cacheKey` field
  - Streamed line-by-line on-demand with early exit on match (no memory loading)
  - String search for `cacheKey` before parsing (50-100x faster than parse+compute)
- **Two modes**:
  1. **Live server mode** (`RTL_LOG_NETWORK=1`):
     - Command: `RTL_LOG_NETWORK=1 BASE_SITE=noi BABEL=1 npm run itest`
     - Starts Django server on **port 3001** (requires `manage.py prep --noinput` first)
     - MSW intercepts requests and proxies them to live server using Node's `http` module
     - Responses cached in `network-testing/logs/network-log-rtl-{site}.ndjson`
     - **Captures authentication**: Session cookies and login flows automatically saved
     - **Use case**: Updating cached responses when backend changes
     - **CRITICAL**: MSW handler checks for port 3001 requests and uses `passthrough()` to prevent infinite recursion
  2. **Cached mode** (default):
     - Command: `BASE_SITE=noi BABEL=1 npm run itest`
     - MSW loads cached responses from `network-testing/logs/network-log-rtl-{site}.ndjson`
     - No server needed, tests run offline and fast
     - **Replays authentication**: Session cookies and auth state restored from cache
     - **Use case**: CI/CD, local testing without backend setup
- **Authentication**: Demo credentials (username='robin', password='1234') from `manage.py prep`
- **Implementation**: `react/lino_react/react/testSetup/mswHandlers.ts`
- **Setup**: MSW server initialized in `setupTests.ts` for all RTL tests
- **Helpers**: 
  - `rtlTestHelpers.ts` - Auth utilities, `waitForNetworkIdle()`, test credentials
  - `getPendingRequestCount()` - Tracks active MSW requests for test synchronization
- **Key files**:
  - `setupJEST.js` - Starts Django server on port 3001 when `RTL_LOG_NETWORK=1`
  - `teardownJEST.js` - Stops Django server after tests
  - `RTL_AUTH_GUIDE.md` - Complete authentication documentation
  - `network-testing/tools/download_cache_from_gitlab.sh` - Downloads .ndjson cache from GitLab Package Registry for CI
  - `network-testing/tools/publish_cache_to_gitlab.sh` - Publishes .ndjson cache to GitLab Package Registry
- **Benefits**: Reliable network mocking, automatic cookie handling, request interception, response caching, offline testing, memory-efficient on-demand streaming with pre-computed cache keys
- **CI Integration**: GitLab CI downloads .ndjson cache files via `.gitlab-ci.yml` download script before running tests

### Building & Tasks
- Uses **atelier** for build orchestration (see `atelier/atelier/invlib/`)
- Each package has `tasks.py` with `setup_from_tasks(globals(), 'package_name', ...)` pattern
- Key invoke tasks (available via `inv <task>`):
  - `bd` - build documentation (Sphinx)
  - `test` - run tests (pytest + doctests)
  - `sdist` - create source distribution
  - `demo` - prepare demo projects for testing
- **React-specific**: 
  - `npm run build` - Production build (minified, optimized)
  - `npm run dev` - Development build (faster, source maps)
  - `npm run watch` - Auto-rebuild on file changes
  - All builds require `NODE_OPTIONS='--max-old-space-size=8192'` (handled in package.json)
- **Environment**: `LINO_LOGLEVEL` controls logging verbosity (DEBUG, INFO, WARNING, ERROR)

### Documentation
- **Format**: ReStructuredText (Sphinx-based)
- **Structure**: `docs/` per package with conf.py, index.rst
- **Code examples**: Embedded doctests in RST allow validation during builds
- **Intersphinx**: Cross-references between lino-framework projects

### Logging
- Framework logger: `from lino import logger`
- Log level configured via `LINO_LOGLEVEL` environment variable
- Separate controls: `LINO_FILE_LOGLEVEL`, `LINO_SQL_LOGLEVEL` for file/SQL logging
- Linod socket handler available for distributed logging

### Python Environment Management
- **CRITICAL**: Always use editable installs for local development: `pip install -e .` in each package
- Monorepo workflow: Changes to `lino` are immediately available to `lino_react`, `noi`, etc.
- Virtual environments recommended: Create one per demo site or shared across all packages
- Dependencies chain: Core → Extensions → Applications (follow this when installing)


### Django Management Commands
Lino extends Django's management commands (in `lino/management/commands/`):
- `prep` - Prepare demo database (equivalent to `initdb --noinput` + `makeui`)
- `initdb` - Initialize database with demo data (creates fixtures, runs migrations)
- `makeui` - Build UI cache files for renderers (React JSON panels, etc.)
- `run` - Run development server
- `dump2py` - Export database to Python fixtures
- `diag` - Display system diagnostics
- `show` - Display database model information
- `buildcache` - Force rebuild of site cache

**Demo Site Workflow**:
```bash
# Standard demo setup sequence:
python manage.py prep --noinput
# Or manually:
python manage.py initdb --noinput
python manage.py makeui
python manage.py runserver
```

## Code Patterns & Conventions

### Object-to-Dict Serialization
Repeatedly used pattern for converting Python objects to JSON/dict:
```python
from lino.utils.jsgen import obj2dict, py2js
# Extract multiple attributes at once:
result.update(obj2dict(actor, "preview_limit hide_navigator max_render_depth"))
```
This creates dict entries: `{'preview_limit': value, 'hide_navigator': value, ...}`

### Choosers & Dependent Fields
Dynamic field value filtering:
- Model defines `_choosers_dict`: Maps field name → chooser function
- Chooser function filters options based on context fields
- React renderer includes in JSON: `choosers_dict: {field_name: [context_field_names]}`

### Layout Elements & Forms
Recursive JSON serialization of UI layouts:
- `LayoutHandle` → `elem2json()` converts tree of `LayoutElement` objects
- Composite elements store child items recursively
- Slave tables (related grids) stored as keyed dict entries within parent element
- Form panels cached by `_formpanel_name` for reuse

### Action Definitions
Actions serialized once globally to reduce JSON size:
- React: All actions stored in `actions_def` (cached once)
- Actors reference actions by `full_name`
- Hotkeys, combo groups, toolbar visibility all metadata-driven

### Version Handling
- Framework version: `from lino import __version__` (currently `25.12.4`)
- Serialized in all cache responses via `constants.URL_PARAM_LINO_VERSION`
- Used for cache invalidation
- **GitLab CI**: RTL cache versioning controlled by `RTL_CACHE_VERSION` variable in `.gitlab-ci.yml`

### Code Generation Patterns
Common patterns for generated code (seen in `commondata.*` packages):
```python
# Generated file header
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
# code generated by commondata {version} make_code.py
# fmt: off

# Content here...

# end of generated code
```
- Code generators in `cd/make_code.py` query Wikidata (WDQS) for country/city data
- Generated files use `# fmt: off` to prevent auto-formatting
- `commondata.*` packages populate regional data via `populate()` functions

## File Organization Patterns

### Package Structure
```
package_name/
  pyproject.toml          # Dependencies, metadata
  tasks.py                # Build configuration
  README.rst              # Overview
  package_name/           # Python package
    __init__.py           # Version, module docstring
    core/                 # Core modules (if applicable)
    models.py             # Data models
    views.py              # Views/URLs
  docs/                   # Sphinx documentation
  tests/                  # Pytest test suite
```

### Demo Projects & Settings
Demo projects follow a standard pattern (e.g., `react/puppeteers/noi/`, `book/lino_book/projects/noi1r/`):
```python
# settings.py
from parent_app.settings import *

class Site(Site):
    default_ui = 'lino_react.react'  # Select renderer
    title = "Demo Site Title"
    languages = "en de fr"           # Enabled languages
    is_demo_site = True              # Marks as demo (affects caching, debug output)

SITE = Site(globals())
DEBUG = True
```

**Key Site attributes**:
- `default_ui` - Renderer plugin ('lino_react.react', 'lino.modlib.extjs', etc.)
- `get_installed_plugins()` - Returns list of enabled plugins (respects `needed_by` dependencies)
- `get_plugin_configs()` - Yields plugin configuration tuples
- `is_demo_site` - Boolean flag for demo environments (pretty-printed JSON, verbose logging)

### Imports & Dependencies
- **Django**: Core ORM, settings, signals
- **deepmerge**: Used for merging status dicts in React renderer
- **etgen**: Code generation utilities
- **atelier**: Build/development task orchestration
- **lxml, odfpy, beautifulsoup4**: Document parsing/generation

## Critical Patterns to Preserve

1. **Plugin registration order matters**: `needed_by` dependencies must be respected (see `NeededByTest` in book/tests)
2. **Settings.SITE access**: All renderers access `settings.SITE` for global configuration (e.g., `Site.setup_model_spec()`)
3. **User profile checks**: Permissions validated via `get_user_profile()` and required roles
4. **Lazy translation**: Use `gettext_lazy()` for all user-visible strings
5. **JSON serialization safety**: Custom `py2js_converter()` prevents circular refs, handles Django types
6. **Cache file naming**: Incorporates actor_id, user_type, language for multi-user/language support
7. **Abstract models**: Use `Site.is_abstract_model()` to check if a model is overridden by `extends_models` in plugin hierarchy
8. **Virtual fields**: Register via `Site.register_virtual_field()` - resolved after startup via `resolve_virtual_fields()`
9. **React fetch pattern**: Always use `ActionHandler.silentFetch()` for AJAX calls to maintain consistent error handling

## Debugging Tips

- Set `LINO_LOGLEVEL=DEBUG` for verbose output (or `INFO`, `WARNING`, `ERROR`)
- Set `LINO_FILE_LOGLEVEL` and `LINO_SQL_LOGLEVEL` for separate control of file/SQL logging
- React renderer uses `settings.DEBUG` to pretty-print JSON for inspection
- Renderer `serialise_js_code` flag controls whether JS code is escaped for JSON
- TestCase framework: `lino.utils.pythontest.TestCase` for Django-aware tests
- Look for `20` + date format comments (e.g., `# 20250514`) for recent changes/issues
- **Cache debugging**: Delete `static_root/` to force cache rebuild; use `build_site_cache(force=True)` in code
- **React DevTools**: Use browser developer tools to inspect `window.App` state and registered components
- **Pytest forking**: Tests use `--forked` flag (see `book/pytest.ini`) to isolate each doctest in separate process
- **Webpack debugging**: Use `npm run debug` for unminified builds with inline source maps

### Common Issues & Solutions
- **Memory errors during builds**: React builds require `NODE_OPTIONS='--max-old-space-size=8192'` (set in package.json)
- **Cache stale/corrupt**: Run `python manage.py buildcache` or delete cache directory
- **Test failures in CI**: Ensure `prep --noinput` runs before e2e tests; check `BASE_SITE` environment variable
- **Import errors in tests**: Use editable installs (`pip install -e .`) for monorepo development
- **MSW infinite recursion**: When proxying to Django server in RTL tests, MSW intercepts its own fetch requests - use port check (`if (urlObj.port === '3001') return passthrough()`) to prevent recursion
- **RTL network idle**: Use `waitForNetworkIdle()` from `rtlTestHelpers.ts` to wait for MSW requests to complete before assertions

## Critical MSW Implementation Patterns

### Infinite Recursion Prevention
**Problem**: MSW intercepts ALL network requests, including fetch calls made by the MSW handler itself to proxy to Django.

**Solution** (from `mswHandlers.ts` line 282-289):
```typescript
// CRITICAL: Check if this is a direct request to port 3001
// These are proxy requests we're making with fetch, and MSW intercepts them
// We must passthrough immediately to prevent infinite recursion
const urlObj = new URL(url);
if (urlObj.port === '3001') {
    return passthrough();
}
```

### Request Tracking for Test Synchronization
**Pattern**: Track pending MSW requests to enable test helpers that wait for network idle state.

**Implementation** (from `mswHandlers.ts` lines 28-44):
```typescript
let pendingMSWRequests = 0;

export function getPendingRequestCount(): number {
    return pendingMSWRequests;
}

// In handler:
incrementPendingRequests();
try {
    // ... handle request
} finally {
    decrementPendingRequests();
}
```

**Usage in tests**:
```typescript
import { waitForNetworkIdle } from '../../testSetup/rtlTestHelpers';

await waitForNetworkIdle(); // Waits until getPendingRequestCount() === 0
```

### Cache Key Optimization
**Pattern**: Pre-compute cache keys during recording for fast string-search lookup during playback.

**Implementation**:
```typescript
interface NetworkLogEntry {
    cacheKey: string;  // Pre-computed: "METHOD:/path?params|body:params"
    timestamp: string;
    request: {...};
    response: {...};
}

// During recording (RTL_LOG_NETWORK=1)
logRequest(method, url, headers, body, response) {
    const cacheKey = this.makeKey(method, url, body, userType);
    fs.appendFileSync(logPath, JSON.stringify({cacheKey, ...}) + '\n');
}

// During playback (cached mode)
findCachedResponse(method, url, body, userType) {
    const targetKey = this.makeKey(method, url, body, userType);
    // String search is 50-100x faster than JSON parse
    if (line.includes(`"cacheKey":"${targetKey}"`)) {
        return JSON.parse(line);  // Only parse matching line
    }
}
```

**Benefits**: 
- Fast lookup: String search before JSON parse
- Early exit: Return immediately on match
- Minimal memory: Only 64KB buffer + current line
- Consistent keys: Parameters sorted alphabetically

### Node HTTP Module for Bypassing jsdom Restrictions
**Why**: jsdom's fetch/XMLHttpRequest have CORS restrictions that prevent proxying to localhost.

**Solution** (from `mswHandlers.ts` lines 330-380):
```typescript
import * as httpModule from 'http';

// Use Node's http module directly to bypass jsdom
const response = await new Promise<{...}>((resolve, reject) => {
    const req = httpModule.request(options, (res) => {
        let data = '';
        res.on('data', (chunk) => { data += chunk; });
        res.on('end', () => resolve({...}));
    });
    req.on('error', reject);
    if (bodyForProxy) req.write(bodyForProxy, 'utf8');
    req.end();
});
```

## Accounting Plugins Architecture

Lino has a comprehensive **accounting ecosystem** with 15+ plugins in `lino/` and `xl/` repositories. Understanding the plugin hierarchy is critical for working with financial features.

### Core Accounting Plugins

**`lino_xl.lib.accounting`** - General Ledger (foundation):
- **Key models**:
  - `Account`: Chart of accounts with structured refs, VAT classes, common accounts
  - `Journal`: Groups vouchers by type (JournalGroups: sales/purchases/wages/financial/VAT/misc)
  - `Voucher`: Base class for all accounting documents (invoices, statements, orders)
  - `Movement`: Individual ledger entries (debit/credit) - implements double-entry bookkeeping
  - `CommonAccounts`: Choicelist for predefined account types (customers, suppliers, net_income_loss)
- **Settings**: `sales_method` ('direct'|'delivery'|'pos'), `has_payment_methods`, `has_purchases`, `currency_symbol`
- **Dependencies**: `lino.modlib.periods`, `lino.modlib.weasyprint`, `lino_xl.lib.xl`, `lino.modlib.uploads`
- **Files**: `models.py` (1200+ lines), `choicelists.py` (DC, JournalGroups, CommonAccounts), `mixins.py` (LedgerRegistrable, ProjectRelated, PaymentRelated)

**`lino.modlib.periods`** - Accounting Periods:
- Models: `StoredYear`, `StoredPeriod` for fiscal year management
- Mixins: `PeriodRange`, `PeriodRangeObservable` for date-range filtering
- Required by all accounting plugins

**`lino_xl.lib.ledgers`** - Multi-Ledger Support:
- Separates accounting scope into multiple ledgers (multi-company scenarios)
- Each ledger has its own journals and accounts
- Foreign key `ledger` on Journal and Account models

**`lino_xl.lib.finan`** - Financial Vouchers:
- Models: `BankStatement`, `PaymentOrder`, `JournalEntry` (manual entries)
- Used for bank reconciliation and payment management
- Setting: `suggest_future_vouchers` for date suggestions

### Trading & Invoicing

**`lino_xl.lib.trading`** - Sales & Invoices:
- Core trading functionality: invoices, invoice items, trading rules
- Dependencies: `lino.modlib.memo`, `lino_xl.lib.products`, `lino_xl.lib.vat`
- Settings: `print_items_table`, `items_column_names`, `subtotal_demo`
- Models: `VatProductInvoice`, `InvoiceItem`, `TradingRule`, `PaperType`

**`lino_xl.lib.invoicing`** - Invoice Generation:
- Automatically generates invoices from invoice generators
- Mixin: `InvoiceGenerator` for models that can generate invoices (time entries, subscriptions, etc.)
- Setting: `order_model` to link with order system
- Dependency: `lino_xl.lib.trading`

### VAT Plugins

**`lino_xl.lib.vat`** - VAT Core:
- Choicelists: `VatRegimes` (normal/exempt/intra_community/outside_eu), `VatClasses` (services/goods/real_estate), `VatColumns` (rate columns)
- Models: `VatRule` (tax calculation), `VatAccountInvoice`, `VatProductInvoice`
- Settings: `default_vat_regime='normal'`, `default_vat_class='services'`, `item_vat`, `unit_price_decpos=4`, `declaration_plugin`, `eu_country_codes`
- Method: `get_vat_class(tt, item)` - override to customize VAT class selection

**`lino_xl.lib.vatless`** - VAT-less Invoicing:
- Simplified invoicing for organizations without VAT obligation
- Dependencies: `lino_xl.lib.countries`, `lino_xl.lib.accounting`

**Country-Specific VAT Declarations**:
- `lino_xl.lib.bevat` - Belgian VAT
- `lino_xl.lib.bevats` - Belgian simplified VAT
- `lino_xl.lib.eevat` - Estonian VAT
- `lino_xl.lib.bdvat` - Bangladeshi VAT
- All depend on `lino_xl.lib.vat`, provide `Declarations` and `DeclarationFields` models

### Analytical & Reporting

**`lino_xl.lib.ana`** - Analytical Accounting:
- Cost center/project accounting (track expenses by department/project)
- Model: `Account` (analytical accounts separate from general ledger)
- Setting: `ref_length=4` for account reference width
- Dependency: `lino_xl.lib.accounting`

**`lino_xl.lib.sheets`** - Accounting Statements:
- Balance sheet and income statement generation using "sheet items"
- Models: `Item` (template items), `Report`, `CommonItems` (standard balance/P&L items)
- Tables: `AccountEntries`, `AnaAccountEntries`, `PartnerEntries`, `ItemEntries` (different aggregation views)
- Setting: `item_ref_width=4` for reference field display
- Dependency: `lino_xl.lib.accounting`

### Payment & Banking

**`lino_xl.lib.sepa`** - SEPA Payment System:
- European bank account management with IBAN validation
- Model: `Account` with IBAN field
- Feature: `uppercasetextfield.js` for automatic IBAN uppercase conversion
- Optional dependency on `lino_xl.lib.accounting`

**`lino_xl.lib.peppol`** - PEPPOL E-Invoicing:
- European e-invoicing standard (XML format)
- Settings: `outbound_model='trading.VatProductInvoice'`, `inbound_model='vat.VatAccountInvoice'`
- Mixin: `PeppolJournal` for journals that support PEPPOL
- Dependency: `lino_xl.lib.vat`

### Plugin Dependency Hierarchy
```
lino.modlib.periods (foundation)
    ↓
lino_xl.lib.accounting (core ledger)
    ↓
├── lino_xl.lib.ledgers (multi-ledger)
├── lino_xl.lib.finan (financial vouchers)
├── lino_xl.lib.ana (analytical)
├── lino_xl.lib.sheets (statements)
├── lino_xl.lib.vat (VAT core)
│   ↓
│   ├── bevat, bevats, eevat, bdvat (country VAT)
│   ├── lino_xl.lib.vatless (VAT-exempt)
│   └── lino_xl.lib.trading (sales)
│       ↓
│       └── lino_xl.lib.invoicing (generation)
└── lino_xl.lib.sepa (payments)
```

### Key Accounting Patterns

**Double-Entry Bookkeeping**:
- `Movement` records have `amount` field (positive=credit, negative=debit)
- `DC` choicelist: `debit.normalized_amount(n) = -n`, `credit.normalized_amount(n) = n`
- Virtual fields: `debit` (if amount < 0), `credit` (if amount > 0)
- Constraint: Sum of movements in a voucher must equal zero

**Voucher State Machine**:
```python
# State transitions via LedgerRegistrable mixin
draft → register_voucher() → registered  # Creates movements
registered → deregister_voucher() → draft  # Deletes movements
```
- `ToggleState` action (Ctrl+X) switches between draft/registered
- Movements only created when state = 'registered'
- `get_wanted_movements(ar)` method generates movement list

**Journal Configuration**:
- `trade_type`: TradeTypes.sales (credit) or TradeTypes.purchases (debit)
- `voucher_type`: VoucherTypes choicelist (links to voucher model)
- `journal_group`: Organizes journals in menu (sales/purchases/wages/financial/VAT/misc)
- `dc`: Primary booking direction (debit or credit)
- `yearly_numbering`: Reset voucher numbers each year
- `make_ledger_movements`, `make_storage_movements`: Control movement generation

**Common Account Pointing**:
- `CommonAccounts` is a `PointingChoice` choicelist
- Each item can point to one `Account` instance per ledger
- Pattern: `CommonAccounts.customers.get_object(ledger=...)` returns the customer account
- Automatic account creation via `create_object(**kwargs)`

**Clearable Accounts**:
- Partner accounts (customers/suppliers) are clearable
- `Movement.match` field groups related movements for clearing
- `cleared` boolean marks when balance is settled
- Example: Invoice movements matched with payment movements

**VAT Calculation Flow**:
1. `VatRule.get_vat_rule(tt, vat_class, vat_regime)` finds applicable rule
2. Rule specifies which `VatColumn` to use
3. Column defines VAT rate percentage
4. Invoice calculates: `vat_amount = base_amount * vat_rate`
5. Movements created for both base and VAT amounts

**Movement Generation Example**:
```python
# In Voucher.get_wanted_movements():
# 1. Debit customer account for invoice total
yield Movement(account=customers_account, partner=customer, 
               amount=-total, dc=DC.debit)
# 2. Credit sales account for base amount
yield Movement(account=sales_account, amount=base, dc=DC.credit)
# 3. Credit VAT account for tax amount
yield Movement(account=vat_account, amount=vat, dc=DC.credit)
```

### Critical Files to Study
- `xl/lino_xl/lib/accounting/models.py` - Core models (1234 lines)
- `xl/lino_xl/lib/accounting/choicelists.py` - DC, JournalGroups, CommonAccounts (551 lines)
- `xl/lino_xl/lib/accounting/mixins.py` - LedgerRegistrable, ProjectRelated, Payable (535 lines)
- `xl/lino_xl/lib/vat/choicelists.py` - VatRegimes, VatClasses, VatColumns, VatRules
- `xl/lino_xl/lib/trading/models.py` - Invoice and item models
- `lino/lino/modlib/periods/models.py` - Fiscal year/period management

## Core Architecture: Actors, Actions, Elements, and Store

Lino's architecture combines Django models with a sophisticated abstraction layer that powers the UI. Understanding how **Actors**, **Actions**, **Elements**, and **Store** work together is essential for Lino development.

### Component Overview

**Actor** (`lino/lino/core/actors.py`):
- Base class for all data-representing components (Tables, Frames, VirtualTables)
- Provides queryable interface to data sources
- Key attributes:
  - `model`: Django model or None (for virtual tables)
  - `detail_layout`, `insert_layout`, `card_layout`: Layout definitions
  - `default_action`: Action shown when actor is accessed
  - `actor_id`: Unique identifier used in URLs and caching
  - `required_roles`: Permission requirements
- Key methods:
  - `get_handle()`: Returns UI-specific layout handle
  - `get_actions()`: Returns available bound actions
  - `request(**kwargs)`: Creates ActionRequest for this actor
- Metaclass: `ActorMetaClass` - automatically registers actors during import

**Table** (`lino/lino/core/dbtables.py`):
- Most common Actor subclass - represents Django Model querysets
- Attributes:
  - `model`: Django Model class
  - `master`, `master_key`: For slave tables (one-to-many relationships)
  - `filter`, `exclude`, `order_by`: Django QuerySet parameters
  - `column_names`: Which fields/columns to display
  - `use_as_default_table`: Whether this becomes model's default table
  - `live_panel_update`: Whether front end should auto-refresh (React only)
- Methods:
  - `get_request_queryset(ar)`: Returns filtered QuerySet for request
  - `add_quick_search_filter(qs, text)`: Applies quick search
  - `get_row_by_pk(ar, pk)`: Fetches single row by primary key

**Action** (`lino/lino/core/base.py`, `lino/lino/core/actions.py`):
- Represents an operation that can be performed on data
- Base class for all user-invokable operations
- Key attributes:
  - `label`: User-visible name
  - `action_name`: Programmatic identifier
  - `parameters`: Dict of action-specific parameters
  - `required_roles`: Permission requirements
  - `opens_a_window`: Whether action shows dialog/window
  - `window_type`: WINDOW_TYPE_DETAIL | WINDOW_TYPE_TABLE | etc.
  - `callable_from`: Where action can be invoked ('t'=table, 'd'=detail, 'td'=both)
  - `hotkey`: Keyboard shortcut
- Key methods:
  - `run_from_ui(ar, **kw)`: Execute action from UI (updates `ar.response`)
  - `run_from_code(ar, *args, **kw)`: Execute action programmatically
  - `get_status(ar)`: Return action state/availability
- Common action types:
  - `ShowTable`: Opens table view (grid)
  - `ShowDetail`: Opens detail view (form)
  - `ShowInsert`: Opens insert form for new record
  - `DeleteSelected`: Deletes selected rows
  - Custom actions: Define via `@dd.action()` decorator on model methods

**BoundAction** (`lino/lino/core/boundaction.py`):
- Links an Action to an Actor
- Created via `actor.get_action_by_name(action_name)`
- Pattern: `bound_action = BoundAction(actor, action)`
- Used in ActionRequest to execute actions in context

**LayoutElement** (`lino/lino/core/elems.py`):
- Base class for all UI widgets/components
- Represents individual fields, panels, buttons in layouts
- Key attributes:
  - `name`: Element identifier
  - `editable`: Whether user can modify
  - `required_roles`: Permission requirements
  - `parent`: Container element (Panel)
  - `layout_handle`: Parent layout handle
- Subclasses:
  - `FieldElement`: Wraps Django field for UI
  - `Panel`: Container for multiple elements (HPanel, VPanel, TabPanel)
  - `ButtonElement`: Action button
  - `Spacer`: Empty space
  - `GridColumn`: Column in table view

**LayoutHandle** (`lino/lino/core/layouts.py`):
- Analyzes and holds metadata for a layout
- One handle per layout per front end (React, ExtJS, etc.)
- Created during startup by renderer
- Attributes:
  - `layout`: BaseLayout instance being handled
  - `ui`: Front end plugin (React, ExtJS)
  - `main`: Root element of layout
  - `_data_elems`: List of data-bound elements
  - `_names`: Dict mapping element names to instances

**FormLayout/DetailLayout/InsertLayout** (`lino/lino/core/layouts.py`):
- Defines structure of detail/insert forms
- String-based DSL for layout definition:
  ```python
  detail_layout = """
  name address
  phone email
  remarks
  """
  ```
- Parsed into tree of LayoutElements during startup
- Multiple syntax options: strings, multi-line strings, LayoutPanels

**Store** (`lino/lino/core/store.py`):
- Collection of StoreFields for an actor
- Manages data serialization/deserialization between Python and front end
- Instantiated during kernel startup for each actor
- Attributes:
  - `actor`: Actor this store represents
  - `rh`: Request handle
  - `all_fields`, `grid_fields`, `detail_fields`, `card_fields`: Field lists per view
- Methods:
  - `collect_fields(fields_list, layout_handle)`: Gathers fields from layout
  - `pv2dict(ar, pv)`: Converts param values to dict (for parameters)
- **StoreField** (`lino/lino/core/storefields.py`):
  - Individual field atomizer - handles one field's serialization
  - Subclasses: `VirtStoreField`, `RowClassStoreField`, `DisableEditingStoreField`

### Request Flow Architecture

**ActionRequest** (`lino/lino/core/requests.py`):
- Central object for handling user requests
- Combines Actor, Action, and data context
- Key attributes:
  - `actor`: The Actor being queried
  - `bound_action`: BoundAction being executed
  - `renderer`: UI renderer (React, ExtJS, text)
  - `selected_rows`: Currently selected/active rows
  - `master_instance`: Parent record for slave tables
  - `param_values`: Filter/search parameters
  - `rqdata`: Raw request data from HTTP
  - `response`: Response dict to return to client
- Key methods:
  - `get_data_iterator()`: Returns rows from actor
  - `get_total_count()`: Total rows (ignoring pagination)
  - `spawn(**kw)`: Creates child request with modified params
  - `to_rst()`: Renders request as reStructuredText (for docs)
  - `render()`: Renders request using current renderer

### Data Flow: From Django Model to UI

1. **Model Layer** (Django):
   - Django Model defines fields, methods, constraints
   - Lino `Model` mixin adds: `get_detail_action()`, workflow methods, display fields
   
2. **Actor Layer** (Lino abstraction):
   - Table/Frame wraps model as queryable data source
   - Defines available actions, layouts, permissions
   - `actor_id` used for URL routing and caching
   
3. **Layout Layer** (UI structure):
   - FormLayout/DetailLayout parsed into LayoutHandle
   - LayoutHandle creates tree of LayoutElements
   - Each LayoutElement maps to Django field or virtual field
   
4. **Store Layer** (serialization):
   - Store collects all fields from layouts
   - StoreFields handle individual field conversion
   - `py2js()` serializes Python values to JSON for front end
   
5. **Action Layer** (operations):
   - Actions registered on Actor
   - BoundAction links action to actor
   - ActionRequest executes action with context
   
6. **Rendering Layer** (front end):
   - Renderer (React, ExtJS) converts to UI-specific format
   - React: JSON sent to `window.App`, rendered as components
   - ExtJS: JavaScript code generated for GridPanel/FormPanel
   
7. **Response Layer** (back to client):
   - ActionRequest.response populated during execution
   - Renderer serializes response: `ar2js()`, `elem2json()`, etc.
   - Front end receives JSON and updates UI

### Example Flow: Opening Detail Window

```python
# 1. User clicks detail button for row pk=5
# Front end sends: GET /api/contacts/Persons/5

# 2. URL routing finds actor and creates request
actor = contacts.Persons  # Table instance
action = actor.detail_action  # ShowDetail action
ar = actor.request(request=http_request, action=action, selected_rows=[obj])

# 3. ActionRequest initialized
ar.actor = contacts.Persons
ar.bound_action = BoundAction(actor, ShowDetail)
ar.selected_rows = [Person.objects.get(pk=5)]

# 4. Layout prepared
layout = actor.detail_layout  # DetailLayout instance
lh = layout.get_layout_handle(renderer.plugin)  # LayoutHandle
store = actor.get_handle(ar).store  # Store with StoreFields

# 5. Action executed
ar.bound_action.action.run_from_ui(ar)

# 6. Response built
ar.response = {
    'data_record': store.row2dict(ar, ar.selected_rows[0]),
    'title': str(ar.selected_rows[0]),
    'param_values': lh.params_store.pv2dict(ar, ar.param_values),
    ...
}

# 7. Serialized and sent
renderer.ar2js(ar)  # Converts response to JSON
# Front end receives and renders DetailPanel
```

### Key Patterns

**Actor Registration**:
- Actors registered automatically at import via `ActorMetaClass`
- Stored in global `actors_dict` and `actors_list`
- Access via `rt.models.app_label.ActorName` or `rt.actors.app_label.ActorName`

**Layout Definition**:
- String-based DSL: `"field1 field2:20 field3"`
- Panel syntax: `"panel1:vflex field1 field2"`
- Element options: `_element_options = {'field_name': {'width': 30}}`

**Virtual Fields**:
- Defined via `@dd.virtualfield()` decorator on model
- Display-only (not stored in database)
- Can have dependencies: `@dd.virtualfield(dd.ForeignKey('contacts.Person'))`
- Access like normal fields in layouts and actions

**Master-Slave Relationships**:
- Slave tables filtered by `master_instance`
- Pattern: `class EntriesByVoucher(dd.Table): master = 'accounting.Voucher'`
- Master key automatically detected from foreign key field

**Permission System**:
- `required_roles`: Set of role classes required
- Checked at multiple levels: Actor, Action, Row
- Pattern: `required_roles = dd.login_required(LedgerStaff)`

### Critical Files to Study
- `lino/lino/core/actors.py` - Actor base class (2050 lines)
- `lino/lino/core/dbtables.py` - Table implementation (794 lines)
- `lino/lino/core/actions.py` - Action base class (691 lines)
- `lino/lino/core/base.py` - Action class definition (439 lines)
- `lino/lino/core/elems.py` - Layout elements (3066 lines)
- `lino/lino/core/layouts.py` - Layout definitions (833 lines)
- `lino/lino/core/store.py` - Store and serialization (464 lines)
- `lino/lino/core/requests.py` - ActionRequest (2660 lines)
- `lino/lino/core/model.py` - Model mixin (1056 lines)
