# Context: Part-DB SQLite Category Export Feature

**Session Date:** April 24, 2026  
**Participant:** Daniel (electronics/hardware engineering, KiCAD integration work)  
**Status:** Architecture & implementation scaffolding complete

---

## Problem Statement

Daniel wants to add an export function to Part-DB that generates a flat SQLite database with a **separate table for each part category**. This would enable:

- External data analysis and reporting
- Sharing category-specific datasets
- Integration with external EM simulation pipelines (PCB electromagnetic analysis)
- Lightweight read-only copies of inventory slices
- Complementary functionality to existing Part-DB CSV/JSON exports

---

## Key Design Requirements

### Modularity Constraints
- **Minimal mainline impact**: Should not require modifications to core Part-DB files
- **Clean separation**: Ideally contained in a single isolated service module
- **Easy to review**: PR should be low-risk and straightforward for maintainers
- **Optional feature**: Should be removable without breaking core functionality

### Functional Requirements
- Export all parts grouped by their categories
- Create one SQLite table per category (with sanitized table names)
- Support nested categories (recursive export)
- Include relevant part metadata (name, IPN, description, parameters, instock, etc.)
- Provide both CLI and optional web-based download
- Handle edge cases (empty categories, special characters in category names, Unicode)

### Integration Approach
- Intended for **contribution back to Part-DB project** (open PR)
- Should follow Part-DB's Symfony 7 conventions
- Use existing Doctrine ORM entities (no schema changes)
- Leverage Symfony services auto-wiring where possible

---

## Architecture Solution

### Module Structure
```
src/Services/ImportExportSystem/SQLiteCategoryExporter/
├── SQLiteCategoryExporter.php          # Main orchestrator (Symfony service)
├── CategoryTableBuilder.php             # Creates & populates per-category tables
├── PartFieldMapper.php                  # Entity → SQL column mapping (configurable)
├── SQLiteConnectionFactory.php          # Creates temp/persistent SQLite connections
└── Resources/
    └── schema_template.sql             # (Optional) base schema DDL
```

### Core Components

#### 1. SQLiteCategoryExporter (Orchestrator)
- **Responsibility**: Coordinate the export workflow
- **Public Methods**:
  - `exportToSQLiteFile(string $outputPath)`: Creates .db file with all categories
  - `getExportResponse(): StreamedResponse`: For HTTP download
  - `getExportStats(): array`: Preview stats before export
- **Dependencies**: EntityManager, CategoryTableBuilder, SQLiteConnectionFactory
- **Behavior**: 
  - Retrieves all root categories from database
  - Recursively exports each category (handles nested categories)
  - Creates metadata table with export timestamp
  - Manages temp file cleanup on errors

#### 2. CategoryTableBuilder (Table Creation & Population)
- **Responsibility**: Create and populate individual category tables
- **Public Methods**:
  - `createAndPopulateTable(PDO $pdo, Category $category, array $parts)`
- **Dependencies**: PartFieldMapper
- **Behavior**:
  - Sanitizes table names (handles unicode, special chars, duplicates, length limits)
  - Defines column schema based on available part fields
  - Generates CREATE TABLE SQL
  - Batches INSERT operations for performance
  - Uses prepared statements (SQL injection protection)

#### 3. PartFieldMapper (Data Extraction & Type Mapping)
- **Responsibility**: Define what data to export and map Part entities to SQL types
- **Public Methods**:
  - `getPartColumns(): array`: Returns `['column_name' => 'SQLite_TYPE']`
  - `mapPartToRow($part, array $columns): array`: Extracts values from Part entity
- **Configurable Fields**:
  - `name`, `ipn`, `ean`, `description`, `category`, `comment`
  - `instock` (summed from all PartLots), `unit`, `created_at`, `updated_at`
  - `parameters` (JSON string), `manufacturers` (JSON string)
- **Extensibility**: Easy to add/remove columns without touching other classes
- **Type Handling**: Properly handles null values, Date objects, collections

#### 4. SQLiteConnectionFactory (Connection Management)
- **Responsibility**: Create and configure SQLite database connections
- **Public Methods**:
  - `createConnection(string $filePath): PDO`
- **Configuration**:
  - `PRAGMA journal_mode = WAL` (Write-Ahead Logging for concurrency)
  - `PRAGMA synchronous = NORMAL` (balanced durability/speed)
  - `PRAGMA cache_size = 10000` (larger memory cache for export performance)
  - `PRAGMA foreign_keys = ON` (optional constraint enforcement)
- **Error Handling**: Throws `PDOException` on connection failure

### Design Patterns Used

