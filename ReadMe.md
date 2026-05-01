# Paul's Blog

Welcome to Paul's Blog, a personal space for sharing random programming thoughts, book summaries, and software development insights. This blog is built using Hugo, a fast and flexible static site generator, and uses the `hugo-simple-beauty` theme.

## Table of Contents
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
- [Running Locally](#running-locally)
- [Building the Site](#building-the-site)
- [Content Creation](#content-creation)
- [Theme](#theme)
- [Deployment](#deployment)
- [SEO Enhancements](#seo-enhancements)
- [Upgrading Hugo](#upgrading-hugo)

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.

### Prerequisites

Before you begin, ensure you have the following installed:

*   **Hugo**: The static site generator. Follow the official Hugo installation guide for your operating system.
    *   [Install Hugo](https://gohugo.io/getting-started/installing/)
*   **Git**: For cloning the repository and managing the theme submodule.

### Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/prule/pauls-blog.git
    cd pauls-blog
    ```

2.  **Initialize and update the theme submodule:**
    ```bash
    git submodule update --init --recursive
    ```
    (If you are setting up a new Hugo site from scratch, you would use `git submodule add https://github.com/prule/hugo-simple-beauty.git themes/hugo-simple-beauty` as shown in the original README.)

## Running Locally

To run the blog locally and see your changes in real-time, navigate to the root of the project and execute:

```bash
hugo server
```

This will start a local development server, usually accessible at `http://localhost:1313`. Hugo will automatically rebuild the site and refresh your browser when changes are detected.

If you encounter issues with fast rendering (e.g., changes not appearing), you can disable it:

```bash
hugo server --disableFastRender
```

## Building the Site

To generate the static files for your blog, run the following command from the project root:

```bash
hugo
```

This will create a `public/` directory containing all the static HTML, CSS, JavaScript, and other assets ready for deployment.

## Content Creation

New posts can be created using Hugo's command-line tool. For example, to create a new post:

```bash
hugo new content/posts/my-new-post.md
```

Edit the generated Markdown file in the `content/` directory. Remember to fill out the front matter (the section between `---` at the top of the file) with relevant information like `title`, `date`, `tags`, `categories`, and `description`.

## Theme

This blog uses the `hugo-simple-beauty` theme, included as a Git submodule. You can find more information about the theme and its customization options in its repository:

*   [hugo-simple-beauty GitHub Repository](https://github.com/prule/hugo-simple-beauty)

## Deployment

This blog is configured for deployment to GitHub Pages, as indicated by the `baseURL` in `hugo.toml`. After building the site (using `hugo`), the contents of the `public/` directory can be pushed to your GitHub Pages branch (e.g., `gh-pages` or `main` if using a `docs` folder).

## SEO Enhancements

Recent updates have been made to improve the blog's Search Engine Optimization (SEO) and social sharing capabilities:

*   **Meta Descriptions**: Dynamic meta descriptions are now generated for each page, pulling from the page's `description` front matter or the site's global description.
*   **Open Graph Tags**: Added Open Graph (`og:`) tags for better integration with social media platforms like Facebook and LinkedIn, ensuring rich previews when links are shared.
*   **Twitter Card Tags**: Implemented Twitter Card (`twitter:`) tags to optimize how content appears on Twitter.
*   **`robots.txt`**: A `robots.txt` file has been added to guide search engine crawlers and specify the sitemap location.

## Upgrading Hugo

To upgrade your Hugo installation, use your package manager:

**macOS (Homebrew):**
```bash
brew update
brew upgrade hugo
```

For manual installations, download the latest binary from the [Hugo Releases page](https://github.com/gohugoio/hugo/releases) and replace your existing Hugo executable.
