# Known issues, pitfalls and tech debt

The analysis of v3.20.0 (2026-10-22) found 17 issues, all fixed since. A second review (2026-10-23) found 6 high, 15 medium and about 30 low-severity issues; the high and medium ones are fixed with tests, the low ones are listed below. If you find a new issue, add it here with where it is and how it shows up, and remove it again in the change that fixes it.

## Tooling pitfalls

| Pitfall                                                                                                                         | What to do                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 6 files are off-format upstream (`prettier --check "src/**/*.{ts,tsx}"`).                                                       | Format only the files you touch. Don't reformat the whole repo in a feature change. |
| TypeScript is non-strict (`strictNullChecks: false`, `noImplicitAny: false`). The compiler won't catch `null`/`undefined` bugs. | Guard optional data yourself (`?.`, `??`), especially for REST and snapshot input.  |

## Dev Mode

| Issue                                                                                                                                                                       | Where                                                         | What to do                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Figma doesn't document `<a download>`, `<input type=file>` or clipboard access in the Dev Mode or VS Code iframe.                                                           | `downloadTokensFile.ts`, `CodePreviewView` (`clipboard-copy`) | Check them in the manual Dev Mode test. If one fails in VS Code, document it in README "Dev Mode". |
| A 2024 forum report says a published `["figma", "dev"]` plugin didn't show up in Dev Mode although the local build did.                                                     | `manifest.json`                                               | Check the published plugin in Dev Mode after the next Community release.                           |
| Reading variables and styles in Dev Mode is allowed by the general rule ("plugin APIs that read from the document"), but no page names `getLocalVariablesAsync` explicitly. | `PluginAPIResolver`                                           | Covered by the manual Dev Mode test.                                                               |

## Open low-severity findings (review of 2026-10-23)

### Export

| Issue                                                                                                                      | Where                                                                                                    | What to do                                                                                            |
| -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Opacity, letter spacing, paragraph spacing and shadow values keep float32 noise (`0.3333000183105469`).                    | `normalizeValue.ts`, `composeColor.ts` (`normalizeOpacity`), `text/getLetterSpacing.ts`, `makeDimension` | Round with `Decimal(…).toDecimalPlaces(6)`, after dividing by 100.                                    |
| HSL hue can be `360`; DTCG allows `[0, 360)`.                                                                              | `color/convertRGBA.ts` (`rgbaToHsla`)                                                                    | `Math.round(h) % 360`.                                                                                |
| Hidden effects and paints (`visible: false`) are exported.                                                                 | `styles/effectStylesToTokens.ts`, `styles/colorStylesToTokens.ts`                                        | Filter `visible !== false` first.                                                                     |
| Grid styles ignore bound variables and `alignment` (STRETCH exports an ignored `sectionSize`, CENTER an ignored `offset`). | `styles/gridStylesToTokens.ts`                                                                           | Alias bound fields with `getAliasVariableName`; leave out the fields Figma ignores for the alignment. |
| A style group silently replaces a variable group with the same key.                                                        | `mergeStylesIntoTokens.ts` (`Object.assign`)                                                             | Deep-merge, or warn on a key collision.                                                               |
| `omitCollectionNames` with `splitByMode` treats top-level groups as collections and mixes modes of several collections.    | `export.ts`                                                                                              | Split by mode before flattening, or reject the combination with a warning.                            |
| An alias to a variable that was skipped still points at it, without a warning.                                             | `variablesToTokens.ts`, `getAliasVariableName.ts`                                                        | Collect the skipped ids and warn (or write `#missing#`).                                              |
| The preview's token count counts multi-mode tokens several times.                                                          | `countTokens.ts`                                                                                         | Don't descend into `$extensions`; stop at objects with `$value`.                                      |
| The `ModeId … not found` warning has no `[tokens-bruecke]` prefix.                                                         | `variablesToTokens.ts`                                                                                   | Add the prefix.                                                                                       |

### Import

