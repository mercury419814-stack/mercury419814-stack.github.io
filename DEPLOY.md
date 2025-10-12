Quick guide to deploy this Jekyll site to GitHub Pages using GitHub Actions

1. Create a GitHub repository
   - On GitHub, create a new repository. If you want a user/organization site, name it `yourusername.github.io`.

2. Push this local code to GitHub
   ```bash
   git remote add origin git@github.com:YOUR_USER/REPO_NAME.git
   git branch -M main
   git push -u origin main
   ```

3. Let Actions build & deploy
   - The included Actions workflow at `.github/workflows/deploy.yml` runs on push to `main` or `master`. It will build the site and publish the generated `_site` to `gh-pages` branch.

4. Configure GitHub Pages (if needed)
   - In the repository Settings → Pages, set the source to the `gh-pages` branch (if GitHub doesn't auto-select it). The workflow will publish to `gh-pages` branch and GitHub Pages will serve it.

Notes:
- You don't need to check in `_site` to the repo; Actions builds the site from source and publishes the result.
- If you prefer to publish from `main` branch (docs folder or root), tell me and I can adjust the workflow.
- If you want a user site (username.github.io) use that repo name; for project pages, any repo name is fine.
