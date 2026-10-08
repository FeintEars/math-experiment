# Shiki in Next.js

## Meta

| Field | Value |
|---|---|
| ID | 9150f7f0-de91-46b8-a2a9-0b26c8a48f8a |
| Title | Shiki in Next.js |
| Slug | shiki-syntax-highlighting-nextjs-dark-light-theme |
| Status | published |
| Published at | 2026-10-09T19:52:47.000Z |
| Content updated at | 2026-10-08T20:51:08.450Z |
| Featured | false |
| Category | Frontend Web Development (`frontend-web-development`) |
| Authors | Yuliia Svitlychna (`yuliia-svitlychna`), Frontend Engineer (React / NextJS) |
| Created at | 2026-10-03T23:48:43.683Z |
| Updated at | 2026-10-08T21:03:46.103Z |

## SEO

| Field | Value |
|---|---|
| SEO title | Shiki in Next.js: dark and light code themes |
| SEO description | How we added code blocks to our Next.js and Payload CMS blog: Shiki on the server, one pass for dark and light themes, a theme built from our design tokens. |
| OG image | — |
| No index | false |

## Teaser

Our blog is about programming, and our posts had no proper way to show code. Here is how we built a code block with Shiki that renders on the server, supports dark and light themes in one pass, and uses colors from our own design system.

## Body

ProdStories is a blog about programming, so most of our posts need code. When we started, the post editor had no code block. Of course, we did not want to put code into articles as plain text, mixed in with the paragraphs. Code like that is hard to read and easy to break.

