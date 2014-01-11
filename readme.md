# `mock-fs`

A configurable mock file system.  You know, for testing.

## Example

The code below makes it so the `fs` module is temporarily backed by a mock file system with a few files and directories.

```js
var mock = require('mock-fs');

mock({
  'path/to/fake/dir': {
    'some-file.txt': 'file content here',
    'empty-dir': {/** empty directory */}
  },
  'path/to/some.png': new Buffer([8, 6, 7, 5, 3, 0, 9]),
  'some/other/path': {/** another empty directory */}
});
```

When you are ready to restore the `fs` module (so that it is backed by your real file system), call `mock.restore()`.

```js
// after a test runs
mock.restore();
```

## Docs

### `mock(config)`

Configure the `fs` module so it is backed by an in-memory file system.

Calling `mock` sets up a mock file system with at least two directories: `process.cwd()` and `os.tmpdir()` (or `os.tmpDir()` for older Node).  When called with no arguments, just these two directories are created.  When called with a `config` object, additional files, directories, and symlinks are created.

Property names of the `config` object are interpreted as relative paths to resources (relative from `process.cwd()`).  Property values of the `config` object are interpreted as content or configuration for the generated resources.

*Note that paths should always use forward slashes (`/`) - even on Windows.*

#### Creating files

When `config` property values are a `string` or `Buffer`, a file is created with the provided content.  For example, the following configuration creates a single file with string content (in addition to the two default directories).
```js
mock({
  'path/to/file.txt': 'file content here'
});
```

#### Creating directories

