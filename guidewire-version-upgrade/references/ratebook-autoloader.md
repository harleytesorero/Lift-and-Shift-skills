# Rate Book Auto-Loader Setup

## When to Apply

Check if the Source+LOB has **custom rate books** for the LOB being migrated. Look for:
- `config/content/cust-ratebooks/<LOB>/` — XML rate book files
- `gsrc/gw/surepath/pc/configuration/rating/autoratebookload/` — SurePath auto-loader framework
- A custom rating engine (e.g., `CustIMRatingEngine_Ext`) that calls `rateBook.getTable(...)` for LOB-specific rate tables

If any of these exist, the LOB requires the auto-loader to import rate books into the H2 database on server startup. Without it, quoting will fail with:
```
java.lang.IllegalStateException: Rate book code: gx_rtm_demo_rating, edition: 1 does not contain table <LOB>CovToRateRoutineCodeLookup_Ext
```

## What to Migrate (WS-4: Rating/UW)

### 1. Auto-Loader Framework (33 files)

Copy the entire directory from Source+LOB to Target:
```
gsrc/gw/surepath/pc/configuration/rating/autoratebookload/
  api/          (3 files)
  impl/         (15 files)
  plugin/       (7 files)
  util/         (6 files)
  webservice/   (2 files)
```

**Critical:** The directory must be `rating/autoratebookload/`, NOT flattened into `rating/` directly. The package declarations in all 33 files expect `gw.surepath.pc.configuration.rating.autoratebookload.*`.

### 2. Startup Plugin Registration

Copy from Source+LOB:
```
config/plugin/registry/AutomaticRateBookLoadOnSystemStartup_SP.gwp
```

This registers an `IStartablePlugin` that triggers rate book loading when the server starts. The `AutoLoad=Yes` parameter enables it.

### 3. RuntimePropertyGroup Typelist Extension

Copy from Source+LOB:
```
config/extensions/typelist/RuntimePropertyGroup.autoratebookloader.ttx
```

Adds the `auto_rate_book_loader_sp` typecode so the runtime property system can read the auto-loader config.

### 4. Runtime Properties

Copy from Source+LOB:
```
runtimeproperties/local/AutoRateBookLoader_SP.xml
```

Contains:
- `AutomaticallyLoadRateBooks=true` — enables the auto-loader
- `AutoLoadRateBooksLocation=/config/content` — tells it where to scan for rate book XMLs (covers `cust-ratebooks/<LOB>/`)

### 5. Runtime Property Config Location

**Check** `config/config.properties` (or `config.local.properties` depending on the project) for:
```
plugin.PLUGIN_RUNTIME_PROPERTY_CONFIG_LOCATION=runtimeproperties/local
```

If this line is missing, the runtime properties XML files are never loaded, and the auto-loader will say **"DISABLED"** at startup. Add it to whichever config properties file the project uses.

**The path is relative to the server's working directory, which is `modules/configuration`.** You can confirm that because H2's `./tmp/pc` lands in `modules/configuration/tmp`. Do NOT use `modules/configuration/runtimeproperties/local`: it resolves to a folder that doesn't exist, the plugin skips it **silently**, and you get DISABLED with `RawValueShouldAutomaticallyLoadRateBooks = null`. This happened in CR7 in Quasar 2026-10, and IM has the same wrong value.
**Verify** that the startup log has 2 "Reading runtime properties from" lines: the OOTB `config\runtimeproperties\RuntimeProperties.xml` and the external `runtimeproperties\local\`. Then look for "on server startup ENABLED".

### 6. StructuredLogger.RATEFLOW

The auto-loader's `LoggerWrapper.gs` uses `StructuredLogger.RATEFLOW`. If the Target OOTB's `StructuredLogger.gs` is missing the `RATEFLOW` static var, add it:

```gosu
uses gw.api.system.PCLoggerCategory
// ...
public static final var RATEFLOW : StructuredLogger = new StructuredLogger(PCLoggerCategory.RATEFLOW)
```

`PCLoggerCategory.RATEFLOW` is a platform class available in both Niseko and Quasar.

### 7. Display Keys

Add to `config/locale/display.properties`:
```
SP.Rating.AutoRateBookLoad.Error.CannotFindZIPFile=Unable to find any entry in ZIP file\: {0}
SP.Rating.AutoRateBookLoad.Error.CannotImportNestedDirectory=ZIP file entry name < {0} > is a DIRECTORY\!  In ZIP file\: {1}
SP.Rating.AutoRateBookLoad.Error.CannotUploadMultiEntryZIP=There was an error importing data\: import zip file can only contain one file
SP.Rating.AutoRateBookLoad.UpdateAllFromRelease=Update All From Release
```

## Making Rate Books Active on Import

By default, rate books may be exported as `<Status>Stage</Status>`. Stage rate books will import but won't be used by the rating engine if `IRatingPlugin.gwp` has `RatingLevel=Approved`.

**Fix:** Change all rate book XML files to `<Status>Active</Status>`:
```bash
sed -i 's|<Status>Stage</Status>|<Status>Active</Status>|g' config/content/cust-ratebooks/<LOB>/*.xml
```

This makes them immediately usable after import. No need to change `IRatingPlugin.gwp` since Active satisfies any RatingLevel setting.

## Verification

On server startup, look for these log lines in order:

1. `"Starting AutomaticRateBookLoadOnSystemStartupPlugin."` — plugin started
2. `"Rating:  Automatically loading Rate Books on server startup ENABLED."` — all conditions passed
3. `"Automatic rate book load results: ..."` — success with count of imported books

If you see **`"Automatic rate book loading is DISABLED."`**, the detailed log above it shows which condition failed:
- `IsProductionServer` — must be false (dev mode)
- `RawValueShouldAutomaticallyLoadRateBooks` — must be true (from runtime property XML)
- `AutomaticallyLoadRateBooks` (AutoLoad param) — must be Yes (from .gwp)
- `AutoLoadRateBooksLocation` — must not be null (from runtime property XML)
- `RatingLevel` parameter — must be Stage or Draft

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `does not contain table <LOB>CovToRateRoutineCodeLookup_Ext` | Rate books not imported into DB | Complete all 7 steps above |
| `Automatic rate book loading is DISABLED` | Runtime properties not loaded | Check step 5 — `PLUGIN_RUNTIME_PROPERTY_CONFIG_LOCATION` missing |
| `No static property descriptor found for RATEFLOW` | `StructuredLogger.gs` missing RATEFLOW | Apply step 6 |
| `Nonexistent display key SP.Rating.AutoRateBookLoad.*` | Display keys not added | Apply step 7 |
| Rate books imported but quoting still fails | Books imported as Stage, plugin requires Approved | Change XML status to Active (see "Making Rate Books Active") |
| H2 still has old data | DB not reset after adding autoloader | Delete H2 and restart |
