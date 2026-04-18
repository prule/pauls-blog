# Working with Git Submodules in IntelliJ / WebStorm

IntelliJ IDEA and WebStorm treat git submodules as nested repositories, allowing you to manage both the blog and the theme seamlessly.

## 1. Committing Changes
Open the **Commit Tool Window** (`Cmd+0` on macOS or `Ctrl+0` on Windows).
- IntelliJ groups changes by repository. You will see separate sections for `pauls-blog` and `themes/hugo-simple-beauty`.
- **CRITICAL**: Always commit and push the **theme** changes first. 
- After the theme is pushed, the blog repository will show a change in the `themes/hugo-simple-beauty` folder (the "gitlink"). Commit this in the blog repo to update the pointer.

## 2. Branch Management
Look at the **Git Branch Widget** in the bottom-right corner of the status bar.
- You will see multiple entries (e.g., `pauls-blog` and `hugo-simple-beauty`).
- Before editing the theme, ensure the `hugo-simple-beauty` repository is on the `main` branch. 
- If it shows a commit hash (e.g., `a1b2c3d`), it is in a "Detached HEAD" state. Click it and select `Checkout 'main'`.

## 3. Pushing to GitHub
When you use **Push** (`Cmd+Shift+K` / `Ctrl+Shift+K`):
- The dialog lists all repositories with outgoing commits.
- Ensure the theme is selected and pushes successfully before or at the same time as the main blog repo.

## 4. Pulling & Updating
If you make changes on another machine:
1. Pull changes in the main blog repo.
2. Go to **Git > Submodules > Update Submodule...** in the top menu.
3. This ensures your local theme folder matches the version tracked by the blog repo.

## 5. Troubleshooting Directory Mappings
If the theme folder doesn't look like a Git repository in the IDE:
1. Go to **Settings/Preferences > Version Control > Directory Mappings**.
2. Ensure both the project root and `themes/hugo-simple-beauty` are listed with `Git` as the VCS.
3. If the theme is missing, click `+` and add it manually.