1. **Separation of Concerns**: Each class has one well-defined responsibility
2. **Dependency Injection**: All dependencies injected via constructor
3. **Symfony Service Auto-wiring**: Automatic registration via type hints
4. **Strategy Pattern**: PartFieldMapper can be swapped for alternative implementations
5. **Prepared Statements**: SQL injection protection built-in
6. **Fail-Safe**: Temp files cleaned up on export errors

---

## Integration Points (3 Options)

### Option A: Console Command (Recommended for Initial PR)
```php
// src/Command/ExportCategoryDataCommand.php
#[AsCommand(name: 'partdb:export:sqlite-categories')]
class ExportCategoryDataCommand extends Command { ... }
```
- **Usage**: `php bin/console partdb:export:sqlite-categories`
- **Mainline impact**: Zero UI changes required
- **Output**: File in `var/exports/parts_by_category.db`

### Option B: Web Endpoint (Optional Follow-up)
```php
// src/Controller/Tools/ExportController.php
#[Route('/tools/export/sqlite-categories')]
public function exportSQLiteCategories(): StreamedResponse { ... }
```
- **UI change**: Add one button to Tools template
- **Usage**: Click button, download .db file directly
- **Same service** used by both CLI and web

### Option C: Service-Only (Minimal)
- Provide the SQLiteCategoryExporter service
- Users call it via API or custom scripts
- No CLI command or UI integration

---

## Implementation Details

### Entity Mapping (Part → Table Columns)
The `PartFieldMapper` extracts these fields (customizable):

| Column | Type | Source | Notes |
|--------|------|--------|-------|
| `id` | INTEGER PRIMARY KEY | Auto | SQLite auto-increment |
| `name` | TEXT NOT NULL | `Part.getName()` | Required |
| `ipn` | TEXT | `Part.getIpn()` | Part number |
| `ean` | TEXT | `Part.getEan()` | Barcode |
| `description` | TEXT | `Part.getDescription()` | Long text |
| `category` | TEXT | `Part.getCategory().getName()` | Direct category only |
| `comment` | TEXT | `Part.getComment()` | User notes |
| `instock` | INTEGER | Sum of `PartLot.getAmount()` | Total across all storage |
| `unit` | TEXT | `Part.getPartUnit().getUnit()` | "piece", "meter", etc. |
| `created_at` | DATETIME | `Part.getCreatedAt()` | ISO format |
| `updated_at` | DATETIME | `Part.getUpdatedAt()` | ISO format |
| `parameters` | TEXT | JSON string of params | `[{"name":"...", "value":"...", "unit":"..."}]` |
| `manufacturers` | TEXT | JSON string of mfg | `[{"name":"..."}]` |

**Flexible approach**: Parameters and nested data stored as JSON columns rather than normalized tables. Balances simplicity with information preservation.

### Table Naming Strategy

Categories → SQLite table names (sanitization process):
1. Remove non-alphanumeric characters (except `_` and `-`)
2. Truncate to 60 characters (SQLite identifier limit ~64)
3. Prefix with `_` if name starts with digit
4. Fallback to `parts` if empty after sanitization
5. Metadata stored in `_export_metadata` table

**Examples**:
- "Resistors" → `Resistors`
- "SMD Capacitors (0603)" → `SMD_Capacitors__0603_`
- "IC & Processors" → `IC___Processors`
- Category with id 5 → `_5_parts`

### Recursive Category Handling

Export process:
```
exportCategory(root)
  ├─ Create table for root's parts
  ├─ exportCategory(subcategory1)
  │   ├─ Create table for subcategory1's parts
  │   └─ exportCategory(subsubcategory)
  │       └─ ...
  └─ exportCategory(subcategory2)
      └─ ...
```

**Behavior**:
- Each category gets its own table
- Parts only appear in their direct category's table (not inherited from parents)
- Nested category structure preserved via table naming convention (e.g., `Resistors_High_Power`)

### Performance Considerations

**Optimizations included**:
- Batch inserts using prepared statements
- WAL mode for concurrent access
- Larger memory cache (10000 pages)
- Reduced sync frequency (NORMAL = fsync every commit)

**For large inventories** (10,000+ parts):
- Streaming inserts already efficient
- Consider async export (background job)
- Can add pagination if needed

---

## File Modifications Summary

### New Files Created
- `src/Services/ImportExportSystem/SQLiteCategoryExporter/SQLiteCategoryExporter.php` (~150 lines)
- `src/Services/ImportExportSystem/SQLiteCategoryExporter/CategoryTableBuilder.php` (~120 lines)
- `src/Services/ImportExportSystem/SQLiteCategoryExporter/PartFieldMapper.php` (~150 lines)
- `src/Services/ImportExportSystem/SQLiteCategoryExporter/SQLiteConnectionFactory.php` (~40 lines)
- `src/Command/ExportCategoryDataCommand.php` (~50 lines) — *Optional*
- `src/Controller/Tools/ExportController.php` (~40 lines) — *Optional*
- `tests/Services/ImportExportSystem/SQLiteCategoryExporterTest.php` (~200 lines) — *Tests*

