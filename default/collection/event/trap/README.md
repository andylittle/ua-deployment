# default/collection/event/trap

SNMP trap rules managed by this repo.

Unlike other content under `default/`, this directory is a **full replacement** of the Assure1 out-of-the-box trap rules — the defaults are intentionally overwritten on each deploy. Do not rename or move this directory to a custom name; the pipeline deploys it directly to:

```
/opt/assure1/var/checkouts/core/default/collection/event/trap/
```

The `json/` subdirectory contains rules that are managed outside of git and must not be overwritten by the pipeline — it is excluded from rsync and not committed here.
