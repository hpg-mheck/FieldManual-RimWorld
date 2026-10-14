# Knacks

Knacks are focused, reusable knowledge units intended to be loaded for a
concrete task or investigation. This pack supplements the stock knacks in
[FieldManual](https://github.com/Hard-Problems-Group-LLC/FieldManual) and
follows its conventions: an overview per topic as
`<topic>.overview.knack.md`, API companions as
`<topic>.api.<subsystem>.knack.md`, each fileset in a `<topic>.knack/`
directory, and an index in each area. See FieldManual's
`knacks/authoring-guide.md` for sizing, naming, sourcing, currency, and
review guidance.

## Areas

- [`games/`](games/README.md) — knacks for building content for specific
  games, starting with RimWorld

Start with the narrowest area that matches the task, then follow its local
index. Load these alongside FieldManual's stock knacks; a project's own
`knacks/` tree may refine both, and contradictions should be recorded and
resolved rather than guessed away.

## Maintenance Model

Knacks are maintained prose, not generated output. Review them for:

- accurate scope and an explicit game version;
- claims labelled by evidence, with decompiled findings marked as such;
- absence of secrets, local identities, and machine-specific paths;
- working internal links; and
- current authoritative references.
