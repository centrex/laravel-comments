# Known Issues — laravel-comments

_Last checked: 2026-08-02_

## Failing tests

The entire test suite fails to even boot. `composer test:unit` (`vendor/bin/pest -p`) aborts with:

```
PHP Fatal error:  Cannot redeclare class TestCase (previously declared in
/home/octopus/lab/laravel_plugins/laravel-comments/tests/TestCase.php:10) in
/home/octopus/lab/laravel_plugins/laravel-comments/tests/TestCase.php on line 10
Script pest -p handling the test:unit event returned with error code 255
```

Root cause: `tests/TestCase.php:10` declares `abstract class TestCase extends Orchestra\Testbench\TestCase` with **no namespace statement** — it lives in the global namespace. But `composer.json`'s `autoload-dev` PSR-4 map declares `"Centrex\\LivewireComments\\Tests\\": "tests/"`, and `tests/Pest.php:5` does `use Centrex\LivewireComments\Tests\TestCase;` then `uses(TestCase::class)->in(__DIR__)`. When Pest's `TestRepository` resolves `Centrex\LivewireComments\Tests\TestCase` via the Composer autoloader, it `include`s `tests/TestCase.php` a second time (the first time via normal file scanning) because the namespaced class was never actually declared in that file — triggering the "Cannot redeclare class TestCase" fatal. As a result, **no test in the suite can run** (`composer test`, `composer test:unit`, and `vendor/bin/pest` all fail identically).

Fix pointer: add `namespace Centrex\LivewireComments\Tests;` to the top of `tests/TestCase.php` so it matches the PSR-4 autoload mapping and the `use` statement in `tests/Pest.php`.

## Style / static-analysis debt

- `vendor/bin/pint --test` passes clean — no style fixes needed.
- `vendor/bin/rector --dry-run` reports no pending refactors — clean.
- PHPStan (`level: max`) reports **54 errors**. `phpstan-baseline.neon` exists but is **empty (0 lines)**, so all 54 errors are live/unbaselined. Notable clusters:
  - `src/Scopes/HasLikes.php` (mixed into `Comment`) — `likes()` return type doesn't specify `HasMany` generics; `where('comment_id', ...)` calls use string column names PHPStan can't verify as model properties; `removeLike()` is declared to return `bool` but actually returns `mixed`.
  - `src/Traits/HasUserAvatar.php:11` — accesses `$this->email` on `Centrex\LivewireComments\Models\User`, but `$email` is not declared as a property/accessor on that model, so PHPStan flags it as undefined.

## TODO / FIXME markers

None found (`grep -rn "TODO\|FIXME" --include="*.php" src/ config/ database/` — no matches).

## Open GitHub issues

Not checked — the `gh` CLI is not installed in this environment.
