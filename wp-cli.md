# WP-CLI Commands

The full list of built-in WP-CLI commands and examples can be found here:

https://developer.wordpress.org/cli/commands/

## WP-CLI not installed? Download and run it directly from the `phar` file

First, download wp-cli.phar using wget or curl. For example:
```
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
```
Then, check if it works:
```
php wp-cli.phar --info
```
If so, you can run any command using `php wp-cli.phar` instead of the `wp` shortcut.
https://make.wordpress.org/cli/handbook/installing/

You can also set an alias for the current shell session:
```
alias wp="php /path/to/wp-cli.phar"
alias wp="php $(pwd)/wp-cli.phar"
```

## Database Commands
WordPress sites may use a unique DB Prefix. This sets the existing prefix as a variable for all future commands in the same session:
```
PFX="$(wp db prefix)"
```
Now instead of manually typing out the custom prefix, you can use `$(PFX)` for table names (`wp_options` becomes `$(PFX)options`).

### Export Database
```
wp db export
```

### Search & Replace
Append `--dry-run` first, then if all looks good, run the command without it.

To avoid malformed URLs, have both URLs include or exclude the trailing slash!

**_Basic search and replace in all tables. Useful for http->https conversion._**
```
wp search-replace "http://domain.com" "https://domain.com" --skip-columns=guid --dry-run
```
**_Remove `/year/month/day/` from hardcoded links. Make sure all links are https: first._**
```
wp search-replace "https://domain.com/([0-9]{4})/([0-9]{2})/([0-9]{2})/" "https://domain.com/" --regex --skip-columns=guid --dry-run
```
**_Remove `/year/month/` from hardcoded links. Make sure all links are https: first._**
```
wp search-replace "https://domain.com/([0-9]{4})/([0-9]{2})/" "https://domain.com/" --regex --skip-columns=guid --dry-run
```
**_Change `/year/month/postname.html` to `/postname/` in permalinks. Make sure all links are https: first._**
```
wp search-replace "https://domain.com/([0-9]{4})/([0-9]{2})/(.*).html" "https://domain.com/\3/" --regex --skip-columns=guid --dry-run
```
Better to make non-greedy...try the below version next time we do this change:
```
wp search-replace "https:\/\/www\.domain\.com\/([0-9]{4})\/([0-9]{2})\/(.+?)\.html(\s|\/|'|\"|>)" "https://www.domain.com/\3\4" --regex --skip-columns=guid --dry-run --log=year-month-html.log
```
**_Change `/postname.html` to `/postname/` in permalinks. Make sure all links are https: first._**
```
wp search-replace "https:\/\/www\.domain\.com/(.*?)\.html(\s|\/|'|\"|>)" "https://www.domain.com/\1/\2" --regex --skip-columns=guid --log=remove-html.log --dry-run
```
**_Change `/category/postname/` to `/postname/` in permalinks. Make sure all links are https: first._**

This needs to be done separately for each hardcoded category URL, replacing the most specific categories (child categories) first. 

`(.+?)(\s|\/|'|\"|>)` [should avoid a greedy regex](https://github.com/wp-cli/search-replace-command/issues/157#issuecomment-876750207) so that all instances in an individual cell will get replaced. 

Use double-quotes around the search/replace fields for the escaping to work properly without an error. It should also work if the link is missing the trailing slash -- it looks for a delimiter of a trailing slash, single quote, double quote, space, or greater-than sign.

Note, also, the `\1\2` in the replacement - this will carry over the slug _and_ any trailing slash, quote, etc.

If there are subcategories, be sure to replace them first!

The sample code below searches only in `$(PFX)posts $(PFX)postmeta $(PFX)comments $(PFX)commentmeta $(PFX)term_taxonomy $(PFX)termmeta $(PFX)options` to speed the process.
```
PFX="$(wp db prefix)"
wp search-replace "https:\/\/domain\.com\/category-name\/subcategory-name\/(.+?)(\s|\/|'|\"|>)" "https://domain.com/\1\2" $(PFX)posts $(PFX)postmeta $(PFX)comments $(PFX)commentmeta $(PFX)term_taxonomy $(PFX)termmeta $(PFX)options --regex --skip-columns=guid --dry-run
wp search-replace "https:\/\/domain\.com\/category-name\/(.+?)(\s|\/|'|\"|>)" "https://domain.com/\1\2" $(PFX)posts $(PFX)postmeta $(PFX)comments $(PFX)commentmeta $(PFX)term_taxonomy $(PFX)termmeta $(PFX)options --regex --skip-columns=guid --dry-run
```

## Malware Cleanup

Not all client sites use the latest WordPress version or the default `en_US` locale. Use these commands to set currently installed WordPress version and active locale as variables for core reinstall commands:
```
WP_VERSION="$(wp core version)"
WP_LOCALE="$(wp language core list --status=active --field=language)"
```

Check core & (wordpress.org hosted) plugin integrity:
```
wp core verify-checksums
wp plugin verify-checksums --all --strict
```

### Core Replacement
Quickly reinstall latest version of core:
```
wp core download --force --skip-content --locale="$WP_LOCALE"
```

If infected/extra files are found in `wp-admin` or `wp-includes`, wipe those folders and then immediately reinstall. If working on a production site, run the above reinstall first to be sure that it will work:
```
rm -rf wp-admin wp-includes && wp core download --force --skip-content
```

Reinstall the currently installed WordPress core version and active locale:
```
wp core download --force --skip-content --version="$WP_VERSION" --locale="$WP_LOCALE"
```

Reinstall a specific WordPress core version and locale:
```
wp core download --force --skip-content --version="7.0" --locale="$WP_LOCALE"
```

Reset **all** user passwords. Add `--skip-email` to not send an email notification.

```
wp user reset-password $(wp user list --field=user_login)
```

Reset administrator and/or editor passwords. Add `--skip-email` to not send an email notification.

```
wp user reset-password $(wp user list --field=user_login --role=administrator)
wp user reset-password $(wp user list --field=user_login --role=editor)
wp user reset-password $(wp user list --role="administrator" --field=user_login && wp user list --role="editor" --field=user_login)
```

Reset a single user's password & display the new password (and don't send an email).

```
wp user reset-password username --show-password --skip-email
```
## List Info

Get a CSV (post_title,post_date,ID) of published posts inside a certain category.
```
wp post list --post_type=post --post_status=publish --fields=post_title,post_date,ID --format=csv --no-header --tax_query='[{"taxonomy":"category","field":"slug","terms":"<your-category>"}]'
```
