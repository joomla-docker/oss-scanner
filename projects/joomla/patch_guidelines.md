# Proposed patches for Joomla CMS

Prepare a focused diff against the tested CMS revision in `/src` for private JSST review. A fix to a Joomla
Framework package under `libraries/vendor/joomla/<package>` belongs to that package's `joomla-framework`
repository: `libraries/vendor/` is not part of the CMS repository, so give that diff against the package and the
version `composer.lock` names. Explain the cause and why
the change corrects it. Preserve the stable branch's public interfaces and existing behavior for legitimate callers.
Follow the patterns of the surrounding component or library and keep unrelated formatting out of the diff.

Follow the checkout's PHP coding standards: `ruleset.xml` and `.php-cs-fixer.dist.php`, checked offline with
`./libraries/vendor/bin/phpcs --standard=ruleset.xml <file>` and
`./libraries/vendor/bin/php-cs-fixer fix --dry-run --diff <file>`. A JavaScript or SCSS fix belongs in
`media_source/`, not in the built `media/` files. For a CMS change, place a focused regression test alongside the
closest existing tests: `tests/Unit` for isolated PHP behavior, `tests/Integration` for behavior that needs the
database, both run with `joomla-phpunit`; name the relevant `tests/System` suite when the behavior is visible only
through a browser. For a Framework change, write the test in the package's own layout and PHPUnit configuration, from
its `joomla-framework` repository at the version `composer.lock` pins (the image's copy of the package has its source
but not its tests), and state which repository's tests you ran, if any.

Include testing instructions, the result before the patch, the expected result after it, and the checks actually run.
Where permissions are involved, cover both the restricted case and a user who should retain access. Identify
AI-assisted changes as proposals so a Joomla contributor can review and validate them under the project's policy.

[CMS contribution guide](https://github.com/joomla/joomla-cms/blob/6.2-dev/.github/CONTRIBUTING.md) · [CMS pull request template](https://github.com/joomla/joomla-cms/blob/6.2-dev/.github/PULL_REQUEST_TEMPLATE.md) · [Generative AI policy](https://developer.joomla.org/generative-ai-policy.html)
