---
name: dummy-data
description: "Generates dummy data for testing and development in Laravel applications. Use when creating factories, seeders, or populating the database with test data that follows the schema structure. Activates when the user mentions dummy data, test data, seed data, factory, seeder, fake data, populate database, sample data, make factory, make seeder, or needs realistic data for development or testing."
argument-hint: "model name, table name, or 'all' for full database seed"
---

# Dummy Data Generation

## When to Use

Activate this skill when:

- Creating or updating Eloquent factories for a model
- Creating or updating database seeders
- Populating the database with realistic test/development data
- Generating data that respects foreign key relationships
- Setting up data for a feature under development

## Procedure

### Step 0 — Interview the User

**Before writing any code**, ask the user these questions. Do not proceed until you have answers.

1. **Domain context** — What is this data for? (e.g., "food products", "e-commerce orders", "medical appointments"). This determines what realistic values look like.
2. **Media** — Does any column store an image or video? Should seed data include real media files?
   - If yes to images: ask what subject/style (e.g., "food photography", "product photos on white background").
   - If yes to videos: ask what kind (e.g., "recipe videos", "product demos").
3. **Volume** — How many records should the seeder create by default?
4. **Relationships** — Should related models (e.g., categories, tags, users) also be seeded, or do they already exist?

Use the answers to drive all naming, values, and data choices in the factory. Data must feel realistic for the domain — a food product should have names like "Grilled Salmon Fillet" and prices like `12.99`, not `fake()->sentence()` producing "Quia et voluptas".

#### Domain Value Examples

| Domain | `name` / `title` examples | `description` examples | `price` range |
|---|---|---|---|
| Food products | "Grilled Chicken Wrap", "Mango Lassi" | Appetizing description of ingredients/taste | 3.00 – 50.00 |
| E-commerce products | "Wireless Earbuds Pro", "Leather Wallet" | Feature-focused description | 9.99 – 499.99 |
| Blog posts | "10 Tips for Better Sleep", "My Year in Review" | Article summary | N/A |
| Medical appointments | "Annual Physical", "Dental Checkup" | Appointment notes | N/A |

Always use `fake()->randomElement([...])` with a hand-picked array of domain-appropriate values for key text fields (name, title, description), rather than generic Faker methods.

**Example — food product factory:**

```php
'name'        => fake()->randomElement([
    'Grilled Chicken Wrap', 'Margherita Pizza', 'Mango Lassi',
    'Caesar Salad', 'BBQ Beef Burger', 'Veggie Burrito',
    'Pad Thai', 'Chocolate Lava Cake', 'Green Smoothie Bowl',
]),
'description' => fake()->randomElement([
    'A delicious blend of fresh ingredients with bold flavors.',
    'Made with locally sourced produce and chef-crafted seasoning.',
    'A customer favourite, prepared fresh daily.',
]),
'price'       => fake()->randomFloat(2, 3, 45),
'category'    => fake()->randomElement(['Mains', 'Drinks', 'Desserts', 'Starters']),
```

### Step 1 — Understand the Schema

Before generating anything, inspect the relevant schema:

1. Read the model file (e.g., `app/Models/Product.php`) — note fillable fields, casts, and relationships.
2. Read the corresponding migration (e.g., `database/migrations/*_create_products_table.php`) — note column types, nullability, unique constraints, and foreign keys.
3. Check for existing factory (`database/factories/ProductFactory.php`) and seeder (`database/seeders/ProductSeeder.php`) to avoid overwriting intentional customizations.

```bash
# Discover migrations and factories quickly
ls database/migrations/
ls database/factories/
ls database/seeders/
```

### Step 2 — Create or Update the Factory

Use artisan to scaffold if the factory doesn't exist:

```bash
php artisan make:factory ProductFactory --model=Product --no-interaction
```

Map each column type to the appropriate `fake()` helper:

| Column type | Faker method |
|---|---|
| `string` name | `fake()->name()` |
| `string` email (unique) | `fake()->unique()->safeEmail()` |
| `string` title/label | `fake()->sentence(3)` |
| `text` / `longText` | `fake()->paragraph()` |
| `integer` / `bigInteger` | `fake()->numberBetween(1, 100)` |
| `decimal` / `float` price | `fake()->randomFloat(2, 1, 999)` |
| `boolean` | `fake()->boolean()` |
| `date` | `fake()->date()` |
| `timestamp` / `dateTime` | `fake()->dateTimeBetween('-1 year', 'now')` |
| `foreignId` (`user_id`) | Use `User::factory()` or `User::inRandomOrder()->first()->id` |
| `enum` | `fake()->randomElement(MyEnum::values())` |
| `json` | Construct a realistic array |
| `uuid` | `fake()->uuid()` |
| `slug` | `fake()->slug()` |
| `string` image / photo / avatar / thumbnail | See Media Files below |
| `string` video / clip / recording | See Media Files below |

### Media Files

When a column name suggests an image or video (e.g., `image`, `photo`, `avatar`, `thumbnail`, `cover`, `banner`, `video`, `clip`), **never use fake-generated or random files**. Always use real, valid media files.

