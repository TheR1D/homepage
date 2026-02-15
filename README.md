# Homepage

Personal website built with Hugo and deployed to GitHub Pages.

## 🚀 GitHub Pages Setup Instructions

To enable automatic deployment to GitHub Pages after pushing this workflow:

### 1. Enable GitHub Pages in Repository Settings

1. Go to your repository on GitHub: `https://github.com/TheR1D/homepage`
2. Click on **Settings** tab
3. In the left sidebar, click on **Pages**
4. Under **Build and deployment**:
   - **Source**: Select "GitHub Actions"
   - (Do NOT select "Deploy from a branch")

### 2. Push to Main Branch

Once the above is configured, any push to the `main` branch will automatically:
1. Build your Hugo site
2. Deploy it to GitHub Pages

Your site will be available at: `https://sadykov.dev/` (or `https://ther1d.github.io/homepage/` if custom domain is not configured)

### 3. Custom Domain (Optional)

If you want to use your custom domain `sadykov.dev`:

1. In the **Pages** settings (same location as step 1):
   - Enter your custom domain in the **Custom domain** field
   - Click **Save**
2. Configure DNS records with your domain provider:
   - Add a CNAME record pointing to `ther1d.github.io`
   - Or add A records pointing to GitHub Pages IPs

## 🔧 Local Development

To run the site locally:

```bash
# Install Hugo (macOS)
brew install hugo

# Or download from https://github.com/gohugoio/hugo/releases

# Run development server
hugo server -D

# Build site
hugo
```

## 📝 Workflow Details

The GitHub Actions workflow (`.github/workflows/hugo.yml`):
- Triggers on any push to the `main` branch
- Can also be manually triggered from the Actions tab
- Uses the official Hugo Setup action (`peaceiris/actions-hugo@v3`) with latest Hugo version
- Builds the site with `hugo --minify`
- Deploys to GitHub Pages using `actions/deploy-pages@v4`

## 🎨 Theme

This site uses the [Mana theme](https://github.com/victoriadrake/hugo-theme-mana).
