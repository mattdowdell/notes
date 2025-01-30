# Git

## Config

```sh
# user setup
git config --global user.email firstname.lastname@example.com
git config --global user.name 'Matthew Dowdell'

# url aliases
git config --global url.git@github.com.insteadof gh
git config --global url.git@github.com:.insteadof https://github.com/

# command aliases
git config --global alias.co checkout
git config --global alias.st status
git config --global alias.br branch
git config --global alias.sw switch
git config --global alias.swc "switch --create"

# pull config
git config --global pull.rebase false
git confif --global push.autosetupremote true
```

## Empty commit

```sh
git commit --allow-empty -m "message"
```

## Modify git commit date

```sh
GIT_COMMITTER_DATE="$(date)" git commit --amend --no-edit --date "$(date)
```

## Soft reset commits on branch

```sh
# merge in default branch
git pull origin main

# soft reset all branch-specific commits
git reset $(git merge-base origin/main $(git branch --show-current))
```

## List vendored and generated code

```sh
git diff origin/main HEAD --name-only --no-renames | \
  git check-attr --stdin linguist-generated linguist-vendored | \
  grep ': set' | \
  awk '{ORS=" "} {gsub(":","",$1); print $1}'
```

Assumes the use of `.gitattributes` to hide certain files in pull requests (see [overrides]). The attributes should only be 'set' instead of being equal to true or false.

```
# vendored dependencies
vendor/** linguist-vendored

# generated files
mocks/** linguist-generated
go.sum   linguist-generated
```

This can be misleading when moving files to non-vendored to vendored or non-generated to generated.

[overrides]: https://github.com/github-linguist/linguist/blob/main/docs/overrides.md

## Get size of diff

```sh
git diff origin/main HEAD --numstat --ignore-space-change -- \
  . :^path/to/excluded/file | \
  awk '{sum += $1 + $2} END {print sum}'
```

This combines the total number of lines removed and lines added. In git terms, this means a single line change results in a value of 2: the line was removed, modified and then added back.