**Setup** — place real seed media files in `database/seeders/media/`:

```
database/seeders/media/
├── images/
│   ├── sample-1.jpg
│   ├── sample-2.jpg
│   └── sample-3.jpg
└── videos/
    ├── sample-1.mp4
    └── sample-2.mp4
```

Instruct the user to add real image/video files to these directories before seeding. These files are committed to the repository so all team members share the same seed media.

**All media is stored via [Spatie Media Library](https://spatie.be/docs/laravel-medialibrary).** Never write directly to `Storage::disk()` or store file paths in model columns. Always use `addMedia()` in an `afterCreating` callback on the factory.

**Before writing the factory**, confirm:
1. The model uses the `HasMedia` interface and `InteractsWithMedia` trait.
2. The relevant media collection name (check `registerMediaCollections()` on the model, or ask the user).

**Images** — attach via `afterCreating`:

```php
use Illuminate\Database\Eloquent\Factories\Factory;

$this->afterCreating(function (Product $product) {
    $samples = glob(database_path('seeders/media/images/*.{jpg,jpeg,png,webp}'), GLOB_BRACE);

    if (empty($samples)) {
        throw new \RuntimeException(
            'No seed images found. Add real images to database/seeders/media/images/ before seeding.'
        );
    }

    $product->addMedia($samples[array_rand($samples)])
        ->preservingOriginal()
        ->toMediaCollection('images');
});
```

**Videos** — attach via `afterCreating`:

```php
$this->afterCreating(function (Product $product) {
    $samples = glob(database_path('seeders/media/videos/*.{mp4,mov,webm}'), GLOB_BRACE);

    if (empty($samples)) {
        throw new \RuntimeException(
            'No seed videos found. Add real videos to database/seeders/media/videos/ before seeding.'
        );
    }

    $product->addMedia($samples[array_rand($samples)])
        ->preservingOriginal()
        ->toMediaCollection('videos');
});
```

**Multiple media per record** — loop to attach several files:

```php
$this->afterCreating(function (Product $product) {
    $samples = glob(database_path('seeders/media/images/*.{jpg,jpeg,png,webp}'), GLOB_BRACE);

    collect($samples)->random(min(3, count($samples)))->each(
        fn ($path) => $product->addMedia($path)
            ->preservingOriginal()
            ->toMediaCollection('images')
    );
});
```

**Rules for media fields:**
- **Always use Spatie Media Library** — never store file paths directly in model columns or write to `Storage::disk()` manually.
- Always use `->preservingOriginal()` so the source seed files remain intact for repeated `migrate:fresh --seed` runs.
- If `database/seeders/media/` is empty, throw a `\RuntimeException` with a clear message — do not silently skip media.
- Never use `UploadedFile::fake()`, `fake()->imageUrl()`, or random strings — they produce broken or non-existent files.
- Never use external placeholder services (e.g., picsum, lorempixel) — they create a network dependency and fail offline.
- Check `registerMediaCollections()` on the model to use the correct collection name.

**Relationship pattern** — use nested factories for `belongsTo`:

```php
public function definition(): array
{
    return [
        'user_id'    => User::factory(),
        'title'      => fake()->sentence(3),
        'price'      => fake()->randomFloat(2, 5, 500),
        'is_active'  => true,
        'created_at' => now(),
    ];
}
```

**States** — add named states for common test scenarios:

```php
public function inactive(): static
{
    return $this->state(fn (array $attributes) => [
        'is_active' => false,
    ]);
}
```

### Step 3 — Create or Update the Seeder

Use artisan to scaffold if the seeder doesn't exist:

```bash
php artisan make:seeder ProductSeeder --no-interaction
```

Seed in dependency order (parents before children). Use `create()` for persisting and `make()` for in-memory only:

```php
public function run(): void
{
    Product::factory(50)->create();

    // With specific state
    Product::factory(10)->inactive()->create();
}
```

Register the seeder in `DatabaseSeeder.php` if it isn't already:

```php
public function run(): void
{
    $this->call([
        UserSeeder::class,
        ProductSeeder::class,
    ]);
}
```

### Step 4 — Run and Verify

```bash
# Seed the database (development only)
php artisan db:seed --class=ProductSeeder

# Or reset and reseed everything
php artisan migrate:fresh --seed
```

Verify row count quickly:

```bash
php artisan tinker --execute "echo App\Models\Product::count();"
```

### Step 5 — Format Code

Run Pint to ensure code style compliance:

```bash
vendor/bin/pint --dirty --format agent
```

## Key Rules

- **Always inspect the migration first** — do not guess column names or types.
- **Respect unique constraints** — use `fake()->unique()` for unique columns.
- **Respect nullable columns** — use `nullable()` or `null` as the default, not a forced value.
- **Foreign keys must resolve** — either use a nested factory (`User::factory()`) or ensure parent records exist before seeding children.
- **Never hard-code IDs** — always resolve foreign keys dynamically.
- **Keep factories realistic** — data should reflect real-world values so tests are meaningful.
- **Do not modify production seeders** — only touch `DatabaseSeeder` or create dedicated `*Seeder` files.
