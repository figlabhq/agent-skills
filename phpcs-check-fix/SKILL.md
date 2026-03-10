---
name: phpcs-check-fix
description: "Fix PHP coding style issues using PHPCS and PHPCBF. Use this skill whenever the user mentions PHPCS, code style, coding standard, cs:fix, cs:check, PHP formatting, or asks to fix/check PHP code style. Also activate when you notice PHP files have been modified and need style compliance, or when a CI PHPCS check has failed."
---

# PHPCS Check and Fix

This project uses PHP CodeSniffer (PHPCS) with the FigLab Coding Standard (which extends the IxDF Coding Standard, built on PSR + Slevomat rules). The config lives in `phpcs.xml` at the project root.

## Commands

- **Auto-fix**: `composer cs:fix` — runs PHPCBF to automatically fix what it can
- **Check**: `composer cs:check` — runs PHPCS to report remaining violations

## Workflow

Follow this exact sequence:

1. **Run `composer cs:fix`** to auto-fix as many issues as possible. This handles spacing, bracket placement, import ordering, and other mechanical fixes.

2. **Run `composer cs:check`** to see what remains. Parse the output carefully — it shows the file path, line number, error type, and the sniff name (e.g., `SlevomatCodingStandard.Complexity.Cognitive.ComplexityTooHigh`).

3. **Fix remaining issues manually**, one by one. The approach depends on the violation type — see the section below.

4. **Re-run `composer cs:check`** after manual fixes to confirm everything passes. If new issues appear, fix those too.

## Handling Common Remaining Issues

After `composer cs:fix`, these are the violations that typically need manual attention:

### Cognitive Complexity (`SlevomatCodingStandard.Complexity.Cognitive.ComplexityTooHigh`)

The project has a warning threshold of 7 and an error threshold of 10.

- **If the method is only slightly over the threshold** (e.g., complexity 8-11 on a method that's inherently branchy), add an inline ignore directive on the line **above** the method declaration:

```php
// phpcs:ignore SlevomatCodingStandard.Complexity.Cognitive.ComplexityTooHigh
public function handleComplexScenario(): void
{
    // ...
}
```

- **If the method is significantly over** (complexity 12+), consider refactoring: extract helper methods, use early returns, or simplify conditional logic. Only add the ignore directive if refactoring would make the code less readable.

### Missing Type Hints

Add proper PHP type declarations. Use union types (`string|int`) or nullable types (`?string`) where appropriate. For arrays, add PHPDoc with array shape when the structure is known:

```php
/** @param array{name: string, email: string} $data */
public function process(array $data): void
```

### Function/Class Length

If a function or class is too long, look for opportunities to extract methods or break the class into smaller focused classes. For test files, these rules are already excluded in `phpcs.xml`.

### Unused Function Parameters

Already excluded for Policy classes and factories in `phpcs.xml`. For other files, either use the parameter or prefix with underscore if it's required by a parent signature.

### Line Length

The limit is 160 characters (absolute max 200). Break long lines at logical points — method chains, array entries, or string concatenation.

## Scoping to Specific Files

When you only changed a few files, you can target the check to just those files instead of the entire project:

```bash
vendor/bin/phpcs --standard=phpcs.xml path/to/File.php
vendor/bin/phpcbf --standard=phpcs.xml path/to/File.php
```

This is faster than running the full project scan. Use this when fixing style issues in files you've just edited.

## CI Integration

The GitHub Actions workflow (`.github/workflows/phpcs.yml`) runs on every push:
1. Auto-fixes with `composer cs:fix` and commits the result
2. Runs `composer cs:check` — failures show as annotations in the PR

So if you fix everything locally, CI will pass cleanly.
