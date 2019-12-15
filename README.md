# Solid-File-Client

A library for managing Solid files and folders.

current version : <a href="http://badge.fury.io/js/solid-file-client@1.0.0">![npm](https://badge.fury.io/js/solid-file-client@1.0.0.svg)</a>
<br>
previous version : <a href="http://badge.fury.io/js/solid-file-client">![npm](https://badge.fury.io/js/solid-file-client.svg)</a>


## Introduction

Solid-File-Client is a JavaScript library with high-level methods to create, read, and manage files and folders.  The methods may be used in browser scripts or in nodejs shell scripts.  The library supports both text and binary files and can read and write data from Solid Pods, from local file systems, and from virtual in-memory storage.  It can also recursively move and copy entire folder trees between any of those storage locations. For advanced users, there are also a number of options and lower-level methods which allow fine-tuned control over linked resources and other features of Solid. 

### Note for users of version 0.x of Solid-File-Client

There are a number of changes which are not backward compatible.  See [TBD : Guide for transitioning to v.1](docs/transition-to-v.1.md) for details and hints for upgrading.

### Installing, invoking, and logging-in

See [TBD : Using in a Browser](docs/using-in-a-browser.md) or [TBD : Using in Node](docs/using-in-node.md) for details of how to install and invoke this libary and how to log in and manage sessions once you have invoked it.

### Error Handling

[TBD : Error Handling](docs/error-handling.md)

### Using alternate storage spaces

Solid-file-client can work with web resources (https://).  It can also work
with resources stored in a browser's local storage (app://), or a local file system (file://).  See the [Upload Demo](examples/upload/index.html) for an example of copying files from a local filesystem to a pod in the browser.  See [Solid-Rest](https://github.com/jeff-zucker/solid-rest) for a description of using browser local storage or accessing the local file system from node scripts.

### Overview of writing methods

By default, all high-level methods that create, copy, or move files or folders have these behaviors :

  * the source always completely overwrites the target
  * if the path to the item doesn't pre-exist, it will be created
  * linked files (.acl and .meta) are copied/moved along with their resources

For many purposes, these defaults will suffice.  However, if you need to, you may change any of them with option flags.  There are several options for merging folder trees as well as for using Solid's POST features. See the sections on [Overwriting](#overwriting), on [Creating Paths](#creating-paths), and on [Linked Files](#linked-files) for more information.

## High-level Methods

### createFile( fileURL, content, contentType, options )

Creates a new file at the specified URL.  Content is required, even if only a blank string.  ContentType should be something like "text/turtle" or "image/png" and is required.

Default behavior :
  * If a file already exists at that URL, it will be overwritten.
  * If the file's parent path does not exist, it will be created.

See [Overwriting](#overwriting) and [Creating Paths](#creating-paths) if you need to change the default behavior.

Note for advanced users : This method uses PUT, if you prefer the behavior of 
PODT (for example creating alternate versions of a resource rather than replacing it) use the post() method, see [Low-level Methods](#low-level-methods).


### <a name="createFolder">createFolder( folderURL, options )</a>

Creates a new folder at the specified URL.  

Default behavior :
  * If a folder already exists at that URL, it will be overwritten and all of its contents lost.
  * If the folder's parent path does not exist, it will be created.

See [Overwriting](#overwriting) and [Creating Paths](#creating-paths) to change the default behavior.


### readFile( fileURL, options )

On success, the readFile() method returns the contents of the specified file.
The return value will be a string for text files and a blob for binary files
such as images and music.  

In some cases Solid servers may offer multiple versions of a resource (for example Turtle and JSON-LD representations of the same file).  If you need to specify a given version, see [Working with Accept headers](#accept-headers).

If you want the content as a ReadableStream, use the get() method - see [Low-level Methods](#low-level).

### readFolder( folderURL, options )

On success, the readFolder() method returns a folder object in this format:
```javascript
{
     type : "folder",
     name : // folder name (without path),
      url : // full URL of the resource,
 modified : // dcterms:modified date
    mtime : // stat:mtime
     size : // stat:size
   parent : // parentFolder or undef if none,
  content : // raw content of the folder's turtle representation,
    files : // an array of files in the folder
  folders : // an array of sub-folders in the folder,
}
```
Each item in the arrays of files and sub-folders will be a file object which is the same as a folder object except it does not have the last two fields (files,folders). The content-type in this case is not guessed, it is read from the folder's triples, i.e. what the server sends.

By default, readFolder() does not list linked resources (.acl and .meta files).  To change this behavior, see [Linked files](#linked-files).

### readHead( folderOrFileURL )

Returns the header for a file.  The file content's are not returned.  

### itemExists( fileOrFolderURL )

Returns true if the URL exists and false otherwise.

### patch()

[TBD: document patch() as high-level?] 

### deleteFile( fileURL, options )

Deletes the specified file and all linked filed (.acl,.meta)

### deleteFolder( folderURL )

Recursively deletes a folder and all of its contents including all linked files (.acl, .meta).

### moveFile( sourceURL, targetURL, options ), copyFile( sourceURL, targetURL, options )

Copies or moves the specified source to the target.

Defaults :
  * If a file already exists at that target, it will be overwritten.
  * If the target URL's parent path does not exist, it will be created.
  * Linked files (.acl and .meta) will be copied if they exist.

See [Advanced Options] to modify these default behaviors.

### moveFolder(sourceURL,targetURL,options), copyFolder(sourceURL,targetURL,options)

Recursively copies or moves a folder and all of its contents.

Defaults :
  * If the target exists, it is replaced by the source.
  * If the top target folder's parent path does not exist, it will be created.
  * Linked files (.acl and .meta) will be copied/moved if they exist.

These default behaviors may all be modified.  For example, you can choose from several ways to merge the source and target folders.  See [Advanced Options](#advanced-options) for more details.


## Advanced Options

### <a name="overwriting">Overwriting</a>

By default, methods which create, copy, or move files or folders will overwrite an item of the same name in the target space, replacing it with the source item.  This behavior may be modified in several ways

With any of the create/copy/move methods, you can use the [itemExists()](#itemExists) method to prevent overwriting items.
```javascript
    if( !itemExists(x) ) {
        createFolder(x) // only create if it doesn't already exist
    }
 ```
With the **copyFolder()** and **moveFolder()** methods, you can elect to merge the source and target with preference for the source or preference for the target:
 
   * **default** - target is replaced by source
   * **merge=source** - target becomes source plus items found only in target 
   * **merge=target** - target becomes target plus items found only in source

### <a name="creating-paths">Creating Paths</a>

When you create a file or folder and the path to that item doesn't already exist, Solid-File-Client will create the path for you if it can.  For example, if you ask to create /foo/bar/baz.txt but there is no /foo/ folder and there is no /bar/ folder, Solid-File-Client will create /foo/ and then create /bar/ inside it and then create baz.txt inside that.

If you would rather the program fails if the path you asked for doesn't exist, you may set the "createPath" flag to false.

  * **createFile(),createFolder(),copyFile(),copyFolder(),moveFile(),moveFolder()**
  
      * **default** - create intermediary paths if they are missing
      * **createPath=false** - fail if intermediary paths are missing
      
  * **note** for copyFolder() and moveFolder(), the createPath option applies only to the top-level target folder, not to folders within the target which are handled by the merge option      

### <a name="linked-files">Linked Files</a>

One of Solid's unique features is the use of linked files.  Currently the two
main types of linked files are .acl files which control access to the main
file they are linked from and .meta files which describe additional features
of the file they are linked from.  Solid-file-client, by default treats these
linked files as tightly bound to the resource they are referencing.  In other
words when you delete, move, or copy a file that has linked files, the linked
files will also be deleted, moved, or copied.  These defaults should be
sufficient for most basic usage.  

Solid-file-client makes a special case for access control (.acl) files.  These
files may contain absolute links which will no longer work when the file is 
copied or moved.  So solid-file-client, by default, will modify .acl files to change absolute links to relative ones.  

Solid servers provide the possible location of linked resources in the headers of all resources.  Solid-file-client supports the links=includePossible option to include these possible locations without checking to see if the linked file actually exists. The possible locations tell you where to create the linked file if they don't already exist.

Advanced users can modify how linked files are handled with the "links" 
option flags shown below.

  * **copyFile(),copyFolder(),moveFile(),moveFolder()**

      * **default**        - linked items are copied/moved, acl is modified
      * **links=exclude**  - linked items are not copied/moved
      * **links=noModify** - linked items are copied/moved, acl is not modified

  *  **readFolder()**

      * **default**               - linked items are not listed
      * **links=include**         - linked items are listed, when they exist
      * **links=includePossible** - possible locations of links are listed

See also, the **getPossibleLinks()** method which finds the possible locations of linked resources for an item.  See [Low-level Methods](#low-level-methods)

### Working with Accept headers

[TBD : working with accept headers]

### <a name="low-level-methods">Low-level methods</a>

Solid-File-Client provides a number of low-level methods which directly reflect the behavior of the Solid server without additional processes as are found in the high-level methods.  

The low-level methods are :

  fetch, get, put, post, delete, options, head, patch

See the [JSdoc for the API](docs/JSdoc/api.md) for more details of these methods.

## Note on terminology

Solid servers can store data directly in a physical file system,  or use a database or other storage.  Instead of talking about "Files" and "Folders", it is more correct to talk about "Containers" and "Resources" - logical terms independent of the storage mechanism.  In this documentation, for simplicity, we've used the file/folder terminology with no implication that it represents a physical file system.

## Contributors

This library was originally authored by **Jeff Zucker**. Version 1.0.0 includes many additions and improvements that were the results of a collaboration between Jeff, **Alain Bourgeois**, and **Otto [TBD : name?]**.

## Acknowledgements

Many thanks for patches and issues from https://github.com/linonetwo, and https://github.com/scenaristeur.

copyright (c) 2018 Jeff Zucker may be freely used with MIT license

