# How the ISIMIP repositories fit together

Load this when a diff touches an interface that another repository depends on. The
question to ask for every such change: *who else reads or calls this, and does the
change ship together with them?*

```
                   ISIMIP protocol (protocol.isimip.org, protocol2…, protocol4…)
                   definitions · pattern · schema · tree (JSON, versioned by commit)
                                   │
                                   ▼
                          isimip-utils  (PyPI library)
          cli · config · protocol · patterns · netcdf · xarray · extractions · plot
              │                    │                         │
              ▼                    ▼                         ▼
          isimip-qc  ──files──▶ isimip-publisher ──────▶ metadata DB (PostgreSQL)
        (check/fix NetCDF,     (copy, JSON sidecars,      │ datasets, files, resources,
         QC global attrs)       DataCite DOIs)            │ search, tree …
                                                          ▼
          isimip-ea                                   isimip-data (Django, data.isimip.org)
        (extract/plot, uses                           REST API /api/v1 ─▶ isimip-client,
         isimip-utils only)                           files API, users' scripts
```

## isimip-utils is a published library

- isimip-qc, isimip-publisher and isimip-ea depend on it via PyPI with a
  compatible-release pin. A change in isimip-utils is not seen by a tool until it is
  released *and* the tool's pin allows it.
- Treat every public function as API: changed signatures, renamed kwargs, changed
  defaults or return shapes (e.g. a tuple growing a member, `list` vs `set`) break
  the tools. Grep the three tool repos (and `examples/`, `docs/`) for call sites
  when reviewing such a change, and ask for a version bump/changelog note.
- `cli.ArgumentParser`/`build_default_args` and `config.Settings` are the backbone of
  every tool's configuration: a behaviour change there changes how *all* tools read
  `isimip.toml` and `ISIMIP_*` environment variables.
- `PROTOCOL_LOCATIONS` exists in several places (isimip-utils `protocol.py`,
  isimip-data `config/settings/base.py`, `--protocol-location` defaults in
  isimip-qc/isimip-publisher). Adding/removing a location in one place usually
  means updating the others.

## The ISIMIP protocol

- The tools fetch protocol JSON at runtime; its content changes independently of the
  code (keys get renamed, e.g. the calendar key, or are missing for some sectors).
  Code must tolerate missing optional keys and should not keep fallbacks for
  renamed keys forever (isimip-qc removed such fallbacks once upstream was fixed).
- isimip-qc records `isimip_protocol_version` (the protocol `commit`) and
  `isimip_qc_version` in the files; isimip-publisher reads global attributes such as
  `isimip_id` from the file headers, so a change in what QC writes can surface in
  the published metadata.

## isimip-publisher ⇄ isimip-data: the metadata database

- isimip-publisher owns the schema (SQLAlchemy models + indexes in
  `isimip_publisher/utils/database.py`, created/updated via `init`/`update_views`).
- isimip-data maps the same tables with **unmanaged** Django models
  (`managed = False`, `db_table = '...'`) in `isimip_data/metadata/models.py`, read
  from the `metadata` database alias. Django migrations do not touch them.
- Therefore a schema change (new/renamed column, changed relation such as a FK
  becoming an M2M, computed columns like `root_id`/`last_changed`, the `search` and
  `tree` tables, the `identifiers`/`specifiers` materialized views) needs:
  1. the SQLAlchemy model change and data migration/backfill path in the publisher,
  2. the matching Django model change in isimip-data, plus every serializer,
     filter, viewset, template and JSON-LD/sitemap that used the old field,
  3. updated SQL dumps: `isimip-publisher/testing/sql` and
     `isimip-data/testing/sql/test_isimip_metadata.sql`,
  4. a deployment order that doesn't leave data.isimip.org querying columns that
     don't exist yet (or no longer exist).
  When reviewing one side only, point out what the other side needs.
- Dataset/file semantics shared by both sides: `version` (`YYYYMMDD`), `public`,
  `restricted`, `archived` timestamps, links (`target_id`, `root_id`), `rights`,
  `specifiers` (JSONB), `identifiers`, `tree_path`. A change in how the publisher
  sets these changes what isimip-data shows and who may download it.

## JSON sidecar files and the REST API

- The publisher writes a `.json` file next to each published file
  (`write_*_jsons`) with the file's metadata; downstream users and services read
  them. Changing their keys is a public format change.
- isimip-data's `/api/v1/` endpoints (datasets, files, resources, caveats,
  annotations, search, tree, checksums, …) are used by isimip-client, the frontend,
  and external users. Field renames, type changes (object → list), pagination or
  filter changes are breaking and should be called out, documented, or provided
  with a compatibility field.

## DataCite DOIs

- isimip-publisher registers and updates DOIs (`insert_doi`, `update_doi`,
  `register_doi`) with prefix `10.48364`. Registered DOIs are permanent; anything
  that could register the wrong metadata or a wrong URL is high severity.
  `--datacite-test-mode` and `--skip-registration` must keep working.

## isimip-ea and user scripts

- isimip-ea and the scripts in `isimip-utils/examples` use the extraction/xarray/plot
  API; changes in `isimip_utils.extractions` or `plot` should be checked against
  them (aggregations, region/period objects, legend/title formatting).
