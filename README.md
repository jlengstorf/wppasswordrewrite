<p align="center">
  <a href="https://www.gatsbyjs.org">
    <img alt="Gatsby" src="https://www.gatsbyjs.org/monogram.svg" width="60" />
  </a>
</p>
<h1 align="center">
  Removing Passwords from Git’s History
</h1>

What do I do if I accidentally committed a password to my Gatsby repo?! 😱😱😱

## Step 1: Immediately change all passwords and invalidate any tokens that were committed.

Even if you remove them from history, there’s no way to guarantee that it hasn’t been copied or shared. **DO NOT SKIP THIS STEP.**

## Step 2: Remove the sensitive data from the code and make a new commit.

Your `master` branch shouldn’t have the sensitive data, so get it out of there!

See [this commit](https://github.com/jlengstorf/wppasswordrewrite/commit/4ce7e7621f348781f0d7fe478e675c2609949559) for an example.

## Step 3: Remove the sensitive data from history.

```sh
# Replace all passwords with the env var.
git ls-files -z "gatsby-config.js" | xargs -0 perl -p -i -e "s#(wpcom_user.*$)#wpcom_user: process.env.WP_USERNAME#g"
```
