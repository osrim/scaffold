# Project name

One sentence on what this project does and who it is for.

## After you create a repository from this template

Finish the setup in the new repository:

1. Replace the title and description at the top of this file.
2. Add the project's tools under `[tools]` in `mise.toml`, then run
   `mise install`.
3. Run `nik install` to restore the agent skills listed in `niksi-lock.json`.
4. In the Renovate app settings on GitHub, check that Renovate has access to
   the new repository.
5. If the repository has no CI checks, add `"ignoreTests": true` to
   `renovate.json`. Renovate merges an update only after its checks pass.
6. Delete this section.

## License

This project is licensed under the
[GNU Affero General Public License v3.0](LICENSE).