This post is about the code block we built to fix that, and about what we learned along the way. The short version: we highlight code with [Shiki](https://shiki.style) inside a Next.js Server Component, so the browser gets finished HTML and none of the highlighter's code. One render produces colors for both our dark and light themes, and the colors come from our own design tokens.

**TL;DR.** Run Shiki in a Server Component, so the highlighter never reaches the browser. Create the highlighter once and share it between all blocks. Use `defaultColor: false` to get both themes as CSS variables, and switch them with one CSS rule. Build the theme from your own palette. Keep the Copy button as the only client component in the block.

## Problem 1: our posts had no code blocks

Our posts live in Payload CMS, and its rich text editor (Lexical) supports inline code inside a paragraph, but not a block of code. A 10-line function pasted as a regular paragraph would lose its monospace layout and indentation, and it would have no file name and no way to copy it.

For a programming blog this is a basic feature, so we added our own block to the editor. In Payload, a block is a small form that an author inserts between paragraphs:

**File:** `code-block.ts`

```typescript
export const CodeBlock: Block = {
  slug: 'codeBlock',
  interfaceName: 'CodeBlock',
  fields: [
    {
      type: 'row',
      fields: [
        { name: 'filename', type: 'text', label: 'File name' },
        { name: 'note', type: 'text', label: 'Note' },
      ],
    },
    {
      name: 'syntax',
      type: 'select',
      required: true,
      defaultValue: DEFAULT_CODE_SYNTAX,
      options: [...CODE_SYNTAXES],
    },
    { name: 'code', type: 'code', required: true, label: 'Code' },
  ],
};
```

On the site, Payload's `RichText` component needs to know how to render this block. We pass it a converter that maps `codeBlock` to our React component:

**File:** `rich-text-converters.tsx`

```tsx
blocks: {
  codeBlock: ({ node }) => (
    <CodeBlock
      filename={node.fields.filename}
      note={node.fields.note}
      syntax={node.fields.syntax}
      code={node.fields.code}
    />
  ),
},
```

**Result.** An author picks "Code" in the editor, pastes the code, and optionally adds a file name and a short note. The reader sees a separate block with a header, the code in a monospace font, and a Copy button.

## Problem 2: the block was black and white

The first version of the block worked, but every character had the same color. Code is much easier to read when keywords, strings, function names and comments look different, the way they do in an IDE. Our eyes find the structure of a function by color before they read a single word.

We could have done this with our own CSS: wrap each token in a `<span>` with a class like `.keyword` or `.string` and style the classes. The hard part is producing those spans. You need a tokenizer that understands TypeScript, SCSS, SQL, Bash and every other language we write about. Writing and maintaining that by hand is a lot of work that has nothing to do with our blog.

So we looked for a library, and the first one we tried was [Shiki](https://shiki.style). It worked on the first try. Shiki uses the same TextMate grammars and theme format as VS Code, so code on the page is tokenized the way your editor tokenizes it.

There is one catch. Shiki is heavy: it ships a WebAssembly regex engine and a grammar file per language. We did not want readers to download it, so it runs only on the server. Our `CodeBlock` is an async Server Component that highlights the code and sends finished HTML:

**File:** `code-block.tsx`

```tsx
export async function CodeBlock({ filename, note, syntax, code }: PropsType) {
  const hasHeader = Boolean(filename || note);
  const html = await highlight(code, syntax);

  return (
    <figure className={styles.block}>
      {hasHeader && (
        <figcaption className={styles.header}>
          <span className={styles.filename}>{filename}</span>
          <span className={styles.meta}>
            {note && <span className={styles.note}>{note}</span>}
            <CopyButton code={code} />
          </span>
        </figcaption>
      )}

      <div className={styles.pre} dangerouslySetInnerHTML={{ __html: html }} />

      {!hasHeader && (
        <span className={styles.floatingCopy}>
          <CopyButton code={code} />
        </span>
      )}
    </figure>
  );
}
```

To see what this saves, we bundled the same setup for the browser with esbuild: Shiki core, the WebAssembly engine, our 22 languages and our 2 themes.

|  | Raw | Gzip |
|---|---|---|
| Shiki in the browser (our minimal set) | 2.96 MB | about 501 KB |
| of which the WebAssembly engine | 622 KB | 230 KB |
| Shiki in our setup | 0 | 0 |

In our local build output, Shiki's code shows up only in the server chunks.

**Result.** Code in posts is highlighted like in an editor, and the reader's browser does no highlighting work at all.

[IMAGE: shiki-before-after.png]

![shiki-before-after.png](https://nextjs-blog-mocha-gamma-61.vercel.app/api/media/file/shiki-before-after-KMVaZ5l2MXcOQAA4TzOeqTOdZnZeXI.png?prefix=media)

<!-- media id: cd038069-465b-4951-aa13-bdac94bd4a24 | file: shiki-before-after-KMVaZ5l2MXcOQAA4TzOeqTOdZnZeXI.png | 1600x727 -->

*Left: the same block with highlighting turned off ("Plain text"), which is how our first version looked. Right: the same code highlighted by Shiki.*

## Create the highlighter once

Creating a Shiki highlighter means loading the WebAssembly engine and compiling the grammars. On our machine the first call took 314 ms, and the next calls took about 48 ms. A post with five code blocks should not pay the first price five times.

So the highlighter lives at module level, and we cache the promise:

**File:** `highlight.ts`

```typescript
let pending: Promise<Highlighter> | null = null;

function loadHighlighter() {
  pending ??= createHighlighter({
    themes: [CODE_THEME_DARK, CODE_THEME_LIGHT],
    langs: CODE_SYNTAXES.map((entry) => entry.value).filter((value) => value !== 'text') as BundledLanguage[],
  });

  return pending;
}
```

We cache the promise and not the finished highlighter on purpose. The blocks on a page render in parallel. If we stored the highlighter only after it was ready, every block that started before that moment would see an empty cache and create its own. With the promise, they all wait for the same one.

## What we learned: Shiki renders both themes in one pass

Our site has a dark and a light theme, and the reader can switch between them. So the code needs two sets of colors. Without help from the library, that means highlighting the code twice and hiding one copy, or highlighting it again on every theme switch.

While setting up Shiki we found that it already solves this. The feature is called [dual themes](https://shiki.style/guide/dual-themes): you pass two themes at once, and Shiki tokenizes the code a single time. For every token it writes the color from both themes. With `defaultColor: false` it does not pick a winner. It puts both colors into CSS variables:

**File:** `highlight.ts`

```typescript
return highlighter.codeToHtml(code, {
  lang,
  themes: { light: CODE_THEME_LIGHT.name!, dark: CODE_THEME_DARK.name! },
  defaultColor: false,
});
```

This is what the HTML looks like for `const a = 'x';`:

**Note:** Shiki output

```html
<pre class="shiki shiki-themes signal-light signal-dark" style="--shiki-light:#1f2328;--shiki-dark:#dfe2eb;--shiki-light-bg:#ffffff;--shiki-dark-bg:#0d1117" tabindex="0">
  <code>
    <span class="line">
      <span style="--shiki-light:#1A7F37;--shiki-dark:#3FB950">const</span>
      <span style="--shiki-light:#1F2328;--shiki-dark:#DFE2EB"> a = </span>
      <span style="--shiki-light:#9A6700;--shiki-dark:#E3B341">'x'</span>
      ...
```

Every token carries both of its colors. Choosing one is now a job for CSS, and it takes two rules:

**File:** `code-block.module.scss`

```scss
.block .pre :global(pre.shiki) span {
  color: var(--shiki-dark);
}

:global([data-theme='light']) {
  .block .pre :global(pre.shiki),
  .block .pre :global(pre.shiki span) {
    color: var(--shiki-light);
  }
}
```

The `data-theme` attribute on `<html>` is the same one the rest of the site uses for its theme. The server sets it from a cookie before it sends the page, so the first HTML already has the right theme.

Here is what this gave us:

- **Instant theme switch.** Clicking the toggle changes one attribute on `<html>`. The code is not highlighted again, and nothing re-renders.
- **No flash of the wrong theme.** The server knows the theme from the cookie, so the code never shows up dark and then turns light.
- **One copy of the HTML.** The markup is the same for both themes, so there is nothing to hide or swap, and no hydration mismatch.

For us this was the most useful find in the whole task. Our problem was "the code is black and white", and on the way to fixing it we got dark and light support for free, with no client code.

![shiki-dark-light.png](https://nextjs-blog-mocha-gamma-61.vercel.app/api/media/file/shiki-dark-light-6FymIVTd18Hho4FW5oyAqgO2xDyJAl.png?prefix=media)

<!-- media id: d417892e-020a-4c87-90d8-ff95a04338ea | file: shiki-dark-light-6FymIVTd18Hho4FW5oyAqgO2xDyJAl.png | 1600x727 -->

*The same HTML in our dark and light theme. Only the CSS variable that **`span`** reads from changes.*

## Our own theme, built from our design tokens

Shiki ships with dozens of themes, including GitHub's. We did not use them. Our site has its own design system with its own color tokens: `--accent` green, `--info` blue, `--warning` yellow and so on, in a dark and a light version. A code block in GitHub's colors would look like a piece of another website pasted into ours.

So we built the theme ourselves from our tokens. Shiki themes are plain objects in the VS Code theme format ([theme colors](https://shiki.style/guide/theme-colors) in the docs). We describe each theme as a small palette of roles:

**File:** `code-theme.ts`

```typescript
const DARK: PaletteType = {
  fg: '#dfe2eb',
  bg: '#0d1117',
  keyword: '#3fb950',
  fn: '#58a6ff',
  type: '#f85149',
  string: '#e3b341',
  property: '#bdcab8',
  comment: '#8b949e',
};

const LIGHT: PaletteType = {
  fg: '#1f2328',
  bg: '#ffffff',
  keyword: '#1a7f37',
  fn: '#0969da',
  type: '#cf222e',
  string: '#9a6700',
  property: '#3f4650',
  comment: '#6e7781',
};
```

These are the same values as our design tokens: `keyword` is our `--accent`, `fn` is `--info`, `type` is `--danger`, `string` is `--warning`. Then one table maps TextMate scopes to these roles:

**File:** `code-theme.ts`

```typescript
const SCOPES: [keyof PaletteType, string[]][] = [
  ['comment', ['comment', 'punctuation.definition.comment']],
  ['keyword', ['keyword', 'storage', 'storage.type', 'storage.modifier', ...]],
  ['fn', ['entity.name.function', 'support.function', 'meta.function-call', ...]],
  ['type', ['entity.name.type', 'entity.name.class', 'support.type', ...]],
  ['string', ['string', 'constant.numeric', ...]],
  ['property', ['variable.other.property', 'meta.object-literal.key', ...]],
  ['fg', ['keyword.operator', 'punctuation', 'variable', ...]],
];

function build(name: string, type: 'dark' | 'light', palette: PaletteType): ThemeRegistration {
  return {
    name,
    type,
    fg: palette.fg,
    bg: palette.bg,
    settings: SCOPES.map(([key, scope]) => ({ scope, settings: { foreground: palette[key] } })),
  };
}

export const CODE_THEME_DARK = build('signal-dark', 'dark', DARK);
export const CODE_THEME_LIGHT = build('signal-light', 'light', LIGHT);
```

Both themes come out of the same `SCOPES` table, so they cannot drift apart. If we decide that decorators should be blue, we add one scope to one line, and it changes in both themes.

**Result.** Code blocks in our posts use the colors of the rest of the site. The typing animation on our homepage uses the same roles (`--accent` for keywords, `--info` for functions), so code looks the same everywhere on ProdStories.

## Highlighting needs a language

Shiki cannot guess the language. It needs to be told whether a block is TypeScript, SCSS or Bash, because each language has its own grammar. That leaves two questions: how does the author tell it, and which grammars does the server load?

If the language were a free text field, an author could type `ts`, `TypeScript` or `typscript`. Some of those would fail and some would quietly fall back to plain text. And if the server loaded every grammar Shiki has (more than 200), it would spend time and memory on languages we never use.

We answered both questions with one list:

**File:** `code-syntaxes.ts`

```typescript
export const CODE_SYNTAXES = [
  { label: 'TypeScript', value: 'typescript' },
  { label: 'TSX', value: 'tsx' },
  { label: 'SCSS', value: 'scss' },
  { label: 'Shell', value: 'bash' },
  // ...23 entries: 22 languages plus plain text
  { label: 'Plain text', value: 'text' },
] as const;

export const DEFAULT_CODE_SYNTAX = 'typescript';
```

The same array is used twice:

1. As the `options` of the `syntax` select in the Payload block. The author picks a language from a dropdown and cannot misspell it.
2. As the `langs` of the highlighter. The server loads exactly these 22 grammars.

And if a value we do not know still reaches the highlighter, it falls back to the default instead of failing:

**File:** `highlight.ts`

```typescript
const SUPPORTED = new Set<string>(CODE_SYNTAXES.map((entry) => entry.value));

export async function highlight(code: string, syntax?: string | null) {
  const lang = syntax && SUPPORTED.has(syntax) ? syntax : DEFAULT_CODE_SYNTAX;
  // ...
}
```

**Result.** Every code block has a valid language, the editor and the highlighter can never disagree about the list, and adding a language is one line.

## Keep the Copy button the only client component

The Copy button needs the browser: it calls `navigator.clipboard` and keeps a "copied" state for two seconds. So it has to be a client component. The rest of the block does not.

This split matters more than it looks. If we had put `'use client'` on the whole `CodeBlock`, the `highlight()` call and Shiki with it would move into the browser bundle. That is the 501 KB from the table above, downloaded by every reader of every post with code. Keeping the client part as small as possible is what keeps Shiki on the server.

So only the button is a client component:

**File:** `copy-button.tsx`

```tsx
'use client';

const RESET_MS = 2000;

export function CopyButton({ code }: PropsType) {
  const [isCopied, setIsCopied] = useState(false);

  useEffect(() => {
    if (!isCopied) return;

    const timer = setTimeout(() => setIsCopied(false), RESET_MS);

    return () => clearTimeout(timer);
  }, [isCopied]);

  const copy = async () => {
    try {
      await navigator.clipboard.writeText(code);
      setIsCopied(true);
    } catch {
      setIsCopied(false);
    }
  };

  return (
    <button type="button" className={styles.copy} onClick={copy} aria-label={isCopied ? 'Code copied' : 'Copy code'}>
      {isCopied ? <Check size={14} aria-hidden="true" /> : <Copy size={14} aria-hidden="true" />}
    </button>
  );
}
```

The server component renders the button and passes it the raw code as a prop. React sends the button's small bundle and the code string, and hydrates only the button. The `<figure>`, the header and the highlighted code stay plain server HTML.

A few details that help readers: the icon turns into a check mark for two seconds, the `aria-label` changes to "Code copied" for screen readers, and blocks without a header get a floating button in the top right corner, so every block can be copied.

## What we would still improve

We are sharing this as it works on our site today, and a few things could be better:

- **Silent copy errors.** If the browser blocks the clipboard, the button does nothing and shows nothing. A short "Copy failed" state would be more honest to the reader.
- **The fallback language.** An unknown language falls back to TypeScript. For code that is not TypeScript, `text` would be a safer default.
- **Cold starts.** The 314 ms setup happens once per server instance. On a serverless host a new instance pays it again. Caching rendered pages hides most of this, but it is not free.

## How we built it

We wrote most of this code together with Claude Code, and it worked well for this task. It knew the Shiki and Payload APIs, and it got us from a black and white block to working highlighting quickly.

## Checklist for your own code blocks

1. Highlight in a Server Component, so the highlighter never reaches the browser.
2. Create the highlighter once per process and cache the promise.
3. Pass both themes to Shiki with `defaultColor: false`, and switch them with CSS variables.
4. Build the theme from your own design tokens, with one scope table for both themes.
5. Keep one list of languages for the editor and the highlighter.
6. Make only the Copy button a client component.

The code is on GitHub: [components/blocks/code-block](https://github.com/captain-solver-news/prodstories-blog/tree/8a6f2b5098814956042df6bec8b6939c3da53841/components/blocks/code-block), and the [commit that added it](https://github.com/captain-solver-news/prodstories-blog/commit/8a6f2b5098814956042df6bec8b6939c3da53841).
