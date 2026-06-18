[[File Handling]]
Before java 7, dev used 
`File file = new File("data.txt")`
the `File` class has two responsibilties
1.  Representing a path
2. Performing file operations

With NIO.2(java 7), Java separated these concerns

`Path path = Path.of("data.txt");`
represents a location

`Files.readString(path);`
performs operation on that location.

This follows a cleaner design principle
```
Path --> What/Where
Files --> Action
```


### Path
A `Path` is an object that represents a sequence of directories and file names that identify a location in a file system.

```
Path path = Path.of("uploads/images/profile.png");
```

This object just stores the path.
it does not create a file, directory, check existence or read contents.

the path structure looks like
```
	uploads
	   ↓
	images   --> Path element
	   ↓
	profile.jpg
```

Each component is called path element
the last element may be a file or directory. Path itself doesn't know or care

#### Relative Path
`Path path = Path.of("data.txt");`

This means:
Look for data.txt relative to the current working directory/
if current directory -> `C:/projects/devsync`
then the abs path of file will be -> `c:/projects/devsync/data.txt`

#### Absolute Path
An absolute path starts from the file system root.
Windows -> `Path.of("C:/uploads/profile.png");`
Linux -> `Path.of("/home/user/profile.jpg");`
These path are independent of current working directory.

> Relative paths are preferred in Application

#### Methods
```
Path path = Path.of("uploads/images/profile.png");
```
##### getFileName()  -> returns the last element of the path
path.getFileName() -> profile.jpg

##### getParent() -> returns the path immediately above the current path
path.getParent() -> uploads/images

##### getRoute() -> returns the filesystem root from which the path begins

Windows  -`Path.of("C:/uploads/file.txt").getRoot();` --> C:\
Linux -   `Path.of("/home/user/file.txt").getRoot();` --> /
for relative paths root is null

##### toAbsolutePath() --> Converts a relative path into its complete filesystem location
```
Path path = Path.of("data.txt");
System.out.println(path.toAbsolutePath());
```
suppose current dir -->  `C:/projects/devsync`
Output --> `C:/projects/devsync/data.txt`

##### normalize() 
it will normalize the messy paths
`Path path = Path.of("uploads/../images/./profile.jpg");`
but the actual location its pointing to is `images/profile.png`

`path.normalize()`; --> images/profile.jpg

##### resolve() -- combines paths safely

many beginners write 
`String path = "uploads/" + fileName;`  --> works but fragile
Instead:
```
Path uploads = Path.of("uploads");
Path file = uploads.resolve("profile.jpg");
```

file --> uploads/profile.jpg
resolve simply attaches child path to parent path

##### relativize() --> computes the path needed to travel from one location to another

```
Path root = Path.of("uploads");
Path file = Path.of("uploads/images/profile.jpg");
```

`root.relativize(file)`  --> images/profile.jpg


### Files
`Files` is a utility class containing static methods for file-system operations

```
Files.exists(path);
Files.createFile(path);
Files.delete(path);
Files.copy(source, target);
```

#### Files.exists() -> boolean
Checks whether a file or directory exists at the specified path.
The `Files.exists()` method only returns `true` if it gets a definitive **Yes**.
true --> file exists
false --> file doesn't exists or JVM lacks permission or filesystem error occured.

`Files.notExists()` returns true only if file not found. 
Files.notExists --> commonly used.

#### Files.createFile()
-> creates an empty file
```
Path path = Path.of("notes.txt");
Files.createFile(path);
```
notes.txt is created on disk.
*** If file already exists `Files.createFile(path)` throws `FileAlreadyExistsException`
** Java wants to prevent accidental overwrites.

#### Files.createDirectory
-> Creates exactly one directory
`Files.createDirectory(Path.of("uploads"));`  -> creates uploads/

Problem
if we try
`Files.createDirectory(Path.of("uploads/images"));`
uploads/ doesnt exists throws "NoSuchFileException"
java cannot create the child when the parent doesn't exits


#### Files.createDirectories() -- mostly used
creates all missing parent directories
`Files.createDirectories(Path.of("uploads/images"));`
Results: `uploads/images`
even if neither exists.

if directory exits --> no issue
if directory doesn't exists -> created

#### Files.delete()
Deletes file or empty directory.
`Files.delete(Path.of("notes.txt"));`

If file doesn't exist throws `NoSuchFileException`
if directory is not empty throws `DirectoryNotEmptyException`

#### Files.deleteIfExists()
Deletes if present. 
`Files.deleteIfExists(path);`

returns true if deleted. returns false. if not found




#### Files.writeString()  --> return the path object
-> writes a String to a file
```
Path path = Path.of("notes.txt");

Files.writeString(path, "Hello World");
```

writes "Hello world" into notes.txt.
if `notes.txt` does not exists. file is created automatically
If file already exists by default the old content is overwritten.

internally the default behaviour is 
```
StandardOpenOption.CREATE
StandardOpenOption.TRUNCATE_EXISTING
```
means --> create if missing and clearing existing content and write new content.

##### APPEND
```
Files.writeString(
    path,
    "Log Entry",
    StandardOpenOption.APPEND
);
```
Existing content remains. 

##### TRUNCATE_EXISTING
Clear existing contents before writing.
```
Files.writeString(
    path,
    "Fresh Content",
    StandardOpenOption.TRUNCATE_EXISTING
);
```


##### Files.readString()
Reads the entire file into a String
```
String content =
    Files.readString(
        Path.of("notes.txt")
    );
```

``` notes.txt
Hello 
World
```

result :  content --> "Hello\nWorld"

*** limitation 
`readString()` loads the entire file into memory.
great for small file. bad for large files

##### Files.readAllLines()
Reads the entire file into memory and returns a list of lines.

