# Elysian Homes Claude Tools — DRAFT

This repository packages the existing Elysian Homes Claude skills as a Claude Code plugin.

## Current draft contents

- elysian-transactions-copilot
- cma-output-skill
- marketing-designer-skill
- cma-intelligence
- elysian-homes-master-os
- elysian-contract-intake
- compliance-review
- elysian-agent-roster

The existing skill files were copied verbatim. No skill instructions were edited in this draft.

`elysian-contract-intake/references/compliance-report.md` is included because the intake skill explicitly references that path.

## Not yet included

- elysian-sign-ordersetup
- elysian-sign-order automation skill

## External dependencies

Some skills refer to external systems/tools such as BoldTrail Back Office/Brokermint, kvCore/BoldTrail front office, Gmail, and Google Drive. Installing this plugin does not itself grant access to those systems. Each Claude environment must have the required tools/connections and permissions configured.

## Installation (after publishing this repository)

In Claude Code, add the GitHub repository as a plugin marketplace, then install `elysian-homes-tools@elysian-homes`.

Exact GitHub commands/URL will be added after the repository location is chosen.

## Draft status

This package has intentionally not altered the source skill logic. Any compatibility edits discovered during validation should be reviewed and approved before they are made.