When `config` property values are an `Object`, a directory is created.  The structure of the object is the same as the `config` object itself.  So an empty directory can be created with a simple object literal (`{}`).  The following configuration creates a directory containing two files (in addition to the two default directories):
```js
// note that this could also be written as
// mock({'path/to/dir': { /** config */ }})
mock({
  path: {
    to: {
      dir: {
        file1: 'text content',
        file2: new Buffer([1, 2, 3, 4])
      }
    }
  }
});

#### Providing additional file and directory properties

To create files or directories with additional properties (custom permissions, atime, uid, etc.), use the `mock.file()` and `mock.directory()` factories below.

#### Creating symlinks

Using a `string` or a `Buffer` is a shortcut for creating files with default properties.  Using an `Object` is a shortcut for creating a directory with default properties.  There is no shortcut for creating symlinks.  To create a symlink, you need to call the `mock.symlink()` function described below.

### `mock.file(properties)`

Create a factory for new files.  Supported properties:

 * **content** - `string|Buffer` File contents.
 * **mode** - `number` File mode (permission and sticky bits).  Defaults to `0666`.
 * **uid** - `number` The user id.  Defaults to `process.getuid()`.
 * **git** - `number` The group id.  Defaults to `process.getgid()`.
 * **atime** - `Date` The last file access time.  Defaults to `Date.now()`.  Updated when file contents are accessed.
 * **ctime** - `Date` The last file change time.  Defaults to `Date.now()`.  Updated when file owner or permissions change.
 * **mtime** - `Date` The last file modification time.  Defaults to `Date.now()`.  Updated when file contents change.

To create a mock filesystem with a very old file named `foo`, you could do something like this:
```js
mock({
  foo: mock.file({
    content: 'file content here',
    ctime: new Date(1),
    mtime: new Date(1)
  })
});
```

Note that if you want to create a file with the default properties, you can provide a `string` or `Buffer` directly instead of calling `mock.file()`.

### `mock.directory(properties)`

Create a factory for new directories.  Supported properties:

 * **mode** - `number` Directory mode (permission and sticky bits).  Defaults to `0777`.
 * **uid** - `number` The user id.  Defaults to `process.getuid()`.
 * **git** - `number` The group id.  Defaults to `process.getgid()`.
 * **atime** - `Date` The last directory access time.  Defaults to `Date.now()`.
 * **ctime** - `Date` The last directory change time.  Defaults to `Date.now()`.  Updated when owner or permissions change.
 * **mtime** - `Date` The last directory modification time.  Defaults to `Date.now()`.
 * **items** - `Object` Directory contents.  Members will generate additional files, directories, or symlinks.

To create a mock filesystem with a directory with the relative path `some/dir` that has a mode of `0755` and a couple child files, you could do something like this:
```js
mock({
  'some/dir': mock.directory({
    mode: 0755,
    items: {
      file1: 'file one content',
      file2: new Buffer([8, 6, 7, 5, 3, 0, 9])
    }
  })
});
```

Note that if you want to create a directory with the default properties, you can provide an `Object` directly instead of calling `mock.directory()`.


### `mock.symlink(properties)`

Create a factory for new symlinks.  Supported properties:

 * **path** - `string` Path to the source (required).
 * **mode** - `number` Symlink mode (permission and sticky bits).  Defaults to `0666`.
 * **uid** - `number` The user id.  Defaults to `process.getuid()`.
 * **git** - `number` The group id.  Defaults to `process.getgid()`.
 * **atime** - `Date` The last symlink access time.  Defaults to `Date.now()`.
 * **ctime** - `Date` The last symlink change time.  Defaults to `Date.now()`.
 * **mtime** - `Date` The last symlink modification time.  Defaults to `Date.now()`.

To create a mock filesystem with a file and a symlink, you could do something like this:
```js
mock({
  'some/dir': {
    'regular-file': 'file contents',
    'a-symlink': mock.symlink({
      path: 'regular-file'
    })
  }
});
```

## Install

Using `npm`:

```
npm install mock-fs --save-dev
```

## Caveats

When you require `mock-fs`, Node's own `fs` module is patched to allow the binding to the underlying file system to be swapped out.  If you require `mock-fs` *before* any other modules that modify `fs` (e.g. `graceful-fs`), the mock should behave as expected.

The following `fs` functions are overridden: `fs.ReadStream`, `fs.Stats`, `fs.WriteStream`, `fs.appendFile`, `fs.appendFileSync`, `fs.chmod`, `fs.chmodSync`, `fs.chown`, `fs.chownSync`, `fs.close`, `fs.closeSync`, `fs.createReadStream`, `fs.createWriteStream`, `fs.exists`, `fs.existsSync`, `fs.fchmod`, `fs.fchmodSync`, `fs.fchown`, `fs.fchownSync`, `fs.fdatasync`, `fs.fdatasyncSync`, `fs.fstat`, `fs.fstatSync`, `fs.fsync`, `fs.fsyncSync`, `fs.ftruncate`, `fs.ftruncateSync`, `fs.futimes`, `fs.futimesSync`, `fs.lchmod`, `fs.lchmodSync`, `fs.lchown`, `fs.lchownSync`, `fs.link`, `fs.linkSync`, `fs.lstatSync`, `fs.lstat`, `fs.mkdir`, `fs.mkdirSync`, `fs.open`, `fs.openSync`, `fs.read`, `fs.readSync`, `fs.readFile`, `fs.readFileSync`, `fs.readdir`, `fs.readdirSync`, `fs.readlink`, `fs.readlinkSync`, `fs.realpath`, `fs.realpathSync`, `fs.rename`, `fs.renameSync`, `fs.rmdir`, `fs.rmdirSync`, `fs.stat`, `fs.statSync`, `fs.symlink`, `fs.symlinkSync`, `fs.truncate`, `fs.truncateSync`, `fs.unlink`, `fs.unlinkSync`, `fs.utimes`, `fs.utimesSync`, `fs.write`, `fs.writeSync`, `fs.writeFile`, and `fs.writeFileSync`.

Mock `fs.Stats` objects have the following properties: `dev`, `ino`, `nlink`, `mode`, `size`, `rdev`, `blksize`, `blocks`, `atime`, `ctime`, `mtime`, `uid`, and `gid`.  In addition, all of the `is*()` method are provided (e.g. `isDirectory()`, `isFile()`, et al.).

The following `fs` functions are *not* currently mocked (if your tests use these, they will work against the real file system): `fs.FSWatcher`, `fs.unwatchFile`, `fs.watch`, and `fs.watchFile`.  Pull requests welcome.

[![Current Status](https://secure.travis-ci.org/tschaub/mock-fs.png?branch=master)](https://travis-ci.org/tschaub/mock-fs)
