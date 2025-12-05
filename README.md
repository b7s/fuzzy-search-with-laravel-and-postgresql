# Fuzzy Search with PostgreSQL pg_trgm in Laravel

<div align="center">
  <img src="fuzzy-search-with-laravel-and-postgresql.webp?_=1" width="99%">
</div>

## What is Fuzzy Search with pg_trgm?

Fuzzy search is all about finding what you meant (not just what you typed). Instead of demanding exact matches, it uncovers results that are close enough to feel natural. With PostgreSQL’s pg_trgm extension, you gain a powerful way to match strings using trigram-based similarity algorithms. This means your searches can handle typos, accents, and partial words, delivering smarter and more human‑like results.

### How it Works

**Trigrams** are groups of three consecutive characters from a string. For example:
- "João" → {" j", "jo", "oã", "ão", "o "}
- "joao" → {" j", "jo", "oa", "ao", "o "}

The extension compares trigrams between strings to calculate similarity scores (0 to 1, where 1 is identical).

### Key Functions

1. **`similarity(text1, text2)`**: Compares entire strings
   - Returns: 0.0 to 1.0
   - Example: `similarity('joao', 'João da Silva')` = 0.12

2. **`word_similarity(text1, text2)`**: Finds best matching word in text2
   - Returns: 0.0 to 1.0
   - Example: `word_similarity('joao', 'João da Silva')` = 0.4
   - **Better for partial matches**

### Benefits

- **Typo tolerance**: "dentista" matches "dntista"
- **Accent insensitive**: "joao" matches "João"
- **Partial matching**: "dent" matches "Dentista"
- **Performance**: GIN indexes make searches fast even on large datasets
- **No external dependencies**: Built into PostgreSQL

## Implementation in Laravel

### 1. Enable the Extension

Create a migration to enable `pg_trgm`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        if (DB::connection()->getDriverName() !== 'pgsql') {
            return;
        }

        DB::statement('CREATE EXTENSION IF NOT EXISTS pg_trgm');
    }

    public function down(): void
    {
        if (DB::connection()->getDriverName() !== 'pgsql') {
            return;
        }

        DB::statement('DROP EXTENSION IF EXISTS pg_trgm');
    }
};
```

### 2. Create GIN Indexes

Add GIN indexes to columns you want to search:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        if (DB::connection()->getDriverName() !== 'pgsql') {
            return;
        }

        // Index for contact names
        DB::statement('CREATE INDEX contacts_name_trgm_idx ON contacts USING gin (name gin_trgm_ops)');
        
        // Index for appointment titles
        DB::statement('CREATE INDEX appointments_title_trgm_idx ON appointments USING gin (title gin_trgm_ops)');
    }

    public function down(): void
    {
        if (DB::connection()->getDriverName() !== 'pgsql') {
            return;
        }

        DB::statement('DROP INDEX IF EXISTS contacts_name_trgm_idx');
        DB::statement('DROP INDEX IF EXISTS appointments_title_trgm_idx');
    }
};
```

### 3. Create a Fuzzy Search Service

```php
<?php

declare(strict_types=1);

namespace App\Services\Search;

use App\Models\Contact;
use Illuminate\Support\Collection;

class FuzzySearchService
{
    private const MIN_SIMILARITY = 0.2;
    private const MIN_WORD_SIMILARITY = 0.3;

    /**
     * @return Collection<int, Contact>
     */
    public function searchContacts(string $searchTerm): Collection
    {
        $searchLower = mb_strtolower($searchTerm);

        // Try exact/partial match first (faster)
        $exactMatch = Contact::query()
            ->whereRaw('LOWER(name) LIKE ?', ["%{$searchLower}%"])
            ->get();

        if ($exactMatch->isNotEmpty()) {
            return $exactMatch;
        }

        // Fall back to fuzzy search
        return Contact::query()
            ->selectRaw(
                '*, GREATEST(
                    word_similarity(?, LOWER(name)),
                    similarity(LOWER(name), ?)
                ) as relevance',
                [$searchLower, $searchLower]
            )
            ->where(function ($q) use ($searchLower): void {
                $q->whereRaw('word_similarity(?, LOWER(name)) > ?', 
                    [$searchLower, self::MIN_WORD_SIMILARITY])
                  ->orWhereRaw('similarity(LOWER(name), ?) > ?', 
                    [$searchLower, self::MIN_SIMILARITY]);
            })
            ->orderByDesc('relevance')
            ->limit(10)
            ->get();
    }
}
```

### 4. Usage Examples

```php
// Basic search
$service = new FuzzySearchService();
$contacts = $service->searchContacts('joao');
// Finds: "João", "João Silva", "Joana", etc.

// Raw query example
$results = DB::table('contacts')
    ->selectRaw('name, word_similarity(?, LOWER(name)) as score', ['dent'])
    ->whereRaw('word_similarity(?, LOWER(name)) > 0.3', ['dent'])
    ->orderByDesc('score')
    ->get();
// Finds: "Dentista", "Dental Clinic", etc.

// Check similarity score
$score = DB::selectOne(
    "SELECT word_similarity('joao', 'João da Silva') as score"
);
// Returns: 0.4
```