```
List<String> lines = Files.readAllLines(path);
```
File
```
A
B
C
```

result -> ["A" "B" "c"]

#### Files.readAllBytes()
Reads the entire file into a byte array.
`byte[] data = Files.readAllBytes(path);`

readString, readAllLines & readAllBytes -- have drawbacks, --> if files are large memory storing everything in memory.. it might cause memory errors like OutOfMemoryError


```
readString()
readAllLines()
readAllBytes()
```
Only for reasonably small files

For large files
```
BufferedReader
Files.lines()
InputStream
Channels
```
Stream data incremental

readString() --> best when data is textual 
readBytes() --> best when data is binary

##### Files.copy() -> returns Path of target
Copies a file (or directory entry) from one location to another.
```
Path source = Path.of("uploads/profile.jpg");

Path target = Path.of("backup/profile.jpg");

Files.copy(source, target);
```

if target already exists `Files.copy(source, target);` throws `FileAlreadyExistsException`

If overwriting is intentional, then we can pass `StandardCopyOption.REPLACE_EXISTING`
```
Files.copy(
    source,
    target,
    StandardCopyOption.REPLACE_EXISTING
);
```


##### Files.move() --> returns Path of target
Moves or renames a file.
```
Files.move(
    Path.of("uploads/profile.jpg"),
    Path.of("archive/profile.jpg")
);
```

if target already exists `Files.copy(source, target);` throws `FileAlreadyExistsException`

If overwriting is intentional, then we can pass `StandardCopyOption.REPLACE_EXISTING`
```
Files.move(
    source,
    target,
    StandardCopyOption.REPLACE_EXISTING
);
```

##### Renaming Files
Moving within the same directory effectively becomes a rename.
```
Files.move(
    Path.of("profile.jpg"),
    Path.of("user123.jpg")
);
```

while copying or moving file.  
```
Files.move(
    Path.of("uploads/profile.jpg"),
    Path.of("archive/profile.jpg")
);
```
if /archive is not present. then it will throw an exception `NoSuchFileException`.
##### Files.list() --> Stream\<Path\>
Returns a stream containing the immediate entries in a directory. i.e 1 level deep
```
Path dir = Path.of("uploads");

Files.list(dir)
     .forEach(System.out::println);
```

```
uploads/
 ├─ a.txt
 ├─ b.txt
 └─ images/
      ├─ p1.jpg
      └─ p2.jpg
```
result
```
uploads/a.txt
uploads/b.txt
uploads/images
```


it return Stream\<Path\> but not List\<Path\> as Stream allows lazy processing.
```example
Files.list(dir)
     .filter(Files::isRegularFile)
     .count();
``` 
counts files without manually creating collections

##### Files.walk()
Traverses a directory tree recursively.
```
Files.walk(Path.of("uploads"))
     .forEach(System.out::println);  --> traverse complete directory
Files.walk(Path.of("uploads"),2)
     .forEach(System.out::println);  -> travese till level 2
```
Possible output:
```
uploads
uploads/a.txt
uploads/b.txt
uploads/images
uploads/images/p1.jpg
uploads/images/p2.jpg
```

Examples
##### Count uploaded files
```
long count =
    Files.list(uploadDir)
         .count();
```

##### Find all PDFs
```
Files.walk(uploadDir)
     .filter(path ->
         path.toString()
             .endsWith(".pdf")
     )
     .forEach(System.out::println);
```

#### File Metadata
> information about a file, not the file's contents.

##### Files.size() --> long
Returns the size of a file in bytes.

##### Files.isRegularFile(path) --> boolean
true -> path contains a file
```
resume.pdf
photo.jpg
notes.txt
```
false -> path is directory
```
uploads/
documents/
```

#### Files.isDirectory()
Checks whether the path represents a directory.
true -> path is directory
```
uploads/
documents/
```
false -> path contains a file
```
resume.pdf
photo.jpg
notes.txt
```

##### Files.isReadable(path) --> boolean
Can the JVM read this file?
true or false depending on permissions

##### Files.isWritable(path) --> boolean
Can the JVM write to this file?
true or false depending on permissions

Files.isExecutable(path) --> boolean
Can the JVM execute this file?
true or false depending on permissions
more relevent on linux/unix systems.

##### Files.getLastModifiedTime(path) --> FileTime

```
FileTime modified =
    Files.getLastModifiedTime(path);
```

return 2026-06-18T12:15:32Z;

#### Basic Attributes
`BasicFileAttributes` -> represents a collection of common file attributes
```
BasicFileAttributes attrs =
    Files.readAttributes(
        path,
        BasicFileAttributes.class
    );
```
now:
```
attrs.size();
attrs.creationTime();
attrs.lastModifiedTime();
attrs.isDirectory();
attrs.isRegularFile();
```


##### Files.lines() 
Returns a lazily populated Stream\<String\> where each element is a line from the file.
entire file is not loaded into memory
`Stream<String> stream = Files.lines(Path.of("server.log"));`


Counting Errors
```
try (Stream<String> lines =
         Files.lines(path)) {

    long count =
        lines.filter(line ->
                  line.contains("ERROR"))
             .count();

}
```

#### Why try-with-resources?
Remember:
```
Files.lines(...)
```
opens a file behind the scenes.
That file must be closed.
Therefore:
```
try (Stream<String> lines = Files.lines(path)) {}
```
is the preferred approach.

The `Stream` object returned by `Files.lines()` implements an interface called **`AutoCloseable`**. Any object that implements this interface can be put inside the `try (...)` parentheses.

##### Files.find()
```
try (Stream<Path> paths =
         Files.find(
             Path.of("uploads"),
             Integer.MAX_VALUE,
             (path, attrs) ->
                 path.toString()
                     .endsWith(".txt")
         )) {

    paths.forEach(System.out::println);
}
```

