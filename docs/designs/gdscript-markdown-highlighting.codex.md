---
tags:
  - vscode
  - textmate
  - markdown
  - gdscript
  - godot-vscode-plugin
status: implemented
created: 2026-05-24
---

# Design: GDScript Highlighting in Markdown Code Fences

## Context

The extension already registers `gdscript` as a VS Code language and contributes `source.gdscript` from `syntaxes/GDScript.tmLanguage.json`. That covers standalone `.gd` files, but Markdown files need a separate TextMate injection so fenced code blocks can delegate their contents to the GDScript grammar.

The missing user-visible case is:

~~~markdown
```gdscript
extends Node

func _ready() -> void:
	print("Hello")
```
~~~

In VS Code, this is handled at the grammar layer rather than by the GDScript language server. The implementation should therefore be limited to extension manifest and syntax grammar changes.

## Goals

- Highlight fenced GDScript code blocks inside Markdown source editors.
- Reuse the existing `source.gdscript` grammar instead of duplicating syntax rules.
- Preserve normal Markdown highlighting and existing fenced code block behavior for other languages.
- Make the embedded block identify as the `gdscript` language so basic editor behavior, such as bracket matching and snippets, can work inside the fenced block when VS Code supports it.
- Keep the change activation-free; the grammar should work without starting the extension host code path.

## Non-goals

- Do not implement semantic tokens, language server features, formatting, or diagnostics inside Markdown code fences.
- Do not change the standalone GDScript grammar except where a bug is independently discovered.
- Do not build a custom Markdown preview renderer.
- Do not try to affect GitHub, npm, or other external Markdown renderers.

## Proposed Design

Add a new TextMate injection grammar, for example `syntaxes/GDScriptMarkdown.tmLanguage.json`, that injects into VS Code's Markdown TextMate scope `text.html.markdown`.

The injection grammar should:

