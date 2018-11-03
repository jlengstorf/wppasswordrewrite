<p align="center">
  <a href="https://www.gatsbyjs.org">
    <img alt="Gatsby" src="https://www.gatsbyjs.org/monogram.svg" width="60" />
  </a>
</p>
<h1 align="center">
  Removing Passwords from Git’s History
</h1>

What do I do if I accidentally committed a password to my Gatsby repo?! 😱😱😱

## Step 1: Create a `.env` file and initialize it

Create a new file in the root of your project (e.g. next to `gatsby-config.js`) and name it `.env`.

Next, edit `.gitignore` to ignore `.env`:

```diff
  # Project dependencies
  .cache
  node_modules
  yarn-error.log

  # Build directory
  /public
  .DS_Store

+ # Sensitive data
+ .env
```

Now that we’re sure the `.env` file won’t end up on GitHub, let’s add our secret values to it:

```env
WP_CLIENT_ID=12345
WP_CLIENT_SECRET=mysupersecrettokenthatshouldnotbepublic
WP_USERNAME=myemailthatdoesnotwantspam@gmail.com
WP_PASSWORD=Ohnothisshouldnotbecommitted!
WP_HTACCESS_USER=secretuser
WP_HTACCESS_PASS=secretpass
```

## Step 2: Read the sensitive data from the env vars

Edit `gatsby-config.js` to read in the `.env` file:

```diff
+ require('dotenv').config()
+
  module.exports = {
    // ... the rest of the config goes here
```

This means that all of the contents of `.env` are now available as properties of `process.env` (e.g. `process.env.WP_USERNAME`).

> **NOTE:** Gatsby already includes `dotenv`, so you don’t need to explicitly install it. However, you may want to do so to be extra super sure it’s available:
>
> ```sh
> npm install --save dotenv
> ```

## Step 3: Test the `develop` and `build` steps.

To make sure everything is working, run `gatsby develop` to make sure your site loads as expected.

If development mode is working normally, double check that `gatsby build` also runs without error, and that serving the site with `gatsby serve` shows the proper data.

Once you’ve confirmed that both are working properly, you can safely rewrite your Git history to remove all sensitive data!

## Step 4: Change all passwords and invalidate any tokens that were committed.

Even if you remove them from history, there’s no way to guarantee that it hasn’t been copied or shared. **DO NOT SKIP THIS STEP.**

## Step 5: Remove the sensitive data from history.

Now that we know that our app is working, we can rewrite our Git history to remove the sensitive data!

We do this by using Git’s [`filter-branch` command](https://git-scm.com/docs/git-filter-branch), which lets us modify the revision history. Don’t worry if this seems like magic; what this means is that we’re making sure we erase the password _entirely_ from GitHub.

We find the `gatsby-config.js` file in history, then use some fancy command line footwork to find and replace the sensitive data with our env vars.

The command is built like this:

```
git filter-branch --tree-filter 'git ls-files -z "<file>" | xargs -0 perl -p -i -e "s#PATTERN#REPLACEMENT#g"' -- --all
```

<details>
  <summary><strong>Expand for more details on how this command works.</strong></summary>
<br />

To break that down a bit:

`git ls-files -z "<file>"` allows us to match a file or files using a pattern. In our case, we want to use `"gatsby-config.js"` to match just that file.

The resulting list of files is then [piped](http://tldp.org/HOWTO/Bash-Prog-Intro-HOWTO-4.html) to the next command.

`xargs -0 perl -p -i -e "s#PATTERN#REPLACEMENT#g"` uses [`xargs -0`](http://man7.org/linux/man-pages/man1/xargs.1.html) to get the input from `git ls-files` and run a regular expression find-and-replace inside it. We use [Perl Pie](http://technosophos.com/2009/05/21/perl-pie-if-you-only-learn-how-do-one-thing-perl-it.html) for this, which is a common way to handle find-and-replace actions from the command line.

The final command, while long and complex, is basically saying:

1. I want to rewrite history in the following files
2. Filter the files to rewrite down to only `gatsby-config.js`
3. Inside `gatsby-config.js`, I want to replace all instances of `PATTERN` with `REPLACEMENT`

</details>

Our `<file>` will always be `gatsby-config.js`.

Our patterns will need to change depending on the sensitive data we need to rewrite.

In this WordPress example, we need the following four patterns:

- `s#(wpcom_app_clientSecret.*$)#wpcom_app_clientSecret: process.env.WP_CLIENT_SECRET,#g`
- `s#(wpcom_app_clientId.*$)#wpcom_app_clientId: process.env.WP_CLIENT_ID,#g`
- `s#(wpcom_user.*$)#wpcom_user: process.env.WP_USERNAME,#g`
- `s#(wpcom_pass.*$)#wpcom_pass: process.env.WP_PASSWORD,#g`
- `s#(htaccess_user.*$)#htaccess_user: process.env.WP_HTACCESS_USER,#g`
- `s#(htaccess_pass.*$)#htaccess_pass: process.env.WP_HTACCESS_PASS,#g`

The `.*$` in these patterns means, “After this text, match everything to the end of the line.”

Here are the full commands for replacing hard-coded sensitive data in the WordPress source plugin with the env variable.

```sh
# Replace the client secret with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(wpcom_app_clientSecret.*$)#wpcom_app_clientSecret: process.env.WP_CLIENT_SECRET,#g"' -- --all

# Replace the client ID with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(wpcom_app_clientId.*$)#wpcom_app_clientId: process.env.WP_CLIENT_ID,#g"' -- --all

# Replace the username and password with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(wpcom_user.*$)#wpcom_user: process.env.WP_USERNAME,#g"' -- --all

# Replace the password with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(wpcom_pass.*$)#wpcom_pass: process.env.WP_PASSWORD,#g"' -- --all

# Replace the .htaccess username with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(htaccess_user.*$)#htaccess_user: process.env.WP_HTACCESS_USER,#g"' -- --all

# Replace the .htaccess password with the env var.
git filter-branch -f --tree-filter 'git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(htaccess_pass.*$)#htaccess_pass: process.env.WP_HTACCESS_PASS,#g"' -- --all
```

## Step 6: Force push to the remote branch

> **HUGE FLASHING NEON WARNING:** Rewriting history is a big deal, so if you’re not the only one working on this repository, let your team know this is about to happen _before_ you do it. Everyone working on this codebase will need to pull a fresh copy of the commit history from `master` after it’s rewritten.

Once history has been rewritten, you need to push the new history up to GitHub. GitHub will not like this, because rewriting history is generally frowned upon. However, it’s the only way to get rid of those passwords and other sensitive details, so we have no choice but to be rebels and do it anyways. 😈

To finalize the rewrite and change history on GitHub, force push your branch:

```
git push origin master --force
```

Once this is done, your history will be rewritten and the evidence will be destroyed.

**Remember to change your passwords and tokens to something different than what was committed,** and you’re now no longer at risk of bad people doing bad things with your sensitive data!