| Issue                                                                            | Where                                                             | What to do                                                            |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------- |
| A boolean token with the string `"false"` becomes `true`.                        | `tokensToVariables.ts` (`convertTokenValueToFigmaValue`, boolean) | Map `"true"`/`"false"`, throw for other strings.                      |
| `scopes: []` doesn't round-trip; new variables keep `ALL_SCOPES`.                | `tokensToVariables.ts` (scopes in the first pass)                 | Assign an explicit empty array too.                                   |
| Top-level `$themes` / `$metadata` (Tokens Studio) become empty collections.      | `tokensToVariables.ts` (collection loop)                          | Skip top-level `$` keys; create a collection only when it has tokens. |
| Two local collections with the same name: values go to the wrong variable.       | `tokensToVariables.ts` (`variableMap`)                            | Key the map by collection id, warn about the duplicate name.          |
| A `fontWeight` token `"700"` for a STRING variable is written as the number 700. | `tokensToVariables.ts` (`fontWeight`)                             | Convert to a number only for FLOAT variables.                         |

### CLI and API

| Issue                                                                                                                                                                             | Where                                                                         | What to do                                                    |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------- |
| A collection named `..` makes `--split-by-mode` write outside `--output`; names that differ only by `/` vs `-` or by case overwrite each other.                                   | `splitTokensIntoFiles.ts` (`safeName`)                                        | Replace dot-only names, fail on duplicate target paths.       |
| `FIGMA_OUTPUT` with an explicit `--stdout` fails as "mutually exclusive"; flags should win.                                                                                       | `src/cli/index.ts` (`.check`)                                                 | Check conflicts only between flags actually passed.           |
| `tokens-bruecke --quiet init` isn't recognized as `init`.                                                                                                                         | `src/cli/index.ts` (`isInitCommand`)                                          | Use the first positional argument.                            |
| A config file containing `null` crashes with a stack trace; `convertTokens(x, null)` throws a raw `TypeError`.                                                                    | `src/cli/index.ts`, `src/api/index.ts`                                        | Reject non-objects with a clear message / `INVALID_ARGUMENT`. |
| The 403 hint names only `file_variables:read`; styles also need `file_content:read`.                                                                                              | `src/cli/index.ts`, `src/api/index.ts`, README, SKILL.md                      | Name both scopes.                                             |
| One `null` node from `/nodes` aborts the whole REST export.                                                                                                                       | `restApiResolver.ts` (`fetchStyleNodes`)                                      | Skip it with a warning.                                       |
| `omitCreatedAt` has no plugin toggle; `sanitizeProfileConfig` merges `includedStyles` shallowly; the schema describes `includeScopes` and `includeValueStringKeyToAlias` wrongly. | `AdvancedSettingsView`, `storageConfig.ts`, `schemas/cli-options.schema.json` | Add the toggle, merge per style type, fix the descriptions.   |

### Plugin UI

| Issue                                                                                                             | Where                                            | What to do                                                          |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------- |
| Two toasts shown in the same tick: the first is lost.                                                             | `components/Toast` (`setToasts([...toasts, …])`) | Use a functional update.                                            |
| The 10 s push-spinner reset is never cleared and can re-enable the button during a slow push.                     | `SettingsView` (`isPushing` effect)              | Remove it; `runPush` always resets the spinner.                     |
| Style-name inputs show stale values after a shared-profile update (`react-figma-ui` `Input` copies `value` once). | `SettingsView`                                   | Add the value or `sharedUpdatedAt` to the `key`.                    |
| An adopted shared profile keeps servers enabled without a token; pushes then fail with misleading errors.         | `sharedProfiles.ts` (`mergeSharedProfile`)       | Skip servers with an empty required credential, with a toast.       |
| `stripCredentials` uses a denylist, so a new server's secret field would be shared.                               | `sharedProfiles.ts`                              | Build the shared profile from an allowlist.                         |
| A non-Latin-1 JSONBin bin name or Custom URL header value throws a cryptic fetch error.                           | `pushToJSONBin.ts`, `pushToCustomURL.ts`         | Validate on Save, or encode the bin name.                           |
| The Custom URL sends no `Content-Type`, so `express.json()`-style servers see no body.                            | `pushToCustomURL.ts`                             | Default to `application/json` (adds a CORS preflight; document it). |
