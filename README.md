# Tricentis Tosca TSU Format Documentation

Unofficial, community-derived documentation of the `.tsu` (Tosca Suite) file format used by [Tricentis Tosca](https://www.tricentis.com/products/automate-continuous-testing-tosca).

## What is a TSU file?

A `.tsu` file is a workspace export from Tosca Commander. It contains the full definition of a test project — folders, test cases, reusable step libraries, UI modules, parameters, and attached files — serialised as a flat JSON array of cross-referenced entities, compressed with gzip.

Tosca does not publish a formal spec for this format. The documentation here was reverse-engineered from real exports and is intended to help teams build tooling around the format without having to rediscover it from scratch.

## Contents

- **[FORMAT_GUIDE.md](FORMAT_GUIDE.md)** — the full format reference:
  - Container format and decompression
  - All 26 entity types with every attribute and association key
  - All encoded sub-formats (base64 gzip XML blobs, plain base64 JSON, inline JSON)
  - Enum tables (ActionMode, DataType, Operator, StatementType, ParamType, …)
  - Tosca expression language reference (`{B[]}`, `{TDS[]}`, `{MATH[]}`, …)
  - Self-healing data structure
  - Entity relationship graph
  - Known gaps and limitations

## Quick start

```js
const fs = require('fs');
const zlib = require('zlib');

// Decompress and parse
const raw = zlib.gunzipSync(fs.readFileSync('export.tsu.gzip'));
const { Entities } = JSON.parse(raw);

// Index by Surrogate for O(1) cross-reference lookups
const byId = Object.fromEntries(Entities.map(e => [e.Surrogate, e]));

// Group by type
const byClass = Entities.reduce((acc, e) => {
  (acc[e.ObjectClass] ??= []).push(e);
  return acc;
}, {});

// Example: list all test case names
byClass.TestCase.forEach(tc => console.log(tc.Attributes.Name));
```

## Entity types at a glance

| Type | Role |
|------|------|
| `TCProject` | Root node |
| `TCComponentFolder` / `TCFolder` | Folder tree |
| `TCConfiguration` | Named run configuration |
| `TestCase` | A single test scenario |
| `TestStepFolder` | Phase within a test case (Precondition / Test / Cleanup) |
| `ReuseableTestStepBlock` | Reusable step sequence (library function) |
| `TestStepFolderReference` | Call-site for a reusable block |
| `XTestStep` | Single executable step |
| `XTestStepValue` | One parameter/action value on a step |
| `TestCaseControlFlowItem` | If / While control flow |
| `XModule` / `ApiModule` | UI component map / HTTP API definition |
| `XModuleAttribute` | Individual element on a module |
| `Parameter` / `ParameterLayer` | Parameter contract on a reusable block |
| `ParameterLayerReference` / `ParameterReference` | Parameter values at a call-site |
| `XParam` | Extended property / self-healing metadata |
| `OwnedFile` / `FileContent` | Attached files (PNG, PDF, …) |
| `RecoveryScenario` | Error recovery procedure |

See the format guide for the complete attribute and association schema for each type.

## Security considerations

TSU exports can contain sensitive data embedded in encoded fields:

- `TCConfiguration.TestConfigurationParameters` may contain OAuth2 client credentials (client ID + secret) encoded as nested base64 JSON inside a gzip XML blob.
- `ApiModule.Headers` on response modules often contains live session cookies and security tokens captured during API scanning.
- `ApiModule.Payload` may contain sensitive API response bodies.

None of this data is encrypted — it is only base64-encoded. Treat TSU files with the same care as credential files and avoid committing them to version control.

## Decoding the blobs

Three distinct encoding layers appear in the format:

| Field | Encoding | How to decode |
|-------|----------|--------------|
| `TestConfigurationParameters` | base64 → gzip → UTF-8 XML | `zlib.gunzipSync(Buffer.from(value, 'base64')).toString('utf8')` |
| `TCProperties` | base64 → gzip → UTF-8 XML | same |
| `ApiModule.ExplicitConnection` | base64 → UTF-8 JSON | `Buffer.from(value, 'base64').toString('utf8')` |
| `ApiModule.Headers` | base64 → UTF-8 JSON | same |
| `ApiModule.Payload` | base64 → UTF-8 string | same |
| `FileContent.Data` | base64 → binary | `Buffer.from(value, 'base64')` |

## Contributing

This documentation was built by analysing real exports. If you find inaccuracies, undocumented fields, or enum values not listed here, please open an issue or a pull request.

Areas that would benefit most from contributions:
- Full `ActionMode` bitmask decoding (confirmed mapping against Tosca SDK source)
- `ContentPolicy` string format on `TCFolder` / `TCComponentFolder`
- `CheckOutState` values for checked-out entities
- Execution log schema (empty in typical exports)

## Disclaimer

This is an unofficial community resource. Tricentis has not endorsed or verified this documentation. The format may change between Tosca versions without notice. Version numbers observed in `TCProperties.Version` fields range from `10.1` to `24.2`.
