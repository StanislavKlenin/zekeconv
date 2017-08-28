# Zekeconv

Command-line tool to convert between [zeke](https://github.com/yodle/zeke) dumps and directory trees.

## Installation
```sh
pip install zekeconv
```

## Usage

Note: Zekeconv only operates on files and directory structures. It does not directly communicate with Zookeeper, all interactions with Zookeeper in the examples below are being done with the help of [zeke](https://github.com/yodle/zeke) utility. Refer to its documentation for server-related options (such as [instance location](https://github.com/yodle/zeke#specify-the-location-of-zookeeper) and [DNS discovery](https://github.com/yodle/zeke#discover-zookeeper-via-dns)).

### Unpack: convert dump file to a directory tree

```sh
$ zekeconv --file dumpfile.zk --unpack /path/to/dir
```

### Pipe output from zeke and unpack it
```sh
$ zeke --dump "/some/origin/node" | zekeconv --file - --unpack /path/to/dir
```
This will create a directory tree starting with `some` under `/path/to/dir`. resulting in `/path/to/dir/some/origin/node`.

To strip that common path prefix, let zekeconv know what was the origin of the dump:
```sh
$ zeke --dump "/some/origin/node" | zekeconv --file - --unpack /path/to/dir --source "/some/origin/node"
```
This will result in _sub-nodes_ of `/some/origin/node` being extracted to `/path/to/dir`

### Pack: convert previously unpacked directory tree to zeke dump file
```sh
$ zekeconv --pack /path/to/dir --file outfile.zk
```
Prepend all keys with a common prefix (for example, if the tree has been unpacked from this node with common prefix stripped):
```sh
$ zekeconv --pack /path/to/dir --source "/some/origin/node" --file outfile.zk
```

### Pack directory tree and pipe it to zeke
```sh
$ zekeconv --pack /path/to/dir --file - | zeke --load
```

## Advanced usage

### Copy Zookeeper sub-tree to another node

```sh
$ zeke --dump "/some/origin/node" | zekeconv --file - --unpack /path/to/dir --source "/some/origin/node"
$ zekeconv --pack /path/to/dir --source "/another/node" --file - | zeke -l
```
The first command above creates a directory structure consisting of all sub-nodes of `/some/origin/node` under `/path/to/dir/`. The second command loads this structure, prepends it with another prefix and pipes it to zeke,
effectively copying this structure to `/another/node`, recursively.

## Model

Zookeeper configuration tree is similar to a file system, but with one essential difference: whereas filesystem node can be either a file (containing some data) or a directory (containing sub-nodes), a Zookeeper node has properties
of both. To reconcile Zookeeper tree structure with filesystem representation, the following model has been chosen:

* Every Zookeeper key becomes a directory
* If there is a value defined for the key, a single file is created in that directory

A direct consequence of these rules: directory names correspond to Zookeeper key names, but _file names do not encode any information_; only the presence of file is essential to specify that given key has a value. Also, there must
be at most one file in each directory (zekeconv will fail if more than one file encountered).

For example, this node:
```
parent = foo
├── child1
└── child2 = bar
```
will map into the following directory structure
```
parent/
├── child1/
├── child2/
│   └── parent-child2.txt (file with value bar)
└── parent.txt (file with value foo)
```
Note that:

* `parent` directory has three descendants; only one of them is a file, others are sub-directories
* `child1` directory has no descendants because it has no value and no sub-nodes in Zookeeper
* `child2` has only one descendant, a file with its value

