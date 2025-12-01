# WFP Jordan Data Quality Automation - Documentation

This repository contains the documentation for the [WFP Jordan Data Quality Automation](https://github.com/WFP-VAM/JOR-data-quality-automation) application.

## 📖 Live Documentation

The documentation is published to GitHub Pages and available at:
**https://wfp-vam.github.io/JOR-data-quality-automation-docs/**

## 🏗️ Repository Structure

This repository contains:

- Documentation source files (`.md` and `.qmd` files)
- Quarto configuration (`_quarto.yml`)
- Custom styling (`styles.css`, `custom.scss`)
- GitHub Actions workflow for automatic deployment

## 🚀 Building Locally

### Prerequisites

Install [Quarto](https://quarto.org/docs/get-started/):

- Download from: https://quarto.org/docs/get-started/
- Or via Homebrew: `brew install quarto`

### Preview the Site

From the repository root:

```bash
quarto preview
```

This will:

- Build the site
- Start a local web server
- Open your browser
- Auto-reload when you edit files

### Build the Site

To build without serving:

```bash
quarto render
```

The rendered site will be in `_site/` (this directory is gitignored).

## 📝 Contributing

### Making Changes

1. Edit the markdown files in the repository
2. Preview locally with `quarto preview`
3. Commit and push your changes
4. The GitHub Actions workflow will automatically deploy to GitHub Pages

### Adding New Pages

1. Create a new `.md` or `.qmd` file in the appropriate directory
2. Add front matter (optional):
   ```yaml
   ---
   title: "Page Title"
   ---
   ```

3. Update `_quarto.yml` navigation to include the new page

## 🔄 Automatic Deployment

This repository uses GitHub Actions to automatically build and deploy the documentation to GitHub Pages whenever changes are pushed to the `main` or `master` branch.

The workflow:

1. Checks out the repository
2. Sets up Quarto
3. Renders the Quarto project
4. Deploys to GitHub Pages

See `.github/workflows/publish-docs.yml` for details.

## 📚 Documentation Structure

- **Getting Started**: Quick start guides and interactive discovery
- **Guides**: Comprehensive guides for adding exercises, metadata, and validation
- **Examples**: Real-world implementation examples
- **Development**: Internal development notes and architecture documentation

## 🔗 Related Repositories

- **Main Application**: [JOR-data-quality-automation](https://github.com/WFP-VAM/JOR-data-quality-automation)

## 📄 License

This documentation is part of the WFP Jordan Data Quality Automation project.

---

**Note**: This is a documentation-only repository. The main application code lives in the [JOR-data-quality-automation](https://github.com/WFP-VAM/JOR-data-quality-automation) repository.
