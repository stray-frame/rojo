# ignore_paths_split_folder

Ensures that two project nodes can point at the same directory with
complementary `$ignorePaths` globs, splitting one folder on disk across
multiple locations in the instance tree. Shared files appear in both
locations; filtered files and directories appear only under the node that
does not ignore them.