- Detect fenced code blocks whose info string is `gdscript`.
- Also support `gd` as a short alias, because users often associate GDScript files with the `.gd` extension.
- Match both backtick and tilde fences.
- Allow optional trailing fence attributes after whitespace, such as ` ```gdscript title="player.gd" `, without treating `gdscriptx` as GDScript.
- Wrap the code body in a `meta.embedded.block.gdscript` scope.
- Include `source.gdscript` inside that embedded body.

Then update `package.json` under `contributes.grammars` with a second grammar contribution:

```json
{
	"scopeName": "markdown.gdscript.codeblock",
	"path": "./syntaxes/GDScriptMarkdown.tmLanguage.json",
	"injectTo": ["text.html.markdown"],
	"embeddedLanguages": {
		"meta.embedded.block.gdscript": "gdscript"
	}
}
```

This should not add a fake language id. VS Code's current extension guidance allows injection grammars to be contributed with `injectTo` and no `language` entry.

## Grammar Shape

The grammar can follow the same structure used by common fenced-code injection examples:

```json
{
	"scopeName": "markdown.gdscript.codeblock",
	"injectionSelector": "L:text.html.markdown",
	"patterns": [
		{
			"include": "#gdscript-code-block"
		}
	],
	"repository": {
		"gdscript-code-block": {
			"begin": "(^|\\G)(\\s*)(`{3,}|~{3,})\\s*(?i:(gdscript|gd)(\\s+[^`~]*)?$)",
			"name": "markup.fenced_code.block.markdown",
			"end": "(^|\\G)(\\2|\\s{0,3})(\\3)\\s*$",
			"beginCaptures": {
				"3": {
					"name": "punctuation.definition.markdown"
				},
				"4": {
					"name": "fenced_code.block.language.markdown"
				},
				"5": {
					"name": "fenced_code.block.language.attributes.markdown"
				}
			},
			"endCaptures": {
				"3": {
					"name": "punctuation.definition.markdown"
				}
			},
			"patterns": [
				{
					"begin": "(^|\\G)(\\s*)(.*)",
					"while": "(^|\\G)(?!\\s*([`~]{3,})\\s*$)",
					"contentName": "meta.embedded.block.gdscript",
					"patterns": [
						{
							"include": "source.gdscript"
						}
					]
				}
			]
		}
	}
}
```

Before implementation, confirm the exact regex behavior in VS Code with the token inspector. If nested Markdown cases are important, also test fenced blocks inside lists and block quotes; those can require a broader begin pattern than the simple top-level example.

## Markdown Preview and Hover Behavior

This design primarily targets Markdown source editor highlighting. VS Code Markdown preview and `MarkdownString` rendering are separate surfaces from editor tokenization.

After adding the injection grammar, manually verify these surfaces:

- A `.md` source editor with ` ```gdscript ` and ` ```gd ` fences.
- Markdown preview for the same file.
- Existing extension hovers that call `contents.appendCodeblock(text, "gdscript")`.

If preview or hover code blocks still do not highlight, treat that as a separate follow-up. The likely constraint is that those renderers may use VS Code's built-in Markdown rendering path rather than extension-provided TextMate injections. The fallback should not be a custom renderer unless there is a clear product need.

## Verification Plan

1. Add a small fixture such as `syntaxes/examples/gdscript-markdown.md` or use an unsaved local Markdown file during manual testing.
2. Launch the Extension Development Host from this extension.
3. Open a Markdown file containing both `gdscript` and `gd` fences.
4. Run `Developer: Inspect Editor Tokens and Scopes` inside the fenced block.
5. Confirm tokens include `meta.embedded.block.gdscript`, `source.gdscript`, and GDScript-specific scopes such as `keyword.language.gdscript` or `keyword.control.gdscript`.
6. Confirm the inspector reports the embedded language as `gdscript`.
7. Confirm JavaScript, JSON, and plain Markdown fences are unchanged.
8. Run `npm run compile`; no TypeScript changes are expected, but this catches accidental manifest or build regressions.
9. Run `npm run package` if validating the VSIX payload is needed. The existing `.vscodeignore` already includes `!syntaxes/*.tmLanguage.json`, so the new grammar file should be packaged.

## Rollout Plan

1. Add `syntaxes/GDScriptMarkdown.tmLanguage.json`.
2. Register the injection grammar in `package.json`.
3. Optionally add `gd` to the `gdscript` language aliases only if Markdown preview or another VS Code surface needs the alias. The source editor injection does not require it.
4. Manually verify the editor behavior in the Extension Development Host.
5. Add a changelog entry once the implementation lands.

## Risks

- Markdown grammar injection regexes are easy to make too broad. The matcher must not hijack `gdscriptx`, `gdshader`, or unrelated fenced blocks.
- GDScript grammar bugs will become more visible in README-style documentation because code snippets are often partial and not full scripts.
- Markdown preview and hover highlighting may not be fixed by editor TextMate injection alone.
- Nested fences inside block quotes or lists may need extra testing before claiming full Markdown coverage.

## Implementation Notes

- 2026-05-24: Added `syntaxes/GDScriptMarkdown.tmLanguage.json` as a Markdown injection grammar.
- 2026-05-24: Registered the injection in `package.json` with `embeddedLanguages` mapping `meta.embedded.block.gdscript` to `gdscript`.
- 2026-05-24: Added `syntaxes/examples/gdscript-markdown.md` with `gdscript`, `gd`, and non-GDScript fence examples for manual token inspection.
- 2026-05-24: Left `.vscodeignore` unchanged because it already packages `syntaxes/*.tmLanguage.json`.

## Resolved Decisions

- Support both `gdscript` and `gd` fence identifiers.
- Treat Markdown source editor highlighting as the implemented scope. Markdown preview and hover rendering remain follow-up surfaces if they do not pick up the injection.
- Keep a fixture at `syntaxes/examples/gdscript-markdown.md` for manual token inspection.

## References

- VS Code Syntax Highlight Guide: https://code.visualstudio.com/api/language-extensions/syntax-highlight-guide
- VS Code fenced code block grammar injection example: https://github.com/mjbvz/vscode-fenced-code-block-grammar-injection-example

## Document Changelog

- 2026-05-24: Initial design for adding GDScript fenced code block highlighting in Markdown.
- 2026-05-24: Implemented the grammar injection, package contribution, fixture, and changelog entry.
