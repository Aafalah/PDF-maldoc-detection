# PDF-maldoc-detection
###### A tutorial for detection PDF maldocs using open source tools
We aim to help those who are starting their PDF maldoc analysis journey with the first steps they can take

## Analysis
In this sction, we provide analysis of 3 PDF maldocs generated through Metasploit.
- adobe_collectemailinfo
- adobe_pdf_embedded_exe
- adobe_pdf_embedded_exe_nojs

### JavaScript analysis
This maldoc was generated using the "collectemailinfo" module on Metasploit.

We start the analysis with performing basic scanning of the file using the PDFiD tool. This scan will show what objects and keywords are used within the document.
You can use the following command:
```
pdfid adobe_collectemailinfo.pdf
```
The scanning results shows that the file has 6 objects, 1 stream, 1 page, 1 /JS, 1 obfuscated /JavaScript and 1 /Openaction action.
We can move to the PDFStreamDumper tool to further inveistigate.
At a glance, object colours tell us the following:
object 1 (green) contains /Launch, /OpenAction or /AA
Object 5 (red) contains JavaScript.
You can find the full details of the colours and their meaning under 
> tools -> About Listview Colors

Click on Object 5: "**5 HLen: 0x3A**", then click on the "**HexDump**" tab. Notice the obfuscation employed by the maldoc author. Hex digits are used to hide the JavaScript tag.
Now click on Object 6, which contains the stream. Click the "**Text**" tab. 
You can see some very suspicious obfuscated and encoded string. Let's invistigate further.
Click on the "**Javascript_UI**" from the menu bar. A new window will launch that looks slightly better but still highly unreadable.
Now click on "**Format_Javascript**".

Typically, you will want to deobfuscate the JavaScript code to find out what it does. However, in this case, we can skip this step because there are 2 very clear indicators of malicious content:
The *unescape* function, followed by a Unicode-encoded string (*%uxxxx*)

Highlight the whole purple string then click click on "**Shellcode_Analysis**". Pick the 2nd option from the dropdown menu: "**scDbg (libEmu - Emulation)**"
This will launch yet another window that allows you to emulate the execution of what appears to be a shellcode. Click on "**Launch**" from the new Window.

The emulation results show that the shellcode attempted to initiate a network connection to the IP address 10.10.19.85 on port 4444.

The analysis of this maldoc is now complete. 

You might encounter some JavaScript blocks that do not execute directly and might require some editting, such as the analysis traps we highlighted in our paper. 
Also, some JavaScript blocks will not reveal the shellcode directly. You will have to modify then execute the code in order to reveal subsequent attack stages which contain the final payload/shellcode.


### EmbeddedFiles analysis
There are 2 ways in which you can generate a *dropper* on Metasploit:
- Executable dropping through JavaScript, which uses the module: "*adobe_pdf_embedded_exe*".
- Executable dropping without JavaScript,  which uses the module: "*adobe_pdf_embedded_exe_nojs"*.

In this section, we'll analyze a sample of both files.

#### Adobe_pdf_embedded_exe
Let's start this analysis with PeePDF.
```
peepdf adobe_pdf_embedded_exe.pdf 
```
The scanning results show several red flags: 
- JavaScript code (object 9)
- OpenAction (object 1) and AdditionalAction (object 3)
- Launch (object 10)
- EmbeddedFiles (object 5)

Time to invistigate all these objects to see what they do: object 1, 3, 5, 9, 10.
Lets start the interactive console of PeePDF using:
```
peepdf adobe_pdf_embedded_exe.pdf -i
```
Follow the following commands to perform command-line analysis:
```
object 1
```
This returns the following:
> << /OpenAction 9 0 R
 /Pages 2 0 R
 /Names 5 0 R
 /Type /Catalog >>

The content of object 1 show that object 9 will be executed once the document is opened. We know object 9 contains a JavaScript block from the previous step.
Let's check object 9
```
object 9
```
`object 9` returns 3 lines. The first line suggest that this object type is "Action", which means it is something that will be performed or executed by the reader.
> << /Type /Action

The second line suggest that is object is a script, of the type "JavaScript. 
> /S /JavaScript

The 3rd line provides the JavaScript code:
> /JS this.exportDataObject({ cName: "template", nLaunch: 0 }); >>

This suggests the code will export something to the user's computer. The *dropped* file name is *template*, and uses 0 as a launch option, which means the file will not be executed once it is saved to the disk.

Let's try and find out what the dropped file does. Seems like it is a good time to check object 5 with `object 5`, which shows
> << /EmbeddedFiles 6 0 R >>

Let's follow the chain: type  `object 6` in the console:
> << /Names [ template 7 0 R ] >>

BINGO! This is a "*names dictionary*" which is used to give a name to object 7, which is now called *template*. Now we know that the JavaScript code will save he content of object 7 to the disk. Let's check what object 7 has in store for us! Type `object 7`

This returns several lines, including the file type of the dropped file, which is another PDF document. The last line suggest the content can be found in object 8. Type `object 8`

Oops! This object contains some encoded string and what looks like a help message of a program. Make a note of the first few lines of the object:
> << /Length 44156
/Filter /FlateDecode

