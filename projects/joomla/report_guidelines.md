# Reports for the Joomla! Security Strike Team

Use a short title that names the affected CMS component, plugin or library and describes the observed behavior.

- **Impact.** Explain who can do what, whose data or permissions are affected, and which Joomla behavior should have
  prevented it. Give the attacker's starting position as one of the groups in `threat_model.md` (anonymous, Registered,
  Author, Editor, Publisher, Manager, Administrator) and say whether a stock installation suffices.
- **Tested environment.** Give the Joomla CMS version and commit. Identify the application (Site, Administrator, API or
  Console) and any PHP or database details that affected the result. State which versions were tested and label any
  suspected version range as an inference.
- **Evidence.** Describe the steps from the image's installed site: the account or group used (`threat_model.md` lists
  the image's accounts and how to get an API token), any setting, permission or content that differs from the stock
  site, the request made, the observed and the expected outcome. Name the image's API grants (`threat_model.md` lists
  them) when the finding depends on them. Leave passwords, API tokens and session cookies out of the report: in a curl
  command, write `$(joomla-api-token <login>)` instead of a token. Identify the affected source file and method. A curl
  command or a PHPUnit test is the preferred reproducer.
- **Source and proposed fix.** If the finding was previously reported, cite its source. If a patch is available,
  explain what it corrects and give the results of the tests run against it.

[JSST reporting procedures](https://developer.joomla.org/security.html) · [Joomla CMS security policy](https://github.com/joomla/joomla-cms/security/policy)
