# WinSharpFuzz: Coverage-based Fuzzing for Windows .NET

[![NuGet][nuget-shield]][nuget-link]
[![Build Status][build-shield]][build-link]
[![License][license-shield]][license-link]

[nuget-shield]: https://img.shields.io/nuget/v/WinSharpFuzz.svg
[nuget-link]: https://www.nuget.org/packages/WinSharpFuzz
[build-shield]: https://ci.appveyor.com/api/projects/status/91tx1501qeev5rwf?svg=true 
[build-link]: https://ci.appveyor.com/project/nallixion/winsharpfuzz
[license-shield]: https://img.shields.io/badge/license-MIT-blue.svg?style=flat
[license-link]: https://github.com/metalnem/sharpfuzz/blob/master/LICENSE

- Fully ported to Windows; can handle .NET/Core/Framework libraries
- Supports both IL and mixed-mode assemblies
- Uses libFuzzer heuristics to generate inputs--most libFuzzer options are supported
- Reports any erroneous exceptions as crashes
- Well-suited for discovering memory corruption issues in `unsafe` or `Marshal` method calls

## Table of Contents

- [WinSharpFuzz: Coverage-based Fuzzing for Windows .NET](#winsharpfuzz-coverage-based-fuzzing-for-windows-net)
  - [Table of Contents](#table-of-contents)
  - [Acknowledgements](#acknowledgements)
  - [Requirements](#requirements)
  - [Installation](#installation)
      - [Using NuGet](#using-nuget)
      - [From Source](#from-source)
  - [Usage](#usage)
      - [Writing a Test Harness](#writing-a-test-harness)
      - [Instrumenting the Target Library](#instrumenting-the-target-library)
      - [Building the Test Executable](#building-the-test-executable)
      - [Running WinSharpFuzz with LibFuzzer](#running-winsharpfuzz-with-libfuzzer)
- [Further Work](#further-work)

## Acknowledgements

This project is an unofficial fork of [WinSharpFuzz](https://github.com/nallixion/winsharpfuzz) which is an unofficial fork of [SharpFuzz](https://github.com/metalnem/sharpfuzz) for the Windows platform; as such, portions of this codebase were written by Nemanja Mijailovic (see NOTICE.txt for license information). This project wouldn't be possible without the groundwork he laid by developing SharpFuzz.
 
Special thanks also goes to ManTech Corporation (www.mantech.com) for facilitating the development and open source release of this fuzzing tool.

## Requirements

As the name suggests, this library is capable of fuzzing C# libraries in a Windows environment. It supports both IL and mixed-mode .dll assemblies, and .NET Framework libraries can also be instrumented and fuzzed with this tool. Linux and MacOS platforms are not supported by this fuzzer. All you will need is a Windows computer with some way to compile the C# code (either [.NET Core/Framework](https://dotnet.microsoft.com/download) or a Visual Studio installation).

In its current state, the WinSharpFuzz library only supports fuzzing with libFuzzer. A zipped binary is available to use that sends libFuzzer inputs to code instrumented with WinSharpFuzz, so no C/C++ compiler is required; however, if you want to build the binary from source then you will need a recent version of [Clang](https://llvm.org/builds/) installed.

## Installation

#### Using NuGet

The easiest way to use this library is by pulling the appropriate tools and assemblies from NuGet. For WinSharpFuzz, 
there are two tools and one library that we'll use.

To install the tools, use the following commands within a PowerShell instance:
- `dotnet tool install -g Nallixion.WinSharpFuzz.CommandLine`
- `dotnet tool install -g Nallixion.WinSharpFuzz.Instrument`

These will install everything needed to run the above commands (the -g flag installs them globally). 
Once installed, these tools can be executed using `dotnet winsharpfuzz [args]` or 
`dotnet winsharpfuzz-instrument [args]`.

Installing the WinSharpFuzz library assemblies actually happens on a per-project basis, so you'll need 
to create an instrumenting project before you complete this step. To do so, you can either create an 
empty C# console project using Visual Studio or else create one in a PowerShell instance using 
`dotnet new -n <PROJECT_NAME>`.

The way you will go about installing the library will depend on what environment you'll be building the 
WinSharpFuzz harness code. If you're building the project in Visual Studio, click the 'Project' tab and 
select 'Manage NuGet Packages...'. Then select the 'Browse' tab, search for WinSharpFuzz and install the 
library that comes up. If you're building the project using the `dotnet` command-line tool, just use 
`dotnet add package WinSharpFuzz`.

Congratulations, you now have WinSharpFuzz installed and ready to use!

#### From Source

Alternatively, one can build this project directly from its C# code. This is relatively 
straightforward, and can be accomplished either in Visual Studio or by using the `dotnet` command 
in a powershell terminal. We'll assume you've already either downloaded and unzipped the code or 
cloned this project using `git clone https://github.com/nallixion/winsharpfuzz.git`.

If using Visual Studio, navigate to the downloaded code and open the WinSharpFuzz.sln project 
(make sure you haven't just opened the folder that the .sln file resides in). Click the 'Build' tab, 
then select 'Build Solution' (or alternatively, just use the shortcut 'Ctrl+Shift+B''). 

If using a powershell terminal, simply navigate to the root directory of the project source code and 
execute the following command: `dotnet build`. It should indicate that the project build successfully.

The resulting libraries can be found in 
`src\WinSharpFuzz\bin\Debug\netstandard2.0\WinSharpFuzz.dll` and 
`src\WinSharpFuzz.Common\bin\Debug\netstandard2.0\WinSharpFuzz.CommandLine.dll`. In addition to building the 
necessary WinSharpFuzz libraries we will need, this build process also outputs two executable tools: 
one for instrumenting C# libraries so that program control flow can be 
detected by WinSharpFuzz, and one for actually running libFuzzer on our project. The instrumentation and 
libFuzzer tools will be output to `src\WinSharpFuzz.Instrument\bin\Debug\net8.0\WinSharpFuzz.Instrument.exe` and 
`src\WinSharpFuzz.CommandLine\bin\Debug\net8.0\WinSharpFuzz.CommandLine.exe` respectively.

As an aside, the libFuzzer tool makes use of two precompiled C++ binaries, `winsharpfuzz-libfuzzer-x64.exe` and 
`winsharpfuzz-libfuzzer-x86.exe`. If you want to build these from source as well, refer to the README found 
within the `libfuzzer-dotnet\` subdirectory of this project.

## Usage

The process of fuzzing C# code using this library can be broken down into the following four steps: 

1. Writing a test harness that calls the library functions you want to fuzz
2. Instrumenting the dll libraries using the `winsharpfuzz-instrument` command
3. Building an executable from the test harness + libraries
4. Running the executable using the `winsharpfuzz` command (or `WinSharpFuzz.CommandLine.exe` if you've built from source)

We'll go through how to perform each of these steps in order.

#### Writing a Test Harness

In order to run fuzzing on a library, we need to build a .NET executable that will call the 
appropriate functions in that library. This executable will be passed inputs from the underlying 
fuzzing framework (such as WinAFL or libFuzzer), and it will report the control flow taken by its 
code (as well as any exceptions that were thrown). Thankfully, WinSharpFuzz provides a simple 
abstraction for this process.

The following code shows a simple test harness built for WinSharpFuzz:

```cs
using System;
using SharpFuzz;

namespace TestExample1
{
    public class Program
    {
        public static void Main(string[] args)
        {
            Fuzzer.LibFuzzer.Run(bytes =>
            {
                // pass input bytes to library functions here
            });
        }
    }
}

```

As we can see, running fuzzing is as simple as calling `Fuzzer.LibFuzzer.Run()` from the main 
function, passing in a function (or in our case, a lambda) that accepts a ReadOnlySpan of bytes, 
and calling our library functions with the passed in bytes. The function passed into 
Fuzzer.LibFuzzer.Run will be called over and over again with unique data each time, and any 
unchecked exceptions that are thrown by the function will be recorded as crashes.

Occasionaly, you may want to fuzz a library class or method that has one-time initialization or 
cleanup procedures. Adding these functions within the `Run` method would introduce that overhead 
initialization for *every single fuzzing execution*--certainly not behavior that is desired, 
especially if the initialization and cleanup procedures are at all expensive on time or resources. 
To provide a faster alternative to this, two other methods are available that can be used to call 
any setup or teardown functions needed:

```cs
namespace TestExample1
{
    public class Program
    {
        public static void Main(string[] args)
        {
            Fuzzer.LibFuzzer.Initialize(() =>
            {
                // Initialize the library here
            });

            Fuzzer.LibFuzzer.Run(bytes =>
            {
                try
                {
                    // pass input bytes to library functions here
                } 
                catch (ExpectedException) 
                {
                    // Catch only exceptions that are meant to be thrown by the library
                }
            });

            Fuzzer.LibFuzzer.Cleanup(() =>
            {
                // Call any cleanup functions on the library here 
            });
        }
    }
}
```

`Fuzzer.LibFuzzer.Initialize()` can be used to initialize the library (or any classes within the 
library) that are to be used during the actual fuzzing, while `Fuzzer.LibFuzzer.Cleanup()` can be 
used to perform any necessary cleanup functions. Both methods are passed in a function (or, in this 
case, a lambda) with no input parameters and no return value.

Any libraries methods or classes that have been instrumented cannot be called outside of these 
three functions; any attempt to do so will most likely result in a memory access violation. 

Now, you may be wondering exactly what instrumentation is or why it results in this behavior--we'll 
cover that next.

#### Instrumenting the Target Library

In order to provide targeted fuzzing, frameworks usually require some kind of feedback on what 
control path was taken during a given execution. WinSharpFuzz gathers this feedback by 
*instrumenting* the target C# library, or adding hooks at each possible control path in the 
library. These hooks report which path of execution was taken for each fuzzing attempt, thereby 
giving the fuzzing framework clear feedback on where it should try to mutate its inputs next.

Thankfully, both the process of adding these hooks and the handling of the information they report 
are taken care of by WinSharpFuzz. The `WinSharpFuzz.CommandLine.exe` executable can be used to 
instrument dll files in place, while the `WinSharpFuzz` and `WinSharpFuzz.Common` libraries provide 
a framework that handles the information recorded by the instrumentation.

To instrument a dll file, simply execute the following command:

`winsharpfuzz-instrument \path\to\library.dll`

Or, if you've built from source:

`WinSharpFuzz.Instrument.exe \path\to\library.dll`

(make sure to execute this within the folder containing `WinSharpFuzz.CommandLine.exe`, and change 
\path\to\library.dll to the path of the library you want to instrument).

You only need to instrument each library once. If you have multiple other libraries that the target 
library depends on, you can choose whether you want to instrument those or not--but remember that 
any classes and methods called from uninstrumented libraries won't give any control flow feedback 
to the fuzzer. 

#### Building the Test Executable

Once the harness is written up, the next step is to add the necessary libraries and build the 
project. The `WinSharpFuzz.dll` and `WinSharpFuzz.Common.dll` libraries are both needed in order to 
include WinSharpFuzz in the project, so make sure to add both of those as references. If you used 
NuGet to install these libraries, then you'll have already added them to your project 
(see [installation](#installation).

Otherwise, they can be found in `src\WinSharpFuzz\WinSharpFuzz\bin\Debug\netstandard2.0\WinSharpFuzz.dll` 
and `src\WinSharpFuzz\WinSharpFuzz.Common\bin\Debug\netstandard2.0\WinSharpFuzz.Common.dll`.

TODO: explain how to add libraries both in .csproj and in Visual Studio

Now the harness should be all ready to be built. You can build by using 
Ctrl+Shift+B in Visual Studio, or by executing `dotnet build` from powershell in the root of the 
project. Once the build has finished, find the path to the resulting executable (it's usually 
something like `bin\x64\Debug\your_project_name.exe`).

#### Running WinSharpFuzz with LibFuzzer

On its own, libFuzzer is meant for C/C++ projects. However, the code provided in  
`libfuzzer-dotnet\` and WinSharpFuzz.CommandLine bridges the gap between libFuzzer and C# code 
through the use of pipes and shared memory. From a high level, these allow the libfuzzer-dotnet 
binary to pass fuzzing inputs to the C# executable, which in turn passes back control flow 
information to the C++ binary. 

With this setup, fuzzing can be performed using one simple command:

`winsharpfuzz --target_path="\path\to\HarnessExecutable.exe" "\path\to\corpus" [other-libfuzzer-options]`

Or, if you've built from source, use the following:

`.\WinSharpFuzz.CommandLine.exe --target_path="\path\to\HarnessExecutable.exe" "\path\to\corpus" [other-libfuzzer-options]`

`"\path\to\corpus"` is optional; if specified, it is the path of a folder containing example inputs 
that will be mutated by the fuzzer to provide unique tests. These inputs can be a mixture of valid 
and invalid test cases, but they should not cause the program to crash.

Additional libFuzzer options can be used from this executable, such as specifying the number of jobs to be run (i.e. the number 
of crashes to be detected before stopping the fuzzer) or the maximum input size:

`.\WinSharpFuzz.CommandLine.exe --target_path="\path\to\HarnessExecutable.exe" "\path\to\corpus" -jobs=8 -max-len=16000`

When running multiple jobs, input will be redirected to several files (usually labelled `fuzz-0.log`, `fuzz-1.log`, ...) 
that contain status information on each of the jobs being run. No information will be passed to the terminal, so it will give 
the appearance that it is frozen--this is not the case. The log files are updated in real time to provide information that would 
normally be passed to the terminal.

The number of jobs that run in parallel is by default set to half of the number of cores available on the computer. 
This can be overridden by using the `-workers=N` option (where N is the number of jobs you want to run simultaneously).
 
More information on these options can be found 
[here](https://llvm.org/docs/LibFuzzer.html#options). It should be noted that some of the memory 
options (such as `-malloc_limit_mb`) will not be useful because of the independant way in which the 
libfuzzer and .NET executables operate.

# Further Work

This library is still a work in progress. Future plans include:

- ~~Adding a build pipeline~~ Done!
- ~~Creating a NuGet repository to make installation of the library easier~~ Done!
- Automating NuGet version updates
- Exploring compatability options with WinAFL and other Windows fuzzers
