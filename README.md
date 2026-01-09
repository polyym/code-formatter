# Code Formatter

A fast, browser-based code formatter for JavaScript, JSX, TypeScript, and TSX. Built with SvelteKit and powered by Prettier.

## Features

- **Multi-language support**: JavaScript, JSX, TypeScript, TSX with auto-detection
- **Diff view**: Compare original and formatted code side-by-side with syntax highlighting
- **Configurable options**: Tab width, semicolons, quotes, trailing commas, bracket spacing, and more
- **File operations**: Upload files via drag-and-drop or file picker, download formatted output
- **Clipboard support**: Copy/paste with keyboard shortcuts
- **Keyboard shortcuts**:
  - `Ctrl+Enter` / `Cmd+Enter` - Format code
  - `Ctrl+S` / `Cmd+S` - Download formatted code
  - `Ctrl+K` / `Cmd+K` - Copy formatted code
- **Syntax highlighting**: Powered by Prism.js
- **Dark theme**: Cyberpunk-inspired UI with neon accents

## Tech Stack

- [SvelteKit](https://svelte.dev/) - Web framework
- [Svelte 5](https://svelte.dev/) - UI framework with runes
- [Prettier](https://prettier.io/) - Code formatter
- [Prism.js](https://prismjs.com/) - Syntax highlighting
- [Tailwind CSS](https://tailwindcss.com/) - Styling
- [diff](https://www.npmjs.com/package/diff) - Diff generation

## Getting Started

### Prerequisites

- Node.js 22+
- npm

### Installation

```sh
npm install
```

### Development

Start the development server:

```sh
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser.

### Building

Create a production build:

```sh
npm run build
```

Preview the production build:

```sh
npm run preview
```

### Type Checking

```sh
npm run check
```

## Deployment

This project is configured for static site deployment with `@sveltejs/adapter-static`. The `netlify.toml` file includes configuration for deploying to Netlify.

To deploy:

1. Push your code to a Git repository
2. Connect the repository to Netlify
3. Netlify will automatically build and deploy using the provided configuration

## Project Structure

```
src/
  app.css          # Global styles (Tailwind + custom CSS)
  app.html         # HTML template with meta tags
  routes/
    +layout.svelte # Root layout
    +layout.ts     # Prerender configuration
    +page.svelte   # Main formatter component
static/
  favicon.svg      # App favicon
  robots.txt       # SEO robots file
  sitemap.xml      # SEO sitemap
```

## License

MIT

## Version

1.0.0