### Files Modified
- Tools UI template: **+1 line** (export button) — *Optional*
- **All other Part-DB files**: UNCHANGED

### Configuration Changes
- None required (Symfony auto-wiring handles everything)

---

## Testing Strategy

### Unit Tests
```php
// tests/Services/ImportExportSystem/SQLiteCategoryExporterTest.php

testExportsAllCategoriesSeparately()      // Verify each category gets own table
testHandlesEmptyCategories()              // Skip tables with no parts
testHandlesSpecialCharactersInNames()     // Unicode, symbols, duplicates
testPreservesPartData()                   // All fields correctly mapped
testRecursiveCategoryExport()             // Nested categories handled
testGeneratesValidSQLiteDatabase()        // Can open/query result file
testCleansUpTempFilesOnError()            // No orphaned temp files
```

### Integration Tests
- Create test fixtures with sample categories/parts
- Export to temp SQLite file
- Query result file directly
- Verify table structure and data integrity

### Manual Testing Checklist
- [ ] Export with no parts
- [ ] Export with 1000+ parts
- [ ] Categories with unicode names (μ, Ω, etc.)
- [ ] Special chars: `&`, `(`, `)`, `-`, `/`, `.`
- [ ] Very long category names (>100 chars)
- [ ] Deeply nested categories (5+ levels)
- [ ] Parts with null/empty fields
- [ ] Large parameters (complex JSON)

---

## PR Strategy (Atomic Commits)

### Commit 1: Core Exporter Module
```
Subject: Add SQLiteCategoryExporter service module

- SQLiteCategoryExporter: Orchestrates export workflow
- CategoryTableBuilder: Creates & populates per-category tables
- PartFieldMapper: Maps Part entities to SQL columns
- SQLiteConnectionFactory: Manages SQLite connections
- Fully testable, zero dependencies on mainline code
- Includes comprehensive test suite

No changes to existing files. Feature is opt-in.
```

### Commit 2: Console Command
```
Subject: Add console command: partdb:export:sqlite-categories

Users can now export via CLI:
  php bin/console partdb:export:sqlite-categories

Output goes to: var/exports/parts_by_category.db

Includes progress indicators and error handling.
```

### Commit 3: Web UI Integration (Optional)
```
Subject: Add export button to Tools UI

- New ExportController endpoint for HTTP downloads
- Adds download button to Tools panel
- Uses same SQLiteCategoryExporter service as CLI
- Downloads as: parts_YYYY-MM-DD_HHMMSS.db

Can be deployed independently of Commit 1-2.
```

### Commit 4: Documentation
```
Subject: Add documentation for SQLite category export

- Usage guide (CLI and web)
- Feature overview
- Customization guide (extending PartFieldMapper)
- Examples and use cases
```

---

## Extension Points (For Future Enhancement)

### 1. Alternative Grouping Strategies
Instead of just categories, enable export by:
- Manufacturer
- Storage location
- Custom tags
- Parametric characteristics

**Implementation**: Create `ManufacturerTableBuilder`, `StorageTableBuilder`, etc.  
**No mainline changes needed** — use same service architecture.

### 2. Format Options
Extend beyond SQLite:
- CSV per category (single file with category column)
- JSON export (nested structure)
- Excel workbook (sheet per category)

**Implementation**: New exporter classes with same interface.

### 3. Filtering & Selection
- Export only specific categories
- Exclude empty categories
- Filter by part properties (e.g., "only in stock")
- Date range filtering (modified since X)

**Implementation**: Add parameters to `exportToSQLiteFile()`.

### 4. Metadata & Relationships
- Include storage location info in separate table
- Add supplier/pricing in related table
- Include parameter definitions table
- Export relationships between parts

**Implementation**: Extend `CategoryTableBuilder` to create related tables.

### 5. Database Engine Support
- PostgreSQL export (different syntax, same logic)
- MySQL/MariaDB export

**Implementation**: Create `PostgreSQLExporter`, `MySQLExporter` abstractions.

---

## Known Limitations & Workarounds

### 1. Parameters as JSON
**Issue**: Parameters stored as JSON instead of normalized table.  
**Reason**: Simplicity; each part has different parameter sets.  
**Workaround**: Users can parse JSON in external tools or create views.

### 2. No Image/Attachment Export
**Issue**: Attachments not included (binary files, storage concerns).  
**Reason**: Out of scope; can be handled separately.  
**Workaround**: Export file URLs or use Part-DB API for full backup.

