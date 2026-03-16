# Recipe Popularity Jekyll Site

This folder contains a ready-to-publish GitHub Pages / Jekyll site for the recipe popularity project.

## Files

- `index.md` - main project page
- `_config.yml` - Jekyll configuration
- `assets/css/style.scss` - small style overrides
- `assets/plots/` - embedded Plotly HTML files extracted from the notebook outputs

## Publish on GitHub Pages

1. Create a new GitHub repository.
2. Upload all files in this folder to the repository root.
3. In GitHub, open **Settings > Pages**.
4. Under **Build and deployment**, choose **Deploy from a branch**.
5. Select the `main` branch and the `/ (root)` folder.
6. Save. GitHub Pages will build the Jekyll site automatically.

## Important note

The current notebook does **not** yet satisfy the missingness dependency requirement, because both reported p-values are non-significant. Update that section in the notebook and replace the website text if you find a column with significant missingness dependence.
