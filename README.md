# Rockstarr AI Plugins

This repository is the **public release surface** for the Rockstarr AI
Growth Operating System — a collection of Cowork plugins delivered to
Rockstarr's B2B clients.

Each top-level folder under `plugins/` is an independent plugin. The
state of each folder reflects the most recent published release of that
plugin. Earlier releases are accessible via Git tags of the form
`<plugin>/v<version>` (e.g. `rockstarr-content/v0.7.0`).

## This repo is read-only for the world

- **Pull requests are not accepted here.** Development happens in a
  separate private repository.
- **Issues** filed here will not be triaged in this repo. Contact
  Rockstarr & Moon directly: hello@rockstarrandmoon.com.
- Commits land here only as part of the publish flow, one per
  released plugin version.

## How clients install these plugins

Clients install plugins through the Rockstarr marketplace, not from
this repo directly. Each Rockstarr AI client receives a per-client
marketplace token and installs through Cowork (Claude Desktop) using
that token. Downloading folders from this repo by hand is supported
but not the intended path.

## Licensing

Source in this repo is © Rockstarr & Moon, all rights reserved, unless
an individual file specifies otherwise (some shared assets — like the
mandatory `stop-slop` reference — are MIT-licensed). The collection as
a whole is proprietary.

## Contact

hello@rockstarrandmoon.com
