[[File Handling2]]
```
File System(Operating System)
			↓
		File (java.io)
			↓
		Input/Output Stream
```


### File (java.io)
`File file = new File('Sample.txt');`

This does not 
- Open file
- Read file
- Write File
<mark> It just represent file system path and provides helper methods to interact with that path via the Os</mark>

Path Helper methods
- file.getName();
- file.getAbsolutePath();

Metadata helper methods
- file.exists()
- file.isDirectory();
- file.lastModified();

File Operation helper methods
- file.createNewFile();
- file.delete();
- file.mkdir


### Input/OutPut Stream

it is the on which actually makes OS system calls to 
- open
- read 
- write file

Considered Stream as bridge to OS to manage file.
Streams are one directional

Input/Output stream internally make use of Decorator Design Pattern

![[Pasted image 20260615082055.png]]


In the lowest level. everything happens in bytes 
reading/ writing of characters,Integers,Images,Objects etc

![[Pasted image 20260615082405.png]]

every write/read makes a system call
![[Pasted image 20260615082614.png]]
![[Pasted image 20260615082846.png]]![[Pasted image 20260615083028.png]]
![[Pasted image 20260615083047.png]]
![[Pasted image 20260615083110.png]]![[Pasted image 20260615083227.png]]
![[Pasted image 20260615083401.png]]