## Best Practices

### 1. Choose the Right Function

- Use `word_similarity()` for partial word matching (e.g., searching names)
- Use `similarity()` for full text comparison
- Combine both with `GREATEST()` for best results

### 2. Set Appropriate Thresholds

```php
// Strict matching (fewer false positives)
private const MIN_WORD_SIMILARITY = 0.5;

// Flexible matching (more results, some false positives)
private const MIN_WORD_SIMILARITY = 0.3;

// Very flexible (many results, more false positives)
private const MIN_WORD_SIMILARITY = 0.2;
```

### 3. Normalize Input

```php
private function normalizeSearchTerm(string $term): string
{
    // Remove extra spaces
    $term = trim(preg_replace('/\s+/', ' ', $term) ?? '');
    
    // Convert to lowercase for case-insensitive search
    return mb_strtolower($term);
}
```

### 4. Use Exact Match First

Always try exact/partial matching before fuzzy search for better performance:

```php
// Fast: Uses regular index
$exact = Model::query()
    ->whereRaw('LOWER(name) LIKE ?', ["%{$search}%"])
    ->get();

if ($exact->isNotEmpty()) {
    return $exact;
}

// Slower: Uses trigram matching
$fuzzy = Model::query()
    ->whereRaw('word_similarity(?, LOWER(name)) > 0.3', [$search])
    ->get();
```

### 5. Limit Results

Fuzzy searches can return many results. Always limit:

```php
->limit(10)  // or 20, 50, etc.
```

### 6. Cache Results

Fuzzy searches are computationally expensive. Cache when possible:

```php
$cacheKey = "fuzzy_search:{$searchTerm}";
return Cache::remember($cacheKey, 300, function () use ($searchTerm) {
    return $this->performFuzzySearch($searchTerm);
});
```

## Performance Considerations

### Index Types

- **GIN (Generalized Inverted Index)**: Best for fuzzy search
  ```sql
  CREATE INDEX name_trgm_idx ON table USING gin (column gin_trgm_ops);
  ```

- **GiST (Generalized Search Tree)**: Faster updates, slower queries
  ```sql
  CREATE INDEX name_trgm_idx ON table USING gist (column gist_trgm_ops);
  ```

### Query Optimization

```php
// Good: Combines exact and fuzzy
->where(function ($q) use ($search) {
    $q->whereRaw('LOWER(name) LIKE ?', ["%{$search}%"])
      ->orWhereRaw('word_similarity(?, LOWER(name)) > 0.3', [$search]);
})

// Bad: Only fuzzy (slower)
->whereRaw('word_similarity(?, LOWER(name)) > 0.3', [$search])
```

## Testing

```php
it('finds contacts with fuzzy search', function () {
    Contact::factory()->create(['name' => 'João da Silva']);
    Contact::factory()->create(['name' => 'Maria Santos']);
    
    $service = new FuzzySearchService();
    
    // Exact match
    $results = $service->searchContacts('joão');
    expect($results)->toHaveCount(1);
    
    // Partial match
    $results = $service->searchContacts('joao');  // without accent
    expect($results)->toHaveCount(1);
    
    // Fuzzy match
    $results = $service->searchContacts('jo');
    expect($results)->toHaveCount(1);
    
    // No match
    $results = $service->searchContacts('xyz');
    expect($results)->toBeEmpty();
});
```

## Common Use Cases

1. **Contact/User Search**: Find people by partial names
2. **Product Search**: Match product names with typos
3. **Address Search**: Find locations with approximate addresses
4. **Tag/Category Search**: Match similar tags
5. **Content Search**: Find articles/posts with similar titles

## Troubleshooting

### Extension Not Found
```bash
# Check if extension is available
psql -d your_database -c "SELECT * FROM pg_available_extensions WHERE name = 'pg_trgm';"

# Install if missing (Ubuntu/Debian)
sudo apt-get install postgresql-contrib
```

### Slow Queries
- Ensure GIN indexes are created
- Check index usage: `EXPLAIN ANALYZE SELECT ...`
- Lower similarity thresholds if too many results
- Add `LIMIT` clause

### No Results
- Lower similarity thresholds (try 0.2 or 0.1)
- Check if search term is normalized (lowercase, trimmed)
- Verify data exists in the table
- Test similarity scores manually:
  ```php
  DB::select("SELECT similarity('search', 'target') as score");
  ```

## References

- [PostgreSQL pg_trgm Documentation](https://www.postgresql.org/docs/current/pgtrgm.html)
- [Trigram Matching](https://www.postgresql.org/docs/current/pgtrgm.html#PGTRGM-FUNCS-OPS)
- [GIN Indexes](https://www.postgresql.org/docs/current/gin.html)
