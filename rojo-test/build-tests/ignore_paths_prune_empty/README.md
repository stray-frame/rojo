# ignore_paths_prune_empty

Ensures that folders left empty because `$ignorePaths` filtered out all of
their contents are pruned from the tree instead of syncing as empty
Folder instances. Pruning cascades bottom-up: a folder whose only contents
were pruned folders is pruned as well.
