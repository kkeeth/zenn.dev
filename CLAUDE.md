# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Zenn.dev content repository containing technical articles and books written in Markdown. Zenn is a Japanese technical publishing platform similar to Medium or Dev.to.

## Repository Structure

- `articles/` - Individual technical articles in Markdown format
- `books/` - Multi-chapter technical books, each in its own directory
  - Each book has a `config.yaml` file defining metadata, chapters, and publishing settings
  - Chapters are individual Markdown files referenced in the config
- `images/` - Static assets for articles and books, organized by content type
- `package.json` - Defines Zenn CLI dependencies and scripts

## Common Commands

```bash
# Initialize Zenn project (if needed)
pnpm run init

# Preview content locally with live reload
pnpm run preview

# Create a new article with slug
pnpm run article <slug-name>

# Create a new book
pnpm run book
```

## Content Architecture

### Articles
- Standalone Markdown files in `articles/` directory
- Include frontmatter with metadata (title, topics, published status, etc.)

### Books
- Multi-chapter publications in `books/<book-name>/` directories
- `config.yaml` defines book metadata:
  - `title`: Book title
  - `summary`: Book description
  - `topics`: Array of relevant tags
  - `published`: Boolean publication status
  - `price`: Pricing (0 for free)
  - `chapters`: Ordered array of chapter filenames (without .md extension)
- Each chapter is a separate Markdown file

### Current Books
1. **riotjs-hello-localstack** - AWS learning with LocalStack (unpublished)
2. **riotjs-tour-of-heroes** - Riot.js tutorial series
3. **tetris-with-js** - JavaScript Tetris game tutorial (published, free)

## Package Manager

This project uses `pnpm` as specified in the `packageManager` field. Always use `pnpm` instead of `npm` or `yarn`.

## Development Workflow

1. Use `pnpm run preview` to start local development server
2. Edit Markdown files directly
3. For books, ensure `config.yaml` chapters array matches actual chapter files
4. Images should be placed in `images/` with appropriate subdirectory structure

## Content Guidelines

- Articles and books focus on JavaScript, Riot.js, AWS, and web development topics
- Content is written in Japanese
- Books use structured chapter approach with clear learning progression
- Free content (price: 0) is preferred for educational materials