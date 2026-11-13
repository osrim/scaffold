# Project name

One sentence on what this project does and who it is for.

## After you create a repository from this template

Finish the setup in the new repository:

1. Replace the title and description at the top of this file.
2. Add the project's tools under `[tools]` in `mise.toml`, then run
   `mise install`.
3. Add the project's checks as the `ci` task in `mise.toml`. CI runs
   `mise run ci` through the shared workflow in `osrim-agents/ci`, and
   passes with a warning while the task is missing.
4. Run `nik install` to restore the agent skills listed in `niksi-lock.json`.
5. In the Renovate app settings on GitHub, check that Renovate has access to
   the new repository.
6. Delete this section.

## License

This project is licensed under the
[GNU Affero General Public License v3.0](LICENSE).
