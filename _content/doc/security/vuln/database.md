---
# Copyright (c) 2009 The Go Authors.

# Documentation licensed under the 3-Clause BSD License.
# The original work was translated from English into Brazilian Portuguese.
# https://github.com/golang/website/blob/master/LICENSE

title: Go Vulnerability Database
layout: article
---

[Back to Go Vulnerability Management](/security/vuln)

## Overview

The Go vulnerability database ([https://vuln.go.dev](https://vuln.go.dev))
serves Go vulnerability information in the
[Open Source Vulnerability (OSV) schema](https://ossf.github.io/osv-schema/).

You can also browse vulnerabilities in the database at [pkg.go.dev/vuln](https://pkg.go.dev/vuln).

**Do not** rely on the contents of the x/vulndb Git repository. The YAML files in that
repository are maintained using an internal format that may change
without warning.

## Contributing

We would love for all Go package maintainers to [contribute](/s/vulndb-report-new)
information about public vulnerabilities in their own projects,
and [update](/s/vulndb-report-feedback) existing information about vulnerabilities
in their Go packages.

We aim to make reporting a low friction process,
so feel free to [send us your suggestions](/s/vuln-feedback).

Please **do not** use the forms above to report a vulnerability in the Go
standard library or sub-repositories.
Instead, follow the process at [go.dev/security/policy](/security/policy)
for vulnerabilities about the Go project.

## API

The canonical Go vulnerability database, [https://vuln.go.dev](https://vuln.go.dev),
is an HTTP server that can respond to GET requests for the endpoints specified below.

The endpoints have no query parameters, and no specific headers are required.
Because of this, even a site serving from a fixed file system (including a `file://` URL)
can implement this API.

Each endpoint returns a JSON-encoded response, in either uncompressed
(if requested as `.json`) or gzipped form (if requested as `.json.gz`).

The endpoints are:

- `/index/db.json[.gz]`

  Returns metadata about the database:

  ```json
  {
    // The latest time the database should be considered
    // to have been modified, as an RFC3339-formatted UTC
    // timestamp ending in "Z".
    "modified": string
  }
  ```

  Note that the modified time *should not* be compared to wall clock time,
  e.g. for purposes of cache invalidation, as there may a delay in making
  database modifications live.

  See [/index/db.json](https://vuln.go.dev/index/db.json) for a live example.

- `/index/modules.json[.gz]`

  Returns a list containing metadata about each module in the database:

  ```json
  [ {
    // The module path.
    "path": string,
    // The vulnerabilities that affect this module.
    "vulns":
      [ {
        // The vulnerability ID.
        "id": string,
        // The latest time the vulnerability should be considered
        // to have been modified, as an RFC3339-formatted UTC
        // timestamp ending in "Z".
        "modified": string,
        // (Optional) The module version (in SemVer 2.0.0 format)
        // that contains the latest fix for the vulnerability.
        // If unknown or unavailable, this should be omitted.
        "fixed": string,
      } ]
  } ]
  ```

  See [/index/modules.json](https://vuln.go.dev/index/modules.json) for a live example.

- `/index/vulns.json[.gz]`

  Returns a list containing metadata about each vulnerability in the database:

  ```json
   [ {
       // The vulnerability ID.
       "id": string,
       // The latest time the vulnerability should be considered
       // to have been modified, as an RFC3339-formatted UTC
       // timestamp ending in "Z".
       "modified": string,
       // A list of IDs of the same vulnerability in other databases.
       "aliases": [ string ]
   } ]
  ```

  See [/index/vulns.json](https://vuln.go.dev/index/vulns.json) for a live example.

- `/ID/$id.json[.gz]`

  Returns the individual report for the vulnerability with ID `$id`,
  in OSV format (described below in [Schema](#schema)).

  See [/ID/GO-2022-0191.json](https://vuln.go.dev/ID/GO-2022-0191.json)
  for a live example.

### Bulk download

To make it easier to download the entire Go vulnerability database,
a zip file containing all the index and OSV files is available at
[vuln.go.dev/vulndb.zip](https://vuln.go.dev/vulndb.zip).

### Usage in `govulncheck`

By default, `govulncheck` uses the canonical Go vulnerability database at [vuln.go.dev](https://vuln.go.dev).

The command can be configured to contact a different vulnerability database using the `-db` flag,which accepts a vulnerability database URL with protocol `http://`, `https://`, or `file://`.

To work correctly with `govulncheck`, the vulnerability database specified must implement the API described above. The `govulncheck` command uses compressed ".json.gz" endpoints when reading from an http(s) source, and the ".json" endpoints when reading from a file source.

### Legacy API

The canonical database contains some additional endpoints that are part of a legacy API.
We plan to remove support for these endpoints soon. If you are relying on the legacy API
and need additional time to migrate, [please let us know](/s/govulncheck-feedback).

## Schema

Reports use the
[Open Source Vulnerability (OSV) schema](https://ossf.github.io/osv-schema/).
The Go vulnerability database assigns the following meanings to the fields:

### id

The id field is a unique identifier for the vulnerability entry. It is a string
of the format GO-\<YEAR>-\<ENTRYID>.

### affected

The [affected](https://ossf.github.io/osv-schema/#affected-fields) field is a
JSON array containing objects that describes the module versions that contain
the vulnerability.

#### affected[].package

The
[affected[].package](https://ossf.github.io/osv-schema/#affectedpackage-field)
field is a JSON object identifying the affected _module._ The object has two
required fields:

- **ecosystem**: this will always be "Go"
- **name**: this is the Go module path
  - Importable packages in the standard library will have the name _stdlib_.
  - The go command will have the name _toolchain_.

#### affected[].ecosystem_specific

The
[affected[].ecosystem_specific](https://ossf.github.io/osv-schema/#affectedecosystem_specific-field)
field is a JSON object with additional information about the vulnerability,
which is used by Go's vulnerability detection tools.

For now, ecosystem specific will always be an object with a single field,
`imports`.

##### affected[].ecosystem_specific.imports

The `affected[].ecosystem_specific.imports` field is a JSON array containing
the packages and symbols affected by the vulnerability. Each object in the
array will have these two fields:

- **path:** a string with the import path of the package containing the vulnerability
- **symbols:** a string array with the names of the symbols (function or method) that contains the vulnerability
- **goos**: a string array with the execution operating system where the symbols appear, if known
- **goarch**: a string array with the architecture where the symbols appear, if known

### database_specific

The `database_specific` field contains custom fields specific to the Go vulnerability database.

#### database_specific.url

The `database_specific.url` field is a string representing the fully-qualified
URL of the Go vulnerability report, e.g, "https://pkg.go.dev/vuln/GO-2023-1621".

#### database_specific.review_status

The `database_specific.review_status` field is a string representing the review
status of the vulnerability report. If not present, the report should be
considered `REVIEWED`. The possible values are:

- `UNREVIEWED`: The report was automatically generated based on another source, such as
a CVE or GHSA. Its data may be limited and has not been verified by the Go team.
- `REVIEWED`: The report originated from the Go team, or was generated based on an external source.
A member of the Go team has reviewed the report, and where appropriate, added additional data.

For information on other fields in the schema, refer to the [OSV spec](https://ossf.github.io/osv-schema).

## Note on Versions

Our tooling attempts to automatically map modules and versions in
source advisories to canonical Go modules and versions, in accordance with
standard [Go module version numbers](/doc/modules/version-numbers). Tools like
`govulncheck` are designed to rely on these standard versions to determine
whether a Go project is affected by a vulnerability in a dependency or not.

In some cases, such as when a Go project uses its own versioning scheme,
the mapping to standard Go versions can fail. When this happens, the Go
vulnerability database report may conservatively list all Go versions as
affected. This ensures that tools such as `govulncheck` do not fail to report
vulnerabilities due to unrecognized version ranges (false negatives).
However, conservatively listing all versions as affected may cause tools to
incorrectly report a fixed version of a module as containing the vulnerability
(false positives).

If you believe `govulncheck` is incorrectly reporting (or failing to report) a
vulnerability, please
[suggest an edit](https://github.com/golang/vulndb/issues/new?assignees=&labels=Needs+Triage%2CSuggested+Edit&template=suggest_edit.yaml&title=x%2Fvulndb%3A+suggestion+regarding+GO-2024-2965&report=GO-XXXX-YYYY)
to the vulnerability report and we will review it.

## Examples

All vulnerabilities in the Go vulnerability database use the OSV schema
described above.

See the links below for examples of different Go vulnerabilities:

- **Go standard library vulnerability** (GO-2022-0191):
  [JSON](https://vuln.go.dev/ID/GO-2022-0191.json),
  [HTML](https://pkg.go.dev/vuln/GO-2022-0191)
- **Go toolchain vulnerability** (GO-2022-0189):
  [JSON](https://vuln.go.dev/ID/GO-2022-0189.json),
  [HTML](https://pkg.go.dev/vuln/GO-2022-0189)
- **Vulnerability in Go module** (GO-2020-0015):
  [JSON](https://vuln.go.dev/ID/GO-2020-0015.json),
  [HTML](https://pkg.go.dev/vuln/GO-2020-0015)

## Excluded Reports

The reports in the Go vulnerability database are collected from different
sources and curated by the Go Security team. We may come across a vulnerability advisory
(for example, a CVE or GHSA) and choose to exclude it for a variety of reasons.
In these cases, a minimal report will be created in the x/vulndb repository,
under
[x/vulndb/data/excluded](https://github.com/golang/vulndb/tree/master/data/excluded).

Reports may be excluded for these reasons:

- `NOT_GO_CODE`: The vulnerability is not in a Go package,
  but it was marked as a security advisory for the Go ecosystem by another source.
  This vulnerability cannot affect any
  Go packages. (For example, a vulnerability in  a C++ library.)
- `NOT_IMPORTABLE`: The vulnerability occurs in package `main`, an `internal/`
  package only imported by package `main`, or some  other location which can
  never be imported by another module.
- `EFFECTIVELY_PRIVATE`: While the vulnerability occurs in a Go package which
  can be imported by another module, the package is not intended for external
  use and is not likely to ever be imported outside the module in which it is
  defined.
- `DEPENDENT_VULNERABILITY`: This vulnerability is a subset of another
  vulnerability in the database. For example, if package A contains a
  vulnerability, package B depends on package A, and there are separate CVE IDs
  for packages A and B, we might mark the report for B as a dependent
  vulnerability entirely superseded by the report for A.
- `NOT_A_VULNERABILITY`: While a CVE ID or GHSA has been assigned, there is no
  known vulnerability associated with it.
- `WITHDRAWN`: The vulnerability has been withdrawn by its source.

At the moment, excluded reports are not served via
[vuln.go.dev](https://vuln.go.dev) API. However, if you have
a specific use case and it would be helpful to have access to this information
through the API,
[please let us know](/s/govulncheck-feedback).
