<h1 align="center">Cabinet 1.0.0</h1>

<p align="center">Data file management and caching for GameMaker</p>

<p align="center">by <b>@shdwcat</b></p>

<p align="center">Using <a href="https://github.com/JujuAdams/Gumshoe">gumshoe</a> by <b>@jujuadams</b> and <b>@nkrapivin</b></p>

<p align="center">Chat about Cabinet on the <a href="https://discord.gg/8krYCqr">Discord server</a></p>

<hr/>

## Description

Cabinet exists to help manage access to data files in GameMaker, including features like caching the contents of a file so that you only have to read it once, as well as being able to convert the raw file contents into something else useful, like converting a JSON file directly into a struct, or decoding binary data. Once you setup a Cabinet, you can simply access the contents of the files you want, without having to manage the reading or caching yourself.

## Table of Contents
- [Compatibility](#compatibility)
- [Basic Usage](#basic-usage)
- [Advanced Usage](#advanced-usage)
- [Documentation](#documentation)
  - [`Cabinet`](#cabinet)
  - [`CabinetFile`](#cabinetfile)
  - [Options](#cabinet-options)

## Compatibility

Cabinet was made with GameMaker 2022.3. It may be compatible with earlier versions of GameMaker (possibly with some manual tweaking).

## Basic Usage

To get started, you'll create a Cabinet for a folder path. You can specify a specific file extension, in which case only files with that extension will be included in the cabinet. And you can also specify some more advanced options (see [Cabinet Options](#cabinet-options) below)

```gml
var cabinet = new Cabinet(folder_path, [extension], [options])
```

Once you've created the Cabinet, you can now inspect the `tree` variable, which contains a tree struct describing folders and files within the `folder_path` for the Cabinet, and the `flat_map` variable, which is a flat struct of just the files. The *values* in the `tree` and the `flat_map` are [`CabinetFile`](#cabinetfile) structs which provide access to further details and functionality.

For example, you could get the `CabinetFile` for the first file that was found like so:

```gml
var first_path = cabinet.file_list[0];
var cabinet_file = cabinet.file(first_path);
```

And then read the contents with:
```gml
var contents = cabinet_file.tryRead();
```

Caching is enabled by default, so the first time you call `tryRead()` the file will be read from disk, but the *second* time you call it, `tryRead()` will returned the value from the cache that was stored after the first call. This can be helpful if you find yourself needing to read the same file from different places in code, without being sure if you've loaded it yet.

## Advanced Usage

Simply caching the file contents is nice, but if you're working with anything more complex than simple text, you're going to want to turn that text into *game data* that you can use.

To get Cabinet to do this automatically, you'll want to specify a `file_value_generator` in the [Cabinet Options](#cabinet-options):

```gml
// passing "" as the folder_path will scan all files in the /datafiles folder, as it is the default directory when reading files in GameMaker
var json_cabinet = new Cabinet("", ".json", {
	file_value_generator: function(text, cabinet_file) {
		var data = json_parse(text);
		return data;
	},
});
```

Now our `json_cabinet` will *automatically* parse the raw JSON text of any file into a struct with `json_parse()`, when reading the file. This means that when we ask for the file contents, we'll get that struct instead of a string:

```gml
var json_data = cabinet_file.tryRead();
// json_data will be the struct that was parsed by json_parse() in the file_value_generator function provided in the options
```

Just getting the struct is nice, but you could also then pass that data to a GML `constructor` function:

```gml
	file_value_generator: function(text, cabinet_file) {
		var data = json_parse(text);
		var enemy_definition = new EnemyDefinition(data);
		return enemy_definition;
	},
```

Remember, when caching is enabled, the value will only be generated *once*. The `file_value_generator` function will run the *first* time the data is read from disk, and after that `.tryRead()` will return the result of that function, which is what was stored in the cache.

## Documentation

### `Cabinet`
```gml
function Cabinet(folder_path, extension = ".*", options = undefined) constructor
```
#### Parameters
- `folder_path` (string) The path of the folder that Cabinet should scan for files
- `extension` (string, optional) The extension of the file type that should be included in the cabinet.
- `options` (struct, optional) The options to use for this cabinet. See [Cabinet Options](#cabinet-options).

#### Variables
- `tree` (struct) Tree structure corresponding to the folders and files that were found in scanned folder. The keys are the names of the folders/files. The value for a folder key will be a struct with more folders and files within it. The value for a file key will be a [CabinetFile](#cabinetfile) providing data and functions for the actual file on disk.
- `flat_map` (struct) Similar to the `tree` variable, except the keys will be the full paths of every file in the `Cabinet`, and each value will be the corresponding `CabinetFile`.
- `file_list` (string array) A flat array with the full path of every file in the `Cabinet`.

#### Functions
- `file(path)` - Returns the [CabinetFile](#cabinetfile) for the `path` if it exists in the `Cabinet`, otherwise `undefined` will be returned.
- `readFile(path)` - Returns the *content* of the file at the `path` if it exists in the `Cabinet`, otherwise `undefined` will be returned (the *content* can be customized by providing a `file_value_generator` in the [Cabinet Options](#cabinet-options)).
- `clearCache()` - Clears all cached data for the `Cabinet`
- `rescan()` - Rebuilds all of the Variables for this `Cabinet` by scanning the `folder_path` again.

### `CabinetFile`

You don't need to create `CabinetFiles` yourself as they are created automatically by a [Cabinet](#cabinet).

#### Variables
- `fullpath` (string) the full path to the associated file
- `directory` (string) the path of the directory containing the file
- `file` (string) the name of the file, including the extension
- `extension` (string) just the extension of the file
- `file_id` (string) the name of the file, *without* the extension
- `scan_time` (datetime) when the file was scanned by the `Cabinet`, either on creation or by calling `rescan()`
- `read_time` (datetime) the time when the file was read and cached, or `undefined` if it has not been read and cached yet

#### Functions
- `tryRead()` - Returns the *content* of the associated file (the *content* can be customized by providing a `file_value_generator`). If the file has been cached, the cached value will be returned, otherwise, the file will be read from disk (and possibly parsed), if it still exists on disk, otherwise `undefined` will be returned.
- `tryLoad()` - If the file exists on disk, it will be read, possibly parsed, cached and returned. Otherwise `undefined` will be returned. Note: The file will be cached even if the `cache-reads` option is false.
- `tryScanLines(match_line)`- (Advanced) This function will read the associated text one line at a time, and call the provided `match_line(line)` function. If the `match_line` function returns a value, that value will be returned by `tryScanLines()`, otherwise the next line will be read. If no line was matched, `undefined` will be returned.
  - NOTE: This function is only valid when the `read_mode` option is set to `"string"`. See [`Cabinet Options`](#cabinet-options).
  - TIP: This function can be useful for quickly reading 'header' information from a long file without actually needing to read the whole file into memory.

### Cabinet Options

When calling the [`Cabinet`](#cabinet) constructor, you can pass a struct specifying the following options to use for that `Cabinet`.

- `cache_reads` (boolean, default: `true`) - Whether to cache the contents of a file after reading it from disk.

- `read_mode` (`"string"` or `"binary"`, default: `"string"`) - Whether to read the raw file content as a `string` or as a binary `buffer`.
  - This affects the type of the first parameter in the `file_value_generator` function option.

- `file_value_generator` (`function(content, cabinet_file)`, default: `undefined`) - Function to convert the raw file content to a custom value of your choice (and implementation).
   - When `read_mode` is `"string"` the `content` parameter of the function will be the file content as a string.
   - When `read_mode` is `"binary"` the `content` parameter of the function will be the file content as a gml `buffer`.
   - The [`CabinetFile`](#cabinetfile) is provided as the second parameter of the function, in case any of the information in it is useful.

- `cabinet_file_customizer` (`function(cabinet_file)`, default: `undefined`) - Function that is called when the [`Cabinet`](#cabinet) scans the `folder_path` for files and creates [`CabinetFiles`](#cabinetfile).
  - Here you can set additional values on the `cabinet_file` parameter as may be helpful. For example, you might want to call `tryScanLines()` to read header information from the file and store that data on the `cabinet_file` for future reference.

## Credits

Cabinet utilizes the [gumshoe](https://github.com/JujuAdams/Gumshoe) library by <b>@jujuadams</b> to scan files on disk, and therefore a version of that code is included in this project. If you're already using gumshoe in your project, make sure to replace it with the Cabinet version when importing the Cabinet package.