### 3. Category Path Not Stored
**Issue**: Only direct category stored, not full hierarchy path.  
**Reason**: Simplicity; nested table names preserve structure.  
**Workaround**: Can extend `getPartColumns()` to add `category_path`.

### 4. No Incremental Export
**Issue**: Always exports entire database.  
**Reason**: Simpler implementation; SQLite is lightweight.  
**Workaround**: Can script externally to diff exports or track timestamps.

---

## Maintenance & Long-Term Considerations

### Version Compatibility
- Built for **Symfony 7**, **Doctrine ORM 2.x+**
- Should work with MySQL, PostgreSQL, SQLite backends (Part-DB supported)
- Compatible with PHP 8.2+

### When Part-DB Adds Fields
- Update `PartFieldMapper.getPartColumns()` — one file
- No changes to builder, orchestrator, or factory
- Backward compatible (new columns optional)

### Code Review Checklist
- [ ] No SQL injection vectors (prepared statements used)
- [ ] No hardcoded paths (configurable, respects Part-DB directories)
- [ ] Proper error handling (exceptions, temp file cleanup)
- [ ] Follows PSR-12 code style (Part-DB conventions)
- [ ] No breaking changes to existing code
- [ ] Tests cover happy path + edge cases
- [ ] Performance acceptable for large datasets

---

## Related Part-DB Issues

- **#618** "Export and print functions for all tables" — Similar feature request, but broader scope
- **#614** "Better import feature" — Complementary: this enables easier data extraction

This feature complements existing CSV/JSON import/export; provides alternative format for users needing relational structure.

---

## Deliverables from This Session

### 1. **PartDB_SQLiteExport_Architecture.md**
Complete design document with:
- Modularity strategy
- Full PHP implementation approach
- Integration options (3 levels)
- Testing strategy
- PR strategy with atomic commits
- Advantages, concerns, and answers
- Example PR description

### 2. **PartDB_SQLiteExporter_Scaffold.php**
Production-ready scaffold code (~350 lines):
- `SQLiteCategoryExporter` — Full implementation
- `CategoryTableBuilder` — Field mapping, table generation
- `PartFieldMapper` — Configurable column definitions
- `SQLiteConnectionFactory` — SQLite connection setup
- Ready to adapt and extend

### 3. **This Context File**
Reference documentation for:
- Problem statement and requirements
- Architecture overview
- Implementation details
- Testing strategy
- PR approach
- Extension points for future work
- Known limitations

---

## Next Steps for Implementation

### Phase 1: Core Development
1. [ ] Copy scaffold code to Part-DB src/Services directory
2. [ ] Adjust PartFieldMapper to match actual Part entity structure
3. [ ] Test locally with sample Part-DB installation
4. [ ] Write unit tests (scaffold provided)
5. [ ] Manual testing with edge cases

### Phase 2: CLI Integration
6. [ ] Create ExportCategoryDataCommand
7. [ ] Add to config/services.yaml (likely auto-wired)
8. [ ] Test `php bin/console partdb:export:sqlite-categories`
9. [ ] Verify output file format and data integrity

### Phase 3: Web UI (Optional)
10. [ ] Create ExportController
11. [ ] Add button to Tools template
12. [ ] Test download flow
13. [ ] Add streaming response for large files

### Phase 4: Polish & Docs
14. [ ] Add inline code documentation
15. [ ] Write user-facing documentation
16. [ ] Prepare PR with atomic commits
17. [ ] Address review feedback

### Phase 5: Upstream Contribution
18. [ ] Fork Part-DB on GitHub
19. [ ] Create feature branch
20. [ ] Push commits
21. [ ] Open PR with description
22. [ ] Engage in review process

---

## References & Resources

### Part-DB Documentation
- Import/Export docs: https://docs.part-db.de/usage/import_export.html
- GitHub repo: https://github.com/Part-DB/Part-DB-server
- Symfony integration: Part-DB uses Symfony 7 framework

### Symfony Resources
- Service auto-wiring: https://symfony.com/doc/current/service_container/autowiring.html
- Console commands: https://symfony.com/doc/current/console.html
- Database access: https://symfony.com/doc/current/doctrine.html

### SQLite Resources
- PRAGMA settings: https://www.sqlite.org/pragmaclause.html
- CREATE TABLE syntax: https://www.sqlite.org/lang_createtable.html
- Performance tips: https://www.sqlite.org/bestpractice.html

---

## Contact & Updates

**Created**: April 24, 2026  
**Context Owner**: Daniel  
**Status**: Architecture finalized, ready for implementation  
**Last Updated**: April 24, 2026

This context is a living document. Update with actual implementation details, test results, and review feedback as work progresses.