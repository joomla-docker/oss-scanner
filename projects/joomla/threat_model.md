# Threat model: Joomla! CMS

The scanner reads this file before it starts. It covers Joomla! CMS as developed on the default branch of
https://github.com/joomla/joomla-cms, which is always the patch branch of the current stable minor (6.2-dev when this
was written): the components, modules, plugins, templates, libraries and web services that a Joomla release ships,
the installation application and the console. The Joomla! Security Strike Team (JSST) assesses what the scanner
finds, assigns the final classification and coordinates the fix; its procedures are at
https://developer.joomla.org/security.html.

## What this project does and where untrusted input enters

Joomla is a PHP content management system on MySQL/MariaDB or PostgreSQL. One checkout serves four applications that
share the code under `libraries/` and one database:

| Application | Where | Who uses it |
| --- | --- | --- |
| Site | `index.php`, `components/`, `modules/`, `plugins/`, `templates/cassiopeia/`, `layouts/` | everyone: anonymous visitors and logged-in users |
| Administrator | `administrator/index.php`, `administrator/components/`, `administrator/modules/`, `administrator/templates/atum/` | Manager, Administrator and Super Users |
| Web services API | `api/index.php/v1/...`, `api/components/`, `plugins/webservices/`, `plugins/api-authentication/` | holders of an API token whose group may log in to the API |
| Console | `cli/joomla.php`, `libraries/src/Console/` | the server operator |

The installation application (`installation/`) installs a site. After installing, a stable release's console
installer deletes it; the web installer leaves its removal to the site owner, and a stable build refuses to run the
site while `installation/index.php` exists (`includes/framework.php`). On an installed stable build only the session
that wrote `configuration.php` may continue the installer (`InstallationApplication::isInstallerAllowed()`); that
protection is in scope. Development builds such as this checkout keep `installation/` and let anyone repeat the
installation by design, so Apache denies `/installation/` in this image.

The CMS library is `libraries/src/` (namespace `Joomla\CMS`). The Joomla Framework packages under
`libraries/vendor/joomla/` (database, filter, input, session, router, http, filesystem, archive and others) are
Joomla's own code and in scope as the CMS uses them. They are developed in the `joomla-framework` repositories and
installed by Composer: `libraries/vendor/` is not part of the joomla-cms repository, so a fix to one of them belongs to
that package (see the patch guidelines). Everything else under `libraries/vendor/` and `media/vendor/` is third-party.
JavaScript and CSS are built from `media_source/` into `media/`: a finding in a built file belongs to its source.

Untrusted input enters through:

- **Every HTTP request** to the site, the administrator or the API: the path and SEF route, query and POST data
  (`option`, `view`, `task`, `layout`, `format`, `tmpl`, `Itemid`, `return`, list filters and `list[ordering]`),
  cookies, the `Host` and proxy headers, and uploaded files.
- **Content stored by one user and shown to another**: articles, categories, tags, contacts, banners, newsfeeds, menu
  items, custom field values (`com_fields`, `plugins/fields/`), user names and profile data, user notes, redirects,
  mail templates, and the site, template and extension settings an administrator edits. Super Users and
  administrators see what lower-privileged users wrote, in lists, in editors and in previews.
- **Text filtering**: what a user may store depends on their group's text filter (Global Configuration › Text
  Filters, applied through `Joomla\CMS\Filter\InputFilter`). The stock filters are in the table below. Getting script
  past a group's filter is a finding; so is output that escapes nothing where the filter did not run.
- **Files**: media manager uploads (`com_media`, `plugins/filesystem/local`, `Joomla\CMS\Helper\MediaHelper`) and the
  extension, language pack and template uploads that Super Users make.
- **Authentication flows**: login (site, administrator and API), the remember-me cookie, registration and account
  activation, password reset, multi-factor authentication (`plugins/multifactorauth/`), passwordless login
  (`plugins/system/webauthn`) and API tokens (`plugins/user/token`, `plugins/api-authentication/token`).
