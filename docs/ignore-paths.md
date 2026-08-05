# Ignoring paths

Rojo has two ways to exclude files on disk from the instance tree: a
project-wide setting and a per-node setting.

## `globIgnorePaths` (project-wide)

An array of globs at the top level of a project file. Matching files and
directories are excluded everywhere in the project. Globs are evaluated
relative to the folder containing the project file.

```json
{
  "name": "MyGame",
  "tree": { "$path": "src" },
  "globIgnorePaths": ["**/*.spec.lua"]
}
```

## `$ignorePaths` (per-node)

An array of globs on a single node in the `tree`. Matching files and
directories are excluded from **that node only**. Globs are evaluated
relative to the node's `$path` target, so patterns don't need to repeat the
`$path` prefix.

```json
{
  "tree": {
    "$path": "src",
    "$ignorePaths": ["**/*.spec.lua"]
  }
}
```

Rules apply recursively: they filter the target directory and every
subdirectory beneath it, including directories brought in through nested
`*.project.json` files. Other nodes in the project — even ones whose `$path`
points inside the same directory — are unaffected.

`$ignorePaths` requires `$path`; on a node without `$path` it is ignored
with a warning.

### Splitting one folder across multiple tree locations

Because filtering is per-node, two nodes can point at the **same** directory
with complementary filters. This lets you keep a feature's server, client,
and shared code together on disk while routing it to different services in
Studio:

```json
{
  "tree": {
    "$className": "DataModel",
    "ServerScriptService": {
      "Features": {
        "$path": "src/features",
        "$ignorePaths": ["**/client/**", "**/shared/**"]
      }
    },
    "ReplicatedStorage": {
      "Features": {
        "$path": "src/features",
        "$ignorePaths": ["**/server/**"]
      }
    }
  }
}
```

With a layout like `src/features/chat/{server,client,shared}`, each
feature's `server` folder appears only under ServerScriptService and its
`client`/`shared` folders only under ReplicatedStorage. New feature folders
are picked up automatically — the globs match at any depth.

Files matched by *neither* node's globs appear in **both** locations (this
is Rojo's existing path-aliasing behavior). Live sync handles the split
correctly: adding or editing a file only patches the location(s) whose
filters include it.

### Empty folders are pruned

Folders left with no children after filtering are removed from the tree,
bottom-up: a folder whose only contents were pruned folders is pruned as
well. This means `"server"` (exclude the directory) and `"server/**"`
(exclude its contents) produce the same result under a node with
`$ignorePaths`.

Folders that carry meaning beyond their contents are kept: a folder whose
`init.meta.json` sets a `className`, properties, or an `id` is never
pruned, and the node's own `$path` target always exists since it is
declared in the project.

Note that pruning only applies to nodes using `$ignorePaths`; the
project-wide `globIgnorePaths` setting keeps its existing behavior of
leaving emptied folders in place.

One live-sync quirk: pruning is applied when a directory is snapshotted, so
deleting the last synced file in a folder removes the file's instance
immediately but leaves the now-empty folder in Studio until the next change
under that node re-snapshots it.

## Negation

Both settings support gitignore-style negation. Rules are evaluated in
order with last-match-wins; a `!` prefix re-includes a path that an earlier
rule excluded. Escape a literal leading `!` as `\!`.

```json
"$ignorePaths": ["**/*.spec.lua", "!keep.spec.lua"]
```

## Limitations

- `$ignorePaths` only filters snapshotting (disk → Studio). `rojo syncback`
  does not use it to route new instances: an instance added in Studio under
  a split location is written into the shared directory, then appears under
  whichever node(s) don't ignore it.
- Filtering applies to directory targets. A `$path` pointing directly at a
  file has nothing to filter.
