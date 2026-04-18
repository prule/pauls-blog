# Theme Maintenance Workflow

Since the `hugo-simple-beauty` theme is managed as a **git submodule**, follow this workflow to make changes and keep everything in sync.

## 1. Development
Make changes directly to the files inside `themes/hugo-simple-beauty/`. Use `hugo server` at the blog root to preview changes in real-time.

## 2. Saving Theme Changes
Once you are happy with the changes, you must commit them **inside** the theme directory first.

```bash
# Move to the theme directory
cd themes/hugo-simple-beauty

# Ensure you are on the main branch (to avoid detached HEAD)
git checkout main

# Commit and push the theme changes
git add .
git commit -m "Describe your theme changes"
git push origin main
```

## 3. Updating the Blog Repository
The main blog repository tracks a specific "pointer" (commit hash) of the theme. After pushing theme changes, you need to update this pointer in the main repo.

```bash
# Move back to the blog root
cd ../..

# Stage the submodule update
git add themes/hugo-simple-beauty

# Commit and push the blog update
git commit -m "Update theme submodule"
git push origin main
```

## 4. Pulling Theme Changes (on another machine)
If you've made changes to the theme elsewhere and want to pull them into your current blog setup:

```bash
git submodule update --remote --merge
```

## Pro Tips
- **Detached HEAD**: If `git status` inside the theme folder says "not on a branch", run `git checkout main` before committing.
- **Independence**: Remember that the theme is its own repository. It can be used by other Hugo sites independently.
