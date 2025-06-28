# CLAUDE.md
必ず日本語で回答してください。
This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Zenn content repository containing Japanese technical articles and books. Zenn is a Japanese developer platform for sharing technical knowledge.

## Key Commands

### Content Management
- `npx zenn preview` - Preview content in browser (most frequently used)
- `npx zenn new:article` - Create new article
- `npx zenn new:book` - Create new book 
- `npx zenn list:articles` - List all articles
- `npx zenn list:books` - List all books

### Package Management
- `npm install` - Install dependencies (primarily zenn-cli)

## Content Structure

### Articles
- Located in `articles/` directory
- Each article is a Markdown file with Front Matter
- Front Matter includes: title, emoji, type (tech/idea), topics, published status
- Articles use unique hash-based filenames (e.g., `5330b19412687ee0b435.md`)

### Books
- Located in `books/` directory
- Structure for longer-form content

## Content Guidelines

### Front Matter Format
```yaml
---
title: "Article Title"
emoji: "📝"
type: "tech" # or "idea"
topics: [javascript, react, etc] # Maximum 5 topics allowed
published: true # or false
publication_name: "publication_name" # optional
---
```

### Content Types
- `tech`: Technical articles
- `idea`: Opinion/idea articles

## Development Workflow

### Content Development
1. Use `npx zenn preview` to start development server
2. Create new content with `npx zenn new:article` or `npx zenn new:book`
3. Edit Markdown files in `articles/` or `books/`
4. Preview changes in browser before publishing

### Git Workflow Rules - STRICTLY ENFORCE
**NEVER commit directly to main branch. Always use feature branches.**

#### Before ANY git commit:
1. **ALWAYS check current branch first**: `git branch --show-current`
2. **If on main branch**: STOP and switch to appropriate branch
3. **Branch selection strategy**:
   - Check existing branches: `git branch -a`
   - Use existing feature branch if relevant (e.g., `feature/claude-code-zenn-analysis-article`)
   - Create new feature branch if needed: `git checkout -b feature/description`

#### Required Git Commands Sequence:
```bash
# 1. Check current branch (MANDATORY)
git branch --show-current

# 2. If on main, switch to feature branch
git checkout feature/appropriate-branch-name
# OR create new branch
git checkout -b feature/new-feature-name

# 3. Then proceed with normal git workflow
git add .
git commit -m "message"
git push
```

#### Pull Request Workflow:
- All changes must go through Pull Requests
- No direct commits to main branch allowed
- Use `gh pr create` for PR creation
- Include appropriate PR descriptions and test plans

## Important Notes

- All content is in Japanese
- Articles must have proper Front Matter to be valid
- **Topics must be limited to maximum 5 items** - exceeding this limit causes errors
- **URLs should be placed on separate lines for card view display** - Zenn automatically generates preview cards for standalone URLs
- Published articles are automatically synced to Zenn platform
- No build process or testing framework - content is purely Markdown-based