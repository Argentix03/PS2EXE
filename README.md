
## Usage
```powershell
  Invoke-ps2exe .\source.ps1 .\target.exe
```
or
```powershell
  ps2exe .\source.ps1 .\target.exe
```
compiles "source.ps1" into the executable target.exe (if ".\target.exe" is omitted, output is written to ".\source.exe").

or start Win-PS2EXE for a graphical front end with
```powershell
  Win-PS2EXE
```

## Parameter
```powershell
ps2exe [-inputFile] '<file_name>' [[-outputFile] '<file_name>']
       [-prepareDebug] [-x86|-x64] [-lcid <id>] [-STA|-MTA] [-noConsole] [-UNICODEEncoding]
       [-credentialGUI] [-iconFile '<filename>'] [-title '<title>'] [-description '<description>']
       [-company '<company>'] [-product '<product>'] [-copyright '<copyright>'] [-trademark '<trademark>']
       [-version '<version>'] [-configFile] [-noOutput] [-noError] [-noVisualStyles] [-exitOnCancel]
       [-DPIAware] [-requireAdmin] [-supportOS] [-virtualize] [-longPaths]
```

```
      inputFile = Powershell script that you want to convert to executable (file has to be UTF8 or UTF16 encoded)
     outputFile = destination executable file name or folder, defaults to inputFile with extension '.exe'
   prepareDebug = create helpful information for debugging    
     x86 or x64 = compile for 32-bit or 64-bit runtime only
           lcid = location ID for the compiled executable. Current user culture if not specified
     STA or MTA = 'Single Thread Apartment' or 'Multi Thread Apartment' mode
      noConsole = the resulting executable will be a Windows Forms app without a console window
UNICODEEncoding = encode output as UNICODE in console mode
  credentialGUI = use GUI for prompting credentials in console mode
       iconFile = icon file name for the compiled executable
          title = title information (displayed in details tab of Windows Explorer's properties dialog)
    description = description information (not displayed, but embedded in executable)
        company = company information (not displayed, but embedded in executable)
        product = product information (displayed in details tab of Windows Explorer's properties dialog)
      copyright = copyright information (displayed in details tab of Windows Explorer's properties dialog)
      trademark = trademark information (displayed in details tab of Windows Explorer's properties dialog)
        version = version information (displayed in details tab of Windows Explorer's properties dialog)
     configFile = write config file (<outputfile>.exe.config)
       noOutput = the resulting executable will generate no standard output (includes verbose and information channel)
        noError = the resulting executable will generate no error output (includes warning and debug channel)
 noVisualStyles = disable visual styles for a generated windows GUI application (only with -noConsole)
   exitOnCancel = exits program when Cancel or "X" is selected in a Read-Host input box (only with -noConsole)
       DPIAware = if display scaling is activated, GUI controls will be scaled if possible (only with -noConsole)
   requireAdmin = if UAC is enabled, compiled executable run only in elevated context (UAC dialog appears if required)
      supportOS = use functions of newest Windows versions (execute [Environment]::OSVersion to see the difference)
     virtualize = application virtualization is activated (forcing x86 runtime)
      longPaths = enable long paths ( > 260 characters) if enabled on OS (works only with Windows 10)
```

A generated executable has the following reserved parameters:

```
-debug              Forces the executable to be debugged. It calls "System.Diagnostics.Debugger.Launch()".
-extract:<FILENAME> Extracts the powerShell script inside the executable and saves it as FILENAME.
                    The script will not be executed.
-wait               At the end of the script execution it writes "Hit any key to exit..." and waits for a key to be pressed.
-end                All following options will be passed to the script inside the executable.
                    All preceding options are used by the executable itself and will not be passed to the script.
```


## Remarks

### List of cmdlets not implemented:
The basic input/output commands had to be rewritten in C# for PS2EXE. Not implemented are *Write-Progress* in console mode (too much work) and *Start-Transcript*/*Stop-Transcript* (no proper reference implementation by Microsoft).

### GUI mode output formatting:
Per default in powershell outputs of commandlets are formatted line per line (as an array of strings). When your command generates 10 lines of output and you use GUI output, 10 message boxes will appear each awaiting for an OK. To prevent this pipe your commandto the comandlet Out-String. This will convert the output to one string array with 10 lines, all output will be shown in one message box (for example: dir C:\ | Out-String).

### Config files:
PS2EXE can create config files with the name of the generated executable + ".config". In most cases those config files are not necessary, they are a manifest that tells which .Net Framework version should be used. As you will usually use the actual .Net Framework, try running your excutable without the config file.

### Parameter processing:
Compiled scripts process parameters like the original script does. One restriction comes from the Windows environment: for all executables all parameters have the type STRING, if there is no implicit conversion for your parameter type you have to convert explicitly in your script. You can even pipe content to the executable with the same restriction (all piped values have the type STRING).

### Password security:
Never store passwords in your compiled script! One can simply decompile the script with the parameter -extract. For example 
```powershell
Output.exe -extract:C:\Output.ps1
```
will decompile the script stored in Output.exe.

### Script variables:
Since PS2EXE converts a script to an executable, script related variables are not available anymore. Especially the variable $PSScriptRoot is empty.

The variable $MyInvocation is set to other values than in a script.

You can retrieve the script/executable path independant of compiled/not compiled with the following code (thanks to JacquesFS):

```powershell
if ($MyInvocation.MyCommand.CommandType -eq "ExternalScript")
 { $ScriptPath = Split-Path -Parent -Path $MyInvocation.MyCommand.Definition }
 else
 { $ScriptPath = Split-Path -Parent -Path ([Environment]::GetCommandLineArgs()[0]) 
     if (!$ScriptPath){ $ScriptPath = "." } }
```

### Window in background in -noConsole mode:
When an external window is opened in a script with -noConsole mode (i.e. for Get-Credential or for a command that needs a cmd.exe shell) the next window is opened in the background.

The reason for this is that on closing the external window windows tries to activate the parent window. Since the compiled script has no window, the parent window of the compiled script is activated instead, normally the window of Explorer or Powershell.

To work around this, $Host.UI.RawUI.FlushInputBuffer() opens an invisible window that can be activated. The following call of $Host.UI.RawUI.FlushInputBuffer() closes this window (and so on).

The following example will not open a window in the background anymore as a single call of "ipconfig | Out-String" will do:

```powershell
$Host.UI.RawUI.FlushInputBuffer()
ipconfig | Out-String
$Host.UI.RawUI.FlushInputBuffer()
```


