# Laravel Query Optimization

## Inhaltsverzeichnis

1. [Eager Loading (N+1 Problem vermeiden)](#1-eager-loading-n1-problem-vermeiden)
2. [Query Optimierung](#2-query-optimierung)
3. [Subqueries](#3-subqueries)
4. [Joins](#4-joins)
5. [Raw Queries & Expressions](#5-raw-queries--expressions)
6. [Caching](#6-caching)
7. [Indexierung & Performance](#7-indexierung--performance)
8. [Spezielle Optimierungen](#8-spezielle-optimierungen)
9. [Relationship Optimierungen](#9-relationship-optimierungen)
10. [Debugging & Monitoring](#10-debugging--monitoring)

---

## 1. Eager Loading (N+1 Problem vermeiden)

### Was ist das N+1 Problem?

Das N+1 Problem ist eines der häufigsten Performance-Probleme in Laravel-Anwendungen. Es tritt auf, wenn Sie eine Liste von Modellen laden (1 Query) und dann für jedes Modell eine Beziehung abfragen (N Queries), was zu N+1 Datenbankabfragen führt.

**Beispiel des Problems:**
```php
// Schlecht: Erzeugt N+1 Queries
$users = User::all(); // Query 1: SELECT * FROM users
foreach ($users as $user) {
    // Query 2 bis N+1: SELECT * FROM posts WHERE user_id = ?
    echo $user->posts->count();
}
```

Bei 100 Usern würden hier 101 Queries ausgeführt!

### with() - Die Lösung für Eager Loading

Mit `with()` laden Sie Beziehungen im Voraus mit nur einer zusätzlichen Query:

```php
// Gut: Nur 2 Queries insgesamt
$users = User::with('posts')->get();
// Query 1: SELECT * FROM users
// Query 2: SELECT * FROM posts WHERE user_id IN (1, 2, 3, ...)

foreach ($users as $user) {
    echo $user->posts->count(); // Keine zusätzliche Query!
}
```

**Mehrere Beziehungen laden:**
```php
// Lädt posts und comments mit 3 Queries total
$users = User::with(['posts', 'comments'])->get();
```

**Verschachtelte Beziehungen:**
```php
// Lädt alle Posts und deren Comments
$users = User::with('posts.comments')->get();
// Query 1: SELECT * FROM users
// Query 2: SELECT * FROM posts WHERE user_id IN (...)
// Query 3: SELECT * FROM comments WHERE post_id IN (...)
```

**Bedingte Eager Loading:**
```php
// Lädt nur published Posts
$users = User::with(['posts' => function ($query) {
    $query->where('published', true)
          ->orderBy('created_at', 'desc');
}])->get();
```

### withCount() - Effizientes Zählen

Wenn Sie nur die Anzahl der Beziehungen benötigen, ist `withCount()` effizienter als das Laden aller Daten:

```php
// Zählt Posts ohne sie zu laden
$users = User::withCount('posts')->get();
foreach ($users as $user) {
    echo $user->posts_count; // Automatisch verfügbar
}

// Mit Bedingungen
$users = User::withCount(['posts' => function ($query) {
    $query->where('published', true);
}])->get();
// $user->posts_count enthält nur published posts
```

### load() - Lazy Eager Loading

Manchmal möchten Sie Beziehungen erst nach dem initialen Laden hinzufügen:

```php
$users = User::all();

// Entscheidung basierend auf Logik
if ($needPosts) {
    $users->load('posts');
}

// Oder bedingt für einzelne Modelle
$user = User::find(1);
if ($user->is_premium) {
    $user->load('premiumContent');
}
```

---

## 2. Query Optimierung

### select() - Nur benötigte Felder laden

Standardmäßig lädt Laravel alle Spalten (`SELECT *`). Mit `select()` kann man die geladenen Daten minimieren:

```php
// Statt SELECT * FROM users
User::all();

// Besser: SELECT id, name, email FROM users
User::select('id', 'name', 'email')->get();

// Bei Beziehungen wichtig: Foreign Keys einschließen!
User::select('id', 'name')->with(['posts' => function ($query) {
    // user_id wird für die Beziehung benötigt!
    $query->select('id', 'user_id', 'title');
}])->get();
```

### chunk() - Große Datenmengen verarbeiten

Bei der Verarbeitung großer Datenmengen kann der Speicher schnell voll werden. `chunk()` verarbeitet Daten in kleineren Blöcken:

```php
// Verarbeitet 100 User auf einmal
User::chunk(100, function ($users) {
    foreach ($users as $user) {
        // Verarbeitung
        $user->sendNewsletter();
    }
});

// Mit where Bedingungen
User::where('subscribed', true)
    ->chunk(100, function ($users) {
        // Nur subscribed users
    });

// chunkById ist sicherer bei Updates
User::where('active', true)
    ->chunkById(100, function ($users) {
        foreach ($users as $user) {
            // Sicher auch wenn user.active geändert wird
            $user->update(['processed' => true]);
        }
    });
```

### cursor() - Lazy Collection für extrem große Datenmengen

`cursor()` verwendet PHP Generatoren und hält nur einen Datensatz im Speicher:

```php
// Sehr speichereffizient
foreach (User::cursor() as $user) {
    // Nur ein User im Speicher
    processUser($user);
}

// Mit where Bedingungen
foreach (User::where('active', true)->cursor() as $user) {
    // Verarbeitung
}
```

**Vorteil:** Minimaler Speicher-Verbrauch  
**Nachteil:** Hält Datenbankverbindung länger offen

---

## 3. Subqueries

### addSelect() - Subquery als zusätzliche Spalte

Mit `addSelect()` kann man berechnete Werte als Spalten hinzufügen:

```php
$users = User::addSelect([
    'last_post_title' => Post::select('title')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1)
])->get();

// Jeder User hat jetzt $user->last_post_title
foreach ($users as $user) {
    echo $user->name . ': ' . $user->last_post_title;
}
```

**Wie es funktioniert:**
- Subquery wird für jede Zeile ausgeführt
- `whereColumn()` verknüpft die Tabellen
- `limit(1)` stellt sicher, dass nur ein Wert zurückgegeben wird

### selectSub() - Benannte Subqueries

Für komplexere Subqueries ist `selectSub()` übersichtlicher:

```php
// Definiere die Subquery
$latestPostDate = Post::select('created_at')
    ->whereColumn('user_id', 'users.id')
    ->latest()
    ->limit(1);

// Verwende sie mit einem Alias
$users = User::select('*')
    ->selectSub($latestPostDate, 'latest_post_date')
    ->get();

// Noch komplexeres Beispiel
$totalRevenue = Order::select(DB::raw('SUM(total)'))
    ->whereColumn('user_id', 'users.id')
    ->where('status', 'completed');

$users = User::select('*')
    ->selectSub($totalRevenue, 'total_revenue')
    ->get();
```

### whereHas() Optimierung mit Subquery

`whereHas()` kann Performance-Probleme verursachen. Eine Subquery ist oft effizienter:

```php
// Langsam bei vielen Daten
$users = User::whereHas('posts', function ($query) {
    $query->where('published', true);
})->get();

// Schneller mit Subquery
$users = User::whereIn('id', function ($query) {
    $query->select('user_id')
        ->from('posts')
        ->where('published', true)
        ->groupBy('user_id');
})->get();
```

---

## 4. Joins

### join() - Basic Join

Joins sind oft schneller als Eager Loading für einfache Abfragen:

```php
// Inner Join
$users = User::join('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*', 'posts.title as post_title')
    ->get();

// Vorsicht: Bei mehreren Posts pro User gibt es doppelte User!
// Lösung: distinct() oder groupBy()
$users = User::join('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*')
    ->distinct()
    ->get();
```

### leftJoin() - Left Join für optionale Beziehungen

Left Join behält alle Datensätze der linken Tabelle, auch ohne Match:

```php
// Alle User, auch ohne Posts
$users = User::leftJoin('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*', DB::raw('COUNT(posts.id) as post_count'))
    ->groupBy('users.id')
    ->get();

// Komplexeres Beispiel mit mehreren Joins
$data = User::leftJoin('profiles', 'users.id', '=', 'profiles.user_id')
    ->leftJoin('subscriptions', 'users.id', '=', 'subscriptions.user_id')
    ->select(
        'users.*',
        'profiles.bio',
        'subscriptions.plan'
    )
    ->get();
```

---

## 5. Raw Queries & Expressions

### DB::raw() - SQL Ausdrücke

Für komplexe SQL-Funktionen oder Berechnungen:

```php
// Concatenation
$users = User::select(
    DB::raw('CONCAT(first_name, " ", last_name) as full_name'),
    'email'
)->get();

// Datums-Berechnungen
$orders = Order::select(
    '*',
    DB::raw('DATEDIFF(NOW(), created_at) as days_old')
)->get();

// Conditional Expressions
$users = User::select(
    '*',
    DB::raw('CASE 
        WHEN age >= 18 THEN "Adult" 
        ELSE "Minor" 
    END as age_group')
)->get();
```

### selectRaw() / whereRaw()

Direktere Syntax für Raw-Queries:

```php
// Aggregation
$stats = User::selectRaw('
    COUNT(*) as total_users,
    AVG(age) as average_age,
    MAX(created_at) as newest_user
')->first();

// Where conditions
$users = User::whereRaw('YEAR(created_at) = ?', [2024])
    ->whereRaw('age > ? AND status = ?', [18, 'active'])
    ->get();

// Having Raw
$usersByCountry = User::selectRaw('country, COUNT(*) as user_count')
    ->groupBy('country')
    ->havingRaw('COUNT(*) > ?', [100])
    ->get();
```

**Wichtig:** Raw Queries umgehen Laravel's SQL Injection Schutz teilweise. Verwenden Sie immer Parameter-Binding (?) statt String-Konkatenation!

---

## 6. Caching

### Query Cache mit remember()

Laravel kann Query-Ergebnisse automatisch cachen:

```php
// Cache für 60 Minuten
$users = User::remember(60)->get();

// Cache mit custom key
$users = User::where('active', true)
    ->remember(60, 'active-users')
    ->get();

// Cache mit Tags (erfordert Redis/Memcached)
$users = User::cacheTags(['users', 'active'])
    ->remember(60)
    ->get();

// Cache leeren
Cache::tags(['users'])->flush();
```

### Cache Facade für mehr Kontrolle

Für komplexere Caching-Logik:

```php
// Standard caching
$users = Cache::remember('all-users', 3600, function () {
    return User::with('posts')->get();
});

// Cache forever
$settings = Cache::rememberForever('app-settings', function () {
    return Setting::all();
});

// Conditional caching
$users = Cache::when($shouldCache, function ($cache) {
    return $cache->remember('users', 3600, function () {
        return User::all();
    });
}, function () {
    return User::all();
});

// Cache mit Tags und custom Store
$users = Cache::store('redis')
    ->tags(['users', 'premium'])
    ->remember('premium-users', 3600, function () {
        return User::where('is_premium', true)->get();
    });
```

**Cache Invalidierung:**
```php
// Specific key
Cache::forget('all-users');

// Mit Tags
Cache::tags(['users'])->flush();

// Alles löschen
Cache::flush();
```

---

## 7. Indexierung & Performance

### Composite Indexes

Composite (mehrspaltige) Indexes sind wichtig für Queries mit mehreren WHERE-Bedingungen:

```php
// Migration
Schema::table('posts', function (Blueprint $table) {
    // Index für WHERE user_id = ? AND created_at > ?
    $table->index(['user_id', 'created_at']);
    
    // Unique composite index
    $table->unique(['user_id', 'slug']);
});

// Query die diesen Index nutzt
Post::where('user_id', 1)
    ->where('created_at', '>', now()->subDays(30))
    ->get();
```

**Wichtig:** Die Reihenfolge der Spalten im Index ist entscheidend! Der Index `(user_id, created_at)` hilft bei:
- WHERE user_id = ?
- WHERE user_id = ? AND created_at > ?

Aber NICHT bei:
- WHERE created_at > ? (ohne user_id)

### Covering Index

Ein Covering Index enthält alle Spalten, die in der Query benötigt werden:

```php
// Wenn Sie oft diese Query ausführen:
User::select('email', 'name', 'created_at')
    ->where('status', 'active')
    ->orderBy('created_at')
    ->get();

// Erstellen Sie einen covering index:
Schema::table('users', function (Blueprint $table) {
    $table->index(['status', 'created_at', 'email', 'name']);
});
```

Die Datenbank kann die Query komplett aus dem Index beantworten, ohne die Tabelle zu lesen!

---

## 8. Spezielle Optimierungen

### exists() statt count()

Wenn Sie nur wissen müssen, ob Datensätze existieren:

```php
// Ineffizient - zählt alle Datensätze
if (User::where('email', $email)->count() > 0) {
    // ...
}

// Effizient - stoppt nach dem ersten Fund
if (User::where('email', $email)->exists()) {
    // ...
}

// Negation
if (User::where('email', $email)->doesntExist()) {
    // ...
}
```

### first() vs find()

`find()` ist optimiert für Primary Key Lookups:

```php
// Optimiert für Primary Key
$user = User::find(1);
// SQL: SELECT * FROM users WHERE id = 1 LIMIT 1

// Mit mehreren IDs
$users = User::find([1, 2, 3]);
// SQL: SELECT * FROM users WHERE id IN (1, 2, 3)

// first() für andere Bedingungen
$user = User::where('email', $email)->first();
// SQL: SELECT * FROM users WHERE email = ? LIMIT 1

// firstOrFail() wirft Exception
$user = User::where('email', $email)->firstOrFail();
```

### updateOrCreate() Optimierung

Atomare Operation zum Erstellen oder Aktualisieren:

```php
// Atomic upsert
$user = User::updateOrCreate(
    // Attributes zum Finden
    ['email' => $email],
    // Attributes zum Updaten/Erstellen
    ['name' => $name, 'active' => true]
);

// Bulk upsert (Laravel 8+)
User::upsert([
    ['email' => 'user1@example.com', 'name' => 'User 1'],
    ['email' => 'user2@example.com', 'name' => 'User 2'],
], ['email'], ['name']);
// Zweites Array: Unique keys
// Drittes Array: Zu aktualisierende Spalten
```

---

## 9. Relationship Optimierungen

### HasOne vs BelongsTo Performance

Die Richtung der Beziehung beeinflusst die Performance:

```php
// BelongsTo - Foreign Key ist auf diesem Model
class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
        // posts.user_id -> users.id
    }
}

// HasOne - Foreign Key ist auf dem anderen Model
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class);
        // users.id -> profiles.user_id
    }
}
```

**Regel:** BelongsTo ist generell performanter, da der Foreign Key direkt verfügbar ist.

### morphWith() für Polymorphe Relationen

Optimiertes Eager Loading für polymorphe Beziehungen:

```php
// Polymorphe Beziehung
class Activity extends Model
{
    public function subject()
    {
        return $this->morphTo();
    }
}

// Ineffizient - lädt alle möglichen Typen
$activities = Activity::with('subject')->get();

// Optimiert - lädt nur spezifische Typen mit ihren Relationen
$activities = Activity::with([
    'subject' => function (MorphTo $morphTo) {
        $morphTo->morphWith([
            Post::class => ['user', 'comments'],
            Comment::class => ['user', 'post'],
            Photo::class => ['album']
        ]);
    }
])->get();
```

---

## 10. Debugging & Monitoring

### DB::listen() für Query Logging

Überwachen Sie alle Datenbankabfragen:

```php
// In AppServiceProvider::boot()
DB::listen(function ($query) {
    Log::info('Query', [
        'sql' => $query->sql,
        'bindings' => $query->bindings,
        'time' => $query->time . 'ms'
    ]);
});

// Detaillierteres Logging
DB::listen(function ($query) {
    if ($query->time > 100) { // Queries über 100ms
        Log::warning('Slow query detected', [
            'sql' => $query->sql,
            'bindings' => $query->bindings,
            'time' => $query->time . 'ms',
            'connection' => $query->connectionName
        ]);
    }
});
```

### Query Log aktivieren

Für temporäres Debugging:

```php
// Query Log aktivieren
DB::enableQueryLog();

// Ihre Queries ausführen
$users = User::with('posts')->get();

// Log anzeigen
$queries = DB::getQueryLog();
dd($queries);

// Formatierter Output
foreach (DB::getQueryLog() as $query) {
    $sql = str_replace('?', '%s', $query['query']);
    $fullQuery = vsprintf($sql, $query['bindings']);
    echo $fullQuery . ' (' . $query['time'] . 'ms)' . PHP_EOL;
}
```

### Laravel Telescope

Telescope ist Laravel's offizielles Debug-Tool:

```php
// Installation
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate

// In AppServiceProvider registrieren
public function register()
{
    if ($this->app->environment('local')) {
        $this->app->register(\Laravel\Telescope\TelescopeServiceProvider::class);
        $this->app->register(TelescopeServiceProvider::class);
    }
}
```

Telescope zeigt:
- Alle Queries mit Execution Time
- Slow Queries
- Model Events
- Cache Hits/Misses
- Und vieles mehr

---

## Best Practices Zusammenfassung

### 1. Query Anzahl minimieren
- Verwenden Sie Eager Loading (`with()`) für Beziehungen
- Nutzen Sie `withCount()` statt Count-Queries in Loops
- Batch Operations verwenden (`insert()`, `update()`)

### 2. Datenvolumen reduzieren
- Nur benötigte Felder mit `select()` laden
- Pagination statt `all()` für große Datensätze
- `chunk()` oder `cursor()` für Batch-Verarbeitung

### 3. Richtige Indexierung
- Indexes auf häufig verwendete WHERE-Spalten
- Composite Indexes für kombinierte Bedingungen
- Covering Indexes für häufige Queries

### 4. Caching strategisch einsetzen
- Cache teure/häufige Queries
- Cache-Tags für einfache Invalidierung
- Cache-Warming für kritische Daten

### 5. Monitoring und Optimierung
- Laravel Telescope in Entwicklung
- Query Logging in Produktion
- Regular Performance Audits

### 6. Architektur-Entscheidungen
- Denormalisierung für Read-Heavy Tabellen
- Materialized Views für komplexe Aggregationen
- Read-Replicas für Skalierung

### Anti-Patterns vermeiden

1. **Keine Queries in Loops**
   ```php
   // Schlecht
   foreach ($users as $user) {
       $posts = Post::where('user_id', $user->id)->get();
   }
   
   // Gut
   $users = User::with('posts')->get();
   ```

2. **Kein SELECT * bei großen Tabellen**
   ```php
   // Schlecht
   $users = User::all();
   
   // Gut
   $users = User::select('id', 'name', 'email')->get();
   ```

3. **Keine ungenutzten Eager Loads**
   ```php
   // Schlecht
   $users = User::with(['posts', 'comments', 'profile'])->get();
   // Wenn Sie nur posts verwenden
   
   // Gut
   $users = User::with('posts')->get();
   ```

---

## Weiterführende Ressourcen

1. [Laravel Dokumentation - Eloquent Performance](https://laravel.com/docs/eloquent)
2. [Laracasts - Eloquent Performance Patterns](https://laracasts.com/series/eloquent-performance-patterns)
3. [Laravel Debugbar](https://github.com/barryvdh/laravel-debugbar)
4. [Laravel Telescope](https://laravel.com/docs/telescope)
5. [MySQL Index Dokumentation](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)

---

*Dieses Dokument basiert auf Laravel Best Practices und Erfahrungen aus produktiven Anwendungen. Aktualisiert für Laravel 10.x/11.x.*