This is the length of the stream and the filter used to encode it with.
Let's see if we can carve this thing out for further analysis. Type the following command in the console:
```
object 8 $> var_a
```
This command saves the content of object 8 into a new variable we called "*var_a*". Now type `help` to see if there are any commands that might help us identify this.

To check the content of the variable, type `show var_a`, which returns a hex view of the content. We can try and scroll up to inspect the header of this file, but the content is too large for the console. Let's save it to the disk using it's original format before encoding. Type:
```
rawstream 8 $> var_b
```
This command saves the raw content of the object into a new variable. We'll attempt to decode it then save it using the following command:
```
decode variable var_b fl  > outputfile
```
This should have decoded the rawstream then saved it to the same directory where the original PDF file is. 

Keep the interactive console running, and start a new terminal. Navigate to where the new file is saved. Let's check out the file type. Type:
``` file outputfile```
This returns:
> outputfile: PE32 executable (GUI) Intel 80386, for MS Windows

This is an excutable. So why did the file specification suggest it was a PDF? This is a trick applied by maldoc writers to bypass Adobe security settings that prevent a PDF document from dropping or saving an executable to the user's disk.
You can use IDA pro or another debugger to invistigate what this executable does. However, seeing as this is out of scope, we'll leave this step to you to figure out.

Our analysis is not complete. We know there is an embeddedfile that gets saved to the disk by the JavaScript code, but we also know the code does not run the dropped file. There must be another way. Let's keep looking. 

Type `info` to check the basic scanning info again.
We are yet to invistigate the /AA (AdditionalAction) in object 3. Type: `object 3`. 

This suggests that there is an additional action that takes place *after* the previous steps. The executed action is in object 10, which contains the "*Launch*" action.
A *Launch* action will start a program on the user's computer when it gets executed. Let's check it out: `object 10`.

This launches a command lime interface and types the following command:
> %HOMEDRIVE%&cd %HOMEPATH%&(if exist "Desktop\template.pdf" (cd "Desktop"))&(if exist "My Documents\template.pdf" (cd "My Documents"))&(if exist "Documents\template.pdf" (cd "Documents"))&(if exist "Escritorio\template.pdf" (cd "Escritorio"))&(if exist "Mis Documentos\template.pdf" (cd "Mis Documentos"))&(start template.pdf)

> To view the encrypted content please tick the "Do not show this message again" box and press Open. >> >>

This command attempts to locate te dropped PDF file then execute it.

The analysis of this maldoc is now complete. You can type `exit` to leave the interactive console and return to the terminal.

#### Adobe_pdf_embedded_exe_nojs
Let's start our analysis in this step using the interactive console of PeePDF:
```
peepdf adobe_pdf_embedded_exe_nojs.pdf -i
```
The results show a launch action and openAction, but no other indicators of malicious content. Also, the file is very small with only 5 objects, and no streams. Something does not look right, specially that the file size is nearly 300kb, so it must have some content.

Type `metadata` to inspect the metadata of the file. No results are returned. Type `tree` to inspect the hierarchical structure of the document. Once again, no indicator of content. Let's inspect those suspicious elements. Type `object 1`. We can see object 5 will be executed when the documen is opened. Let's inspect that with `object 5`. 

We can see a lanuch action that will start a command line interface and type a very long and complicated command. 

The command seems to be retrieving some data from the document itself using a Visual Basic script. As inspecting the lecgical structure did not reveal any content, it is time we ispect the physical structure of the document. Type `offsets`.

The results show that the header is at offset 0 - as it should be. However, object 1 does not start until offset 295,220. This is the size in bytes. Which nearly equals the size of the entire document. This suggests that there is something embedded between the header and the first object. The easiest way to ispect this is to use a hex viewer. Launch your favorite hex viewer and inspect the document.

You can clearly see a massive hexadecimal string right after the header of the file. Highlight the string, upto the declaration of the first object "1 0 obj<<". Now copy the string into a new file. 

The hexadecimal string can be converted into a binary using a short script (Perl or Python will do).

If you do not want to use a script, you can follow the following steps:
- Paste the hex code into a text editor.
- Remove all the \x using the replace option (replace it with nothing).
- Copy the new string.
- Create a blank file on a hex viewer.
- Insert or paste the string as a hex code (middle-pane). If you paste this as ASCII string, the process will fail.
- Save your file.

Once you have carved out the new file, you can follow the same steps from the previous section to identify the type using `file` then analyse it using IDA pro.

This concludes the analysis in this of this maldoc.



## Distros

The majority of our analysis was performed on tools that are preloaded on the [REMnux distro](https://github.com/REMnux). We also make use of the [FlareVM](https://github.com/fireeye/flare-vm) 


## Tools
If you'd rather install the tools on a different environment, here is a list of the tools we used:
- [PeePDF](https://github.com/jesparza/peepdf)
- [PDFStreamDumper](https://github.com/dzzie/pdfstreamdumper)
- [PDFiD & PDF-Pareser](https://blog.didierstevens.com/didier-stevens-suite/)

## Maldocs
You can find the maldocs we used in this tutorial in this repo.


## Disclaimer
We created this tutorial as part of our paper submitted to **coming soon**.

The work contains malicious file secured using the industry-standard password. Use at your own risk. We are not responsible for any damage caused through the misuse of these maldocs.
