- **Server-side requests**: update servers and Install from URL (Super Users), the Joomla update itself, which TUF
  protects (`php-tuf`), newsfeeds (the site fetches each newsfeed's URL), the Requests scheduled task
  (`plugins/task/requests`), and the HTTP client in `libraries/vendor/joomla/http`.

### Trust model

Joomla's access control is user groups, access levels and per-asset permissions
(`administrator/components/*/access.xml`, the `#__assets` table, `Joomla\CMS\Access`). A stock installation seeds
these groups, permissions and text filters (`installation/sql/mysql/base.sql`, `extensions.sql`):

| Group | Trust | Text filter |
| --- | --- | --- |
| Public, Guest (anonymous) | Untrusted. | No HTML |
| Registered | Can log in to the site and edit their own profile. Untrusted beyond that. | No HTML |
| Author, Editor, Publisher | The site content workflow: create (Author), edit (Editor) and publish (Publisher) content from the site. No administrator login, no API. | Default Forbidden List |
| Manager | Administrator login. `core.manage` on banners, contacts, content (articles, featured, their categories and fields), media, newsfeeds and Smart Search, with create, edit, publish and delete on what it reaches. Not menus, modules, users, plugins, templates, extensions or any configuration. | Default Forbidden List |
| Administrator | `core.manage` on every component except the installer, and `core.admin` (options and permissions) on 16: users, menus, modules, plugins, templates, languages, content, media, banners, contacts, newsfeeds, Smart Search, redirects, cache, check-in and messages. Cannot install or remove extensions, use Joomla Update, Global Configuration, Privacy or User Actions Log, edit template files or overrides (`TemplateController::allowEdit()` requires `core.admin` on the root), or edit Super Users. Escalation from Administrator to Super User or to code execution is a finding. | Default Forbidden List |
| Super Users | Trusted with the whole site. Install extensions, edit template files, change any setting: each runs code by design. A finding that needs Super User access is not a vulnerability. | No Filtering |

The server operator is trusted: `configuration.php`, the file system, the database and the console. Third-party
extensions are out of scope and go to the Vulnerable Extensions List (https://vel.joomla.org/). Third-party libraries
go upstream; report them here only when Joomla's use of a library creates the vulnerability.

## Components that matter most / least

Highest priority:

1. **Access control.** `authorise()` and `canDo` checks in controllers and models, access levels on items and
   categories, `core.edit.own` against other users' items, user and group management (`com_users`: who may edit whom,
   assign which groups, reset which passwords), batch and check-in tasks, and the `#__assets` tree itself. In the API,
   `ApiDispatcher` checks `core.api` on the component and `ApiController` checks create, edit, edit own and delete;
   look for endpoints or models that skip a check the administrator applies to the same action. The API does not
   check `core.manage`, and reads are not filtered by state or access level, by design of `core.api` (see
   "How to exercise it").
2. **Cross-site scripting.** Output that bypasses `htmlspecialchars` and `HTMLHelper` escaping, `Text::_()` strings
   built from input, the text filter configuration and `InputFilter`, SVG handling (`MediaHelper::isValidSvg()` and
   `InputFilter`'s checks of SVG `data:` URIs, both through `enshrined/svg-sanitize`), custom field rendering, the media
   manager and its previews, the TinyMCE and CodeMirror integrations, JavaScript in `media_source/`, mail templates,
   error pages and template overrides.
3. **SQL injection.** The query builder in `libraries/vendor/joomla/database` as the CMS uses it: concatenated input
   in `where()` and `order()`, list ordering and direction outside `filter_fields`, `escape()` misuse, Smart Search
   (`com_finder`) query building, and the PostgreSQL driver paths, which this image cannot run.
4. **Files.** Path traversal, and the extension and MIME checks in `MediaHelper::canUpload()`: the executable blocklist
   (`MediaHelper::EXECUTABLES`, `InputFilter::FORBIDDEN_FILE_EXTENSIONS`), which applies whatever the media options
   say, and the options `restrict_uploads`, `restrict_uploads_extensions`, `ignore_extensions`, `check_mime`,
   `upload_mime` and `image_extensions`. SVG is not an allowed upload by default. Also `Joomla\Filesystem`
   (`libraries/vendor/joomla/filesystem`), archive extraction (`libraries/vendor/joomla/archive`) and `tmp/` handling.
5. **Authentication and sessions.** The login flows of all three web applications, the remember-me cookie, password
   reset and activation tokens, multi-factor enrolment and verification, WebAuthn, API token checks, session storage
   (`#__session`, `libraries/vendor/joomla/session`), CSRF tokens (`Session::checkToken()`,
   `HTMLHelper::_('form.token')`) on every state change, and open redirects through `return` parameters
   (`Uri::isInternal()`).
6. **Deserialisation and mass assignment.** `unserialize()` on anything an attacker influences, `Registry` and JSON
   params parsing, `Table::bind()` and form filtering that could set fields a user may not set (`block`, `groups`,
   `sendEmail`, `activation`), and `filter="raw"` form fields.
7. **Server-side requests.** Managers choose newsfeed URLs and Administrators the URLs of Requests scheduled tasks, by
   design. The finding to look for is an anonymous visitor or a group below Manager making the site fetch a URL of
   their choice, or a fetch that reaches further than the feature intends.

Lower priority or out of scope:

- What a release does not ship (`build/build.php` lists it): `build/`, `tests/`, `media_source/`, `node_modules/`,
  `.github/`, the tooling configuration at the top of the checkout (`composer.*`, `package*.json`, `phpunit*.xml*`,
  `phpstan*.neon`, `ruleset.xml`, `.php-cs-fixer.dist.php`, `cypress.config.dist.mjs`), and `.oss-scanner/`. Apache
  serves none of them here.
- Composer's development packages, installed here for testing: PHPUnit, PHP CS Fixer, PHP_CodeSniffer, PHPStan,
  `joomla/test` and `joomla/mediawiki`. A release installs none of them, and `libraries/.htaccess` keeps all of
  `libraries/` off the web, here as on a release.
- `libraries/vendor/` other than the `joomla/*` packages above, and `media/vendor/`: upstream, unless Joomla's use
  creates the issue.
- The Cypress System tests under `tests/System/`.
- Third-party extensions and live Joomla sites.

## How to exercise it

- Run `joomla-start` first. It starts MariaDB and Apache (the container has no init system) and is safe to run again.
  `joomla-stop` stops both; if one is still running a minute later, it says which and exits 1. The site is
  `http://127.0.0.1/`, the administrator `http://127.0.0.1/administrator/`, the API
  `http://127.0.0.1/api/index.php/v1/`. Apache listens on the loopback interface only. At the top level of the
  checkout it serves only what a Joomla release ships (`build/build.php`); any other top-level file or directory gets
  403, including one created after the build, such as a `sitemap.xml` or a file an exploit writes. A finding that
  writes a file there is still a finding: confirm the write on disk.
- The site is installed with the stock profile and no sample data. One account per core group exists; every password
  is `<login>-password`:
  - `superuser` (Super Users), `administrator` (Administrator), `manager` (Manager), `publisher` (Publisher),
    `editor` (Editor), `author` (Author), `registered` (Registered); anonymous is the Guest group.
- The API differs from a stock site, where Super Users alone may log in to it and use its web services (`core.api`).
  Here the other two groups that log in to the administrator may use it too, for what the administrator lets them
  manage: Manager and Administrator have `core.login.api`; Administrator has `core.api` everywhere but the installer,
  which the administrator denies it; Manager has `core.api` on banners, contacts, content, media and newsfeeds. API
  tokens are enabled for these three groups. Author, Editor, Publisher and Registered cannot use the API, as on a
  stock site. Reading through the API what the administrator shows the same user is not a finding. Name these grants
  when a finding depends on them, and name any further grant you make.
- `joomla-api-token <login>` prints a user's API token, creating one if they have none. For example:
  `curl -H "Authorization: Bearer $(joomla-api-token manager)" http://127.0.0.1/api/index.php/v1/content/articles`.
  Basic authentication for the API is disabled, as on a stock site.
- `joomla-permission <asset> <action> <group id>=<1|0|->` changes a group's permission on an asset (`root.1`, a
  component such as `com_users`, or an item such as `com_content.article.1`), as the Permissions tabs do; Joomla has
  no console command for it. Group ids: Public 1, Registered 2, Author 3, Editor 4, Publisher 5, Manager 6,
  Administrator 7, Super Users 8, Guest 9.
- Emails are not sent; each one lands in `/var/mail/joomla/` as an `.eml` file, so password reset links,
  activation links and multi-factor codes can be read.
- The database is `joomla` on MariaDB, user `joomla` / `joomla`, table prefix `jos_`, over the unix socket;
  `mariadb joomla` opens a client as root. `joomla_test` is for the Integration suite. The configuration is
  `/src/configuration.php`.
- `joomla-console <command>` runs `cli/joomla.php` as www-data: `user:add`, `user:addtogroup`, `user:list`,
  `extension:list`, `config:get`, `config:set`, `cache:clean`, `site:down`, `site:up`, `scheduler:run` and the others
  `joomla-console list` shows. `as-www-data <command>` runs anything else as the web server's user.
- Patches go to `/src`. A PHP change takes effect on the next request. A JavaScript or SCSS change goes to
  `media_source/` and is rebuilt offline with `as-www-data npm run build -- -a` (everything) or
  `as-www-data npm run build -- -n <resource>` (one resource).
- Tests: `joomla-phpunit --testsuite Unit` runs the CMS unit suite, `joomla-phpunit --testsuite Integration` the
  integration suite against `joomla_test` (its LDAP tests are skipped: there is no LDAP server). Both run offline. The
  Cypress System suite needs a browser and does not run here; reproduce through curl or PHP instead. The build ran
  both PHPUnit suites without failing on red tests: `/var/log/phpunit-results.txt` holds each suite's exit code and
  `/var/log/phpunit-Unit.log` and `/var/log/phpunit-Integration.log` their output. A log the build could not write in
  full is incomplete, and the build printed a warning naming it.
- Logs: PHP and Apache errors in `/var/log/apache2/error.log`, requests in `/var/log/apache2/access.log`, MariaDB's
  in `/var/log/mysql/error.log`, Joomla's own in `/src/administrator/logs/`. Debug mode and error display are off, as
  on a production site.
- Stock settings worth knowing: user registration is off (Users › Options); `htaccess.txt` is not active (copy it to
  `/src/.htaccess` to test the recommended production configuration); the HTTP Headers plugin is enabled with its
  defaults; the Smart Search index is empty until content is saved (the Content - Smart Search plugin indexes on save).

## How you rate severity

The JSST classifies a confirmed issue and may move it up or down. For the scanner's severity field use this scale,
worded in this file's group terms:

- **Critical**: anonymous visitors or Registered users can run code on the server, inject SQL, bypass authentication
  or take over a Super User account, in a stock configuration.
- **High**: SQL injection, authentication bypass or account takeover that needs a site content group (Author, Editor,
  Publisher) or a common non-default setting; stored XSS that anonymous, Registered or site content users can plant
  where Managers, Administrators or Super Users see it; escalation from Manager or Administrator to Super User or to
  code execution; reading or changing other users' private data at scale.
- **Medium**: XSS that needs Manager or Administrator rights or an uncommon setting; CSRF on a state-changing action;
  access control slips that expose unpublished or access-restricted content or let a group act outside its
  permissions; path traversal that reads non-sensitive files; open redirects; denial of service from a single request.
- **Low**: findings that need a configuration most sites do not run, information disclosure of little value, and
  self-XSS.

Always state the attacker's starting position in the group table's terms, whether the stock configuration suffices,
and any setting, extension or permission the finding depends on. Assess the stock installation first.

## Anything to leave alone

- Joomla's sample data plugins (`plugins/sampledata/`).
- Host header injection.
- Anything that needs Super User access, and what Super Users can do by design: unfiltered HTML, the template file
  editor, extension installation and Install from URL.
- What Administrators can do by design: make the site request a URL through a Requests scheduled task; change the
  media options, including the allowed extensions and MIME checks (getting a server-executable file past the
  executable blocklist is still a finding); configure plugins, modules, menus, template styles, mail templates and
  the options and permissions of the components they administer. And what Managers can: set newsfeed URLs the site
  fetches.
- Version disclosure: `administrator/manifests/files/joomla.xml`, the generator meta tag, `README.txt`, `LICENSE.txt`,
  `htaccess.txt`, `web.config.txt` and `robots.txt.dist` are public by design.
- Missing or weak HTTP security headers, the absence of login rate limiting or account lockout, and other hardening
  that the site owner configures (the HTTP Headers plugin, `.htaccess`, the web server).
- Findings that need `$debug = true`, `display_errors`, maximum error reporting, or No Filtering for a group below
  Super Users.
- What a release does not ship, Composer's development packages, and on this development checkout the presence of
  `installation/` (see above).
- Third-party extensions (the Vulnerable Extensions List), third-party libraries under `libraries/vendor/` and
  `media/vendor/` unless Joomla's use of them is the problem, and volumetric denial of service.
