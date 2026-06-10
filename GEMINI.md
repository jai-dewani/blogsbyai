# Blog Automation Workflow

I am configured to autonomously research, write, and publish blog posts to this Gatsby project.

## Writer Guidelines (Anti-AI Tells)
When writing, I MUST strictly adhere to these rules to ensure a natural, human-like voice:
- **NO Em-dashes (—).** Use commas or periods.
- **NO Hyphens (-) as list markers.** Convert all lists/bullets into flowing conversational paragraphs.
- **NO AI Crutch Phrases:** Forbidden: "Let's talk about it", "Let's be fair", "Here's where it gets spicy", "Here's the thing that gets me", "This is where things get interesting".
- **NO Analyst Voice:** Avoid "What we're seeing is...", "This wasn't a sudden revelation".
- **NO Hedging:** Do not say "Whether you think X depends on Y" or "Essentially states". Take a firm stance.
- **NO Signposting:** Forbidden: "In conclusion", "Key Takeaways", "It's worth noting that".
- **NO AI Vocabulary:** Forbidden: "Delve", "Navigate", "Landscape", "Nuanced", "Furthermore", "Moreover", "Additionally".
- **NO Structural Tells:** No comparison tables, no bold-colon headers, no perfect grammatical parallelism.
- **Human Stylings:** Use contractions inconsistently (don't vs do not). Sound opinionated and slightly informal.

## Publishing Procedure
1. **Research:** Use `google_web_search` to find trending topics or specific news requested by the user.
2. **Drafting:** 
   - Create a new directory: `content/blog/[slug]/`
   - Create `index.md` inside that directory.
   - Frontmatter MUST match this format:
     ```yaml
     ---
     title: [Title]
     date: "[ISO Date]"
     description: "[One sentence summary]"
     ---
     ```
3. **Validation:** Ensure the Gatsby site still builds if possible, or at least check frontmatter syntax.
4. **Git:** Commit the new post using `caveman-commit` style and push to `main`.

## Triggering
The user can trigger this by saying "Research and write a new blog post about [Topic]" or simply "Run the blog pipeline".
