# Implementation Notes

## Console Mutation Lifecycle Mapping

This mapping is the implementation baseline for the `align-console-config-lifecycle` change.

### Runtime

These mutations already have dedicated Gateway runtime RPCs and should remain on those paths:

- `agents.update`
  - Surface: Agent model change
  - Effect: new runs read updated config; in-flight runs are not switched
- `skills.update`
  - Surface: Skills page enable/disable, API key/env updates from skill detail
  - Effect: runtime/path-specific Gateway behavior; no generic restart affordance in Office
- `cron.add`
- `cron.update`
- `cron.remove`
- `cron.run`
  - Surface: Cron page and agent cron management
  - Effect: immediate runtime update

### Save

These mutations edit persisted config and should default to `config.set`, allowing Gateway's
native config watcher and reload plan to decide whether the change is noop, hot-reloadable, or
restart-requiring:

- Settings provider add/edit/delete
- Agent tools config save
- Agent skills allowlist save

### Apply

These mutations use the same config editing model as `save`, but the user explicitly chooses
"save and restart now", so Office should call `config.apply`:

- Settings provider changes with explicit restart confirmation
- Agent tools or skills-allowlist changes when the UI offers a restart action and a valid config
  snapshot is available

### Save With CLI Restart Hint

Fallback mode for config-backed flows when Office cannot safely perform `config.apply` from the
current UI context:

- any config-backed mutation lacking a valid `raw`/`hash` snapshot pair
- any config-backed mutation where apply cannot be completed after save

The fallback instruction shown to the user should be:

`openclaw gateway restart`
