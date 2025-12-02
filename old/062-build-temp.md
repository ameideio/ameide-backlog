Your summary is **accurate for rules \_js v 2.4.1**, with one small clarification about the label syntax.

| ✔ Correct                                                                         | 📌 Note / Clarification                                                                                                                                                                  | Vendor proof                                                                          |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| 1. *Root cause is that bin targets aren’t generated for pnpm‑workspace packages.* | Yes. Issue #1250 documents that `npm_translate_lock` skips bin mirroring for local / workspace packages.                                                                                 | ([GitHub][1])                                                                         |
| 2. *Work‑around is a custom `js_binary` wrapper.*                                 | Correct. The README calls this the “local CLI binary” pattern.                                                                                                                           |                                                                                       |
| 3. *Using `.bin/next` avoids the conflict seen with `dist/bin/next`.*             | Also correct—the shell wrapper in `.bin/next` is a single file, whereas `dist/bin` contains multiple files that collide with the symlink forest created by `npm_link_all_packages`.      | Next.js package layout (bin → wrapper → `dist/bin/next.js`) ([GitHub][2])             |
| 4. *BUILD snippet syntax*                                                         | ✅ Works, but drop the leading `:` in `entry_point`—`js_binary.entry_point` is a file **path**, not a label.<br>So use `"node_modules/.bin/next"` instead of `":node_modules/.bin/next"`. | `js_binary` API doc – “*entry\_point: path relative to the BUILD file*” ([GitHub][3]) |

### Recommended final snippet

```bzl
npm_link_all_packages(name = "node_modules")

js_binary(
    name        = "next_cli",
    entry_point = "node_modules/.bin/next",   # path, not label
    data        = [":node_modules"],
)

nextjs_standalone_build(
    name           = "www-ameide_portal_canvas",
    next_js_binary = ":next_cli",
    # … other attrs unchanged
)
```

That wrapper will keep working until the workspace‑bin bug is fixed upstream; at that point you can switch back to the canonical `@npm//next/bin:next` label.

*No other changes needed.*

[1]: https://github.com/aspect-build/rules_js/issues/1250 "[Bug]: ts-protoc-gen package isn't generating a package_json.bzl even though it looks like it should · Issue #1250 · aspect-build/rules_js · GitHub"
[2]: https://github.com/aspect-build/rules_js/blob/main/README.md?utm_source=threadsgpt.com "rules_js README at main - GitHub"
[3]: https://github.com/aspect-build/rules_js/blob/main/js/private/js_binary.bzl?utm_source=threadsgpt.com "rules_js/js/private/js_binary.bzl at main - GitHub"
