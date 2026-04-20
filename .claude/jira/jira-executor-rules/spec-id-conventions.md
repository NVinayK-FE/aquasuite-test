# Spec ID Conventions {#spec-id}

IDs are dotted paths that match the spec file and its sections.

`<spec-id>.<section>.<sub-section>...`

## Rules {#spec-id.rules}

- Spec ID comes from the filename (kebab-case, shortened)
- Words longer than 8 characters use a short form
- All lowercase, kebab-case inside segments, dots between
- IDs are written as `{#id}` — wrapped in curly braces with a `#` prefix

## Example {#spec-id.example}

- Example Spec {#ex-spec}
  - Main Section {#ex-spec.main-sect}
    - Sub Section {#ex-spec.main-sect.sub-sect}
    - Sub Section 2 {#ex-spec.main-sect.sub-sect2}
  - Other Section {#ex-spec.other-sect}
