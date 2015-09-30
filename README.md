# Rust docs on GitHub Pages via Travis

This is based on [hoverbear's
instructions](http://www.hoverbear.org/2015/03/07/rust-travis-github-pages/),
but it's more secure against a compromise of your Travis builder. As far as I
know, a GitHub personal access token cannot be restricted to a scope narrower
than "all public repositories". I don't want Travis to be allowed to push to
all of my repos, just to a designated docs repo.

This is especially true because some of the repos I want to document are owned
by organizations on GitHub. I don't want organization members to have push
access to my personal repos, but as far as I know there's no way to generate
a personal access key for an organization.

So, this is what I did:

## One time setup

1. Create a [new SSH keypair](https://help.github.com/articles/generating-ssh-keys/)
   somewhere *other* than `~/.ssh`. Don't specify a passphrase. This key will be used
   just for documentation uploads.

2. Create a GitHub repo for hosting docs, and add that SSH key as a [deploy key](https://developer.github.com/guides/managing-deploy-keys/#deploy-keys) on the repo. Be sure to create and push a gh-pages branch on the repo.

3. [Install the Travis CI gem, and log into GitHub](http://docs.travis-ci.com/user/encrypting-files/#Preparation).

## For each project

### Step 1

Run this command in a directory named `scripts` in your Rust project:

```
travis encrypt-file ~/path-to-docs-key/.../id_rsa
```

Travis will guess your repo's name on GitHub. Make sure this is correct, especially if your repo is
owned by an organization!

This command should create `id_rsa.enc` inside `scripts`. Check this file
into Git; *do not* check in `id_rsa`! Keeping it outside the source tree, as
illustrated above, is safest.

The command will also print an instruction like

```
Please add the following to your build script (before_install stage in your .travis.yml, for instance):

   openssl aes-256-cbc -K $encrypted_0a6446eb3ae3_key -iv $encrypted_0a6446eb3ae3_key -in super_secret.txt.enc -out super_secret.txt -d
```

You don't need to do this. Just save the hex ID of the file, which in the above example is `0a6446eb3ae3`.

### Step 2

Also in `scripts`, create a file [`travis-doc-upload.cfg`](https://github.com/kmcallister/futf/blob/master/scripts/travis-doc-upload.cfg) similar to the following:

```
PROJECT_NAME=futf
DOCS_REPO=kmcallister/docs.git
SSH_KEY_TRAVIS_ID=0a6446eb3ae3
```

`PROJECT_NAME` is a subdirectory in your docs repo, which is identified by `DOCS_REPO`.
`SSH_KEY_TRAVIS_ID` is the hex ID generated in the previous step.

### Step 3

Add this line to your `.travis.yml`:

```
after_success: curl https://raw.githubusercontent.com/kmcallister/travis-doc-upload/master/travis-doc-upload.sh | sh
```

You can also host the script yourself, or check it into your repository. I'm not making any hard guarantees about backwards compatibility of my hosted version, although I'll try to avoid egregious breakage.

Also make sure that `cargo doc` is part of your build recipe.

### The result

After a successful Travis build, you'll find docs hosted at

```
https://[username].github.io/[doc repo name]/[project name]/[crate name]/index.html
```

"[project name]" is the `PROJECT_NAME` you defined in the config file, and "[crate name]" is a directory created by `rustdoc` — a single `rustdoc` invocation can generate docs for many crates.

See for example [`https://kmcallister.github.io/docs/futf/futf/index.html`](https://kmcallister.github.io/docs/futf/futf/index.html).

## Caveats

It's a little complicated!

There's a possible race condition if two builders try to push to the docs repo
at the same time. The script doesn't use force-push, so one of them will simply
fail. However this means you'll have stale docs until the next build.

The docs repo can get really big, because it contains the full `rustdoc` output
for every project. It would be great to de-duplicate files, or even to put all
crates in the same directory. The downside is a higher chance of breaking the
upload script or the functionality of the resulting docs (e.g. in-browser
search). The contents of identical files are already de-duplicated in Git and
maybe that's good enough.

Cross-linking between docs from different builds would be even better but, to
my knowledge, would require substantial `rustdoc` work. We would also lose the
ability to upload docs built with various different `rustc` / `rustdoc`
versions.
