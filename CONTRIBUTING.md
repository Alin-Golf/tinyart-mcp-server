# Contributing (lite)

This repository is the public description of the TINY ART hosted service. Its source code is not here, so there is no code to change.

Useful contributions:

- Report a mistake or an unclear passage in `README.md`, `openapi.json` or `llms.txt` by opening an issue.
- Report that a documented endpoint does not behave as described. Include the request and the answer, without credentials.
- Suggest an addition for agent developers, such as a client example.

Please do not include API keys, owner secrets or personal data in an issue. For a security problem, follow `SECURITY.md` instead.

`openapi.json` is generated from the service. A pull request that edits it by hand will be closed with a pointer to an issue; the maintainers regenerate it.

The repository is licensed under Apache-2.0 (`LICENSE`). Under section 5 of that license, a contribution you intentionally submit is licensed under the same terms unless you state otherwise. Pull requests are welcome for documents and examples; edits to `openapi.json` are not (see above).

Questions: `support@tinyart.es`. Misuse or rule-breaking content: `abuse@tinyart.es`. Security: `security@tinyart.es`.
