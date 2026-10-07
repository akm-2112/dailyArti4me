# Evening Tech Discovery Archive

This repository contains the Evening Tech Discovery learning archive. The Markdown files are the source of truth for future GitHub sharing, website publishing, searching, and updates.

## Start here

- [Article index](index.md)
- [Volume 1](volumes/volume-1.md)
- [Volume 2](volumes/volume-2.md)
- [Volume 3](volumes/volume-3.md)
- [Volume 4](volumes/volume-4.md)
- [New article template](templates/article-template.md)

## Structure

```text
evening-tech-discovery/
├── README.md
├── index.md
├── articles/       # One Markdown file per article
├── volumes/        # Navigation grouped into six-entry volumes
├── templates/      # Reusable file template for future articles
└── assets/         # Images or other media added later
```

## Adding a future article

1. Copy `templates/article-template.md` into `articles/`.
2. Use the next three-digit number in the filename.
3. Add the title, order, volume, category, and source in the front matter.
4. Add the article link to `index.md` and the appropriate volume file.
5. Commit the change with a clear Git message.

Keep the article content in Markdown. Generate DOCX or PDF files later from this source when needed.
