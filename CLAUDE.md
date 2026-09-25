# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A small Symfony bundle (`deileo/pager-bundle`) that paginates a Doctrine ORM `QueryBuilder`, with URL-driven filters and sorters rendered through Twig helpers. The design goal (per README) is to stay tiny and backward compatible: it only ever paginates a Doctrine ORM `QueryBuilder`, and only one pagination per request is supported because the URL query parameter names are fixed.

Supported range is wide — PHP >= 7.1, `symfony/framework-bundle` 2.3–7.4, Doctrine ORM 2.3+ and 3.7+, Twig 1.34/2/3 — so avoid language features or framework APIs that would break older versions in that range (e.g. use explicit `?Type` nullable parameters, no newer-PHP-only syntax).

## Commands

- `make` (or `make example`) — runs `composer install`, rebuilds the SQLite demo DB (`example/app/pager.db3`) with schema + fixtures, clears cache and starts the demo app at http://localhost:8000.
- There is no test suite: `make test` is a placeholder (`echo "todo"`) even though PHPUnit is listed in `require-dev`. No linter is configured.

Note: `require-dev` pins the demo to Symfony 3.3 / Doctrine ORM 2.5, so the example app exercises the bundle against old versions only.

## Architecture

- `src/DataDog/PagerBundle/Pagination.php` — the whole paging engine. It extends `\ArrayIterator`, so the object is passed straight to templates and iterated. The constructor does all the work eagerly:
  1. Merges `Request` query params + route attributes (dropping `_`-prefixed keys) into `$query`; `filters` from the URL are merged over default `filters` option, while URL `sorters` *replace* default `sorters`.
  2. Clones the QB, applies filters and sorters, builds a `COUNT(DISTINCT rootAlias)` counter (or the `applyCounter` callable), clamps `page`/`limit` (`limit` capped by static `$maxPerPage`), then fetches the page of results.
  - Filter handling: a value equal to static `$filterAny` (`'any'`) is skipped. If an `applyFilter` callable is given it handles **all** filters (default `eq`/`in` handling is disabled). Default handling builds a parameter name from the key via `preg_replace('/[^A-z]/', '_', $key)`.
  - Sorter handling: an `applySorter` callable is called first; the default `addOrderBy` is skipped only if the callable changed the DQL.
  - ORM 3.7 deprecates string sort directions in favour of PHP's global `\SortDirection` enum (polyfilled by `symfony/polyfill-php86`); `sortDirection()` detects support by reflecting on `QueryBuilder::addOrderBy()` rather than checking the enum exists, since PHP 8.6 ships the enum even with ORM 2.
  - Global defaults are mutated via static `Pagination::$defaults` / `$maxPerPage` / `$filterAny` (e.g. in the front controller).
- `Twig/PaginationExtension.php` — Twig functions `pagination`, `sorter_link`, `filter_select`, `filter_search`, `filter_dropdown`, `filter_uri`, `filter_is_active`. URLs are generated with the router from `$pagination->route()` + `$pagination->query()`, always resetting `page` to 1 on filter/sort change. Templates are referenced as `@DataDogPager/...`.
- `Resources/config/twig.yml` — registers the extension as `datadog.pager.twig_extension`; its class comes from the `datadog.pager.twig_extension.class` parameter, which is the documented extension point for users to subclass and add their own filter functions. Keep that parameter name and the extension's protected API stable.
- `Resources/views/` — Bootstrap/FontAwesome-based templates meant to be overridden by apps.
- `example/` — a standalone Symfony 3-style demo app (autoloaded via `autoload-dev` with an empty PSR-4 prefix) showing custom `applyFilter` handlers, select/search filters and preserving pagination state in links (`ProjectController`).
