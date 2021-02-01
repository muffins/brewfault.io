Considering extensions on osquery are getting more and more support, I figured I'd throw up this guide for building osquery extensions on Windows in C++, as we're still [working on developing osquery python extensions for Windows](https://github.com/muffins/osquery-python/tree/osquery-python-windows-port). What follows are the build steps for developing Windows C++ extensions in osquery:

First, from within the master osquery repo, drop your extensions implementation file into a subdirectory under `external`. In the below sample I created a new directory for my implemenation files (this is needed) called `extension_test`, and I created a `.cpp` with my code living in this folder:
```
C:\Users\thor\work\repos\osquery [master ≡]
λ  cat .\external\extension_test\sample_extension.cpp
// Note 1: Include the sdk.h helper.
#include <osquery/sdk.h>

using namespace osquery;

// Note 2: Define at least one plugin.
class ExampleTablePlugin : public tables::TablePlugin {
 private:
  tables::TableColumns columns() const override {
    return {
      std::make_tuple("example_text", TEXT_TYPE, ColumnOptions::DEFAULT),
      std::make_tuple("example_integer", INTEGER_TYPE, ColumnOptions::DEFAULT),
    };
  }

  QueryData generate(tables::QueryContext& request) override {
    QueryData results;
    Row r;

    r["example_text"] = "example";
    r["example_integer"] = INTEGER(1);
    results.push_back(r);
    return results;
  }
};

// Note 3: Use REGISTER_EXTERNAL to define your plugin.
REGISTER_EXTERNAL(ExampleTablePlugin, "table", "example");

int main(int argc, char* argv[]) {
  // Note 4: Start logging, threads, etc.
  osquery::Initializer runner(argc, argv, ToolType::EXTENSION);

  // Note 5: Connect to osqueryi or osqueryd.
  auto status = startExtension("example", "0.0.1");
  if (!status.ok()) {
    LOG(ERROR) << status.getMessage();
    runner.requestShutdown(status.getCode());
  }

  // Finally shutdown.
  runner.waitForShutdown();
  return 0;
}
```
We can now make use of the osquery build scripts to generate the Visual Studio solution and also build the sample code for us. The `CMake` logic in our build scripts will automatically build any projects living under `external` so long as they're named `extension_`.
```
C:\Users\thor\work\repos\osquery [master ≡]
λ  .\tools\make-win64-binaries.bat
--
-- Welcome to osquery's build-- thank you for your patience! :)
-- For a brief tutorial see: http://osquery.readthedocs.io/en/stable/development/building/
-- Building for platform Windows (windows, windows10)
-- Building osquery version  2.8.0-14-g9d332617 sdk 2.8.0
mkdir: cannot create directory 'C:/Users/thor/work/repos/osquery/build/windows10/generated': File exists
-- Configuring done
-- Generating done
-- Build files have been written to: C:/Users/thor/work/repos/osquery/build/windows10
Microsoft (R) Build Engine version 14.0.25420.1
Copyright (C) Microsoft Corporation. All rights reserved.
...
  osquery_extensions.vcxproj -> C:\Users\thor\work\repos\osquery\build\windows10\osquery\extensions\osquery_extensions.dir\Release\osquery_extensions.lib
...
  osquery_extensions.vcxproj -> C:\Users\thor\work\repos\osquery\build\windows10\osquery\extensions\osquery_extensions.dir\Release\osquery_extensions.lib
...
  osquery_tables_tests.vcxproj -> C:\Users\thor\work\repos\osquery\build\windows10\osquery\Release\osquery_tables_tests.exe
Test project C:/Users/thor/work/repos/osquery/build/windows10
    Start 1: osquery_tests
1/5 Test #1: osquery_tests ....................   Passed    1.92 sec
    Start 2: osquery_additional_tests
2/5 Test #2: osquery_additional_tests .........   Passed   38.74 sec
    Start 3: osquery_tables_tests
3/5 Test #3: osquery_tables_tests .............   Passed    1.94 sec
    Start 4: python_test_osqueryi
4/5 Test #4: python_test_osqueryi .............   Passed   67.82 sec
    Start 5: python_test_osqueryd
5/5 Test #5: python_test_osqueryd .............   Passed   19.57 sec

100% tests passed, 0 tests failed out of 5

Total Test time (real) = 130.09 sec
```
Now that we have our sample extension built we have two options. We can either launch Visual Studio and use this to build our extension project, or if we'd prefer to avoid VS, you can manually invoke `msbuild` to compile the project code for us. 

I recommend just leveraging the Visual Studio route, however for the ambitious below are the instructions for manually invoking. It basically amounts to use a new Powershell commandlet to invoke the `vcvarsall.bat` script (**note** you could also just use a VS Native Build Tools command prompt, but I love my shell ^.^), and then invoking `msbuild` with our project name:
```
C:\Users\thor\work\repos\osquery [master ≡]
λ  cd .\build\windows10\

C:\Users\thor\work\repos\osquery\build\windows10 [master ≡]
λ  (Get-Command Invoke-BatchFile).Definition

  param([string]$Path, [string]$Parameters)
  $tempFile = [IO.Path]::GetTempFileName()
  cmd.exe /c " `"$Path`" $Parameters && set > `"$tempFile`" "
  Get-Content $tempFile | Foreach-Object {
    if ($_ -match "^(.*?)=(.*)$") {
      Set-Content "env:\$($matches[1])" $matches[2]
        }
  }
  Remove-Item $tempFile

C:\Users\thor\work\repos\osquery\build\windows10 [master ≡]
λ  Invoke-BatchFile "$env:VS140COMNTOOLS\..\..\vc\vcvarsall.bat" amd64

C:\Users\thor\work\repos\osquery\build\windows10 [master ≡]
λ   msbuild osquery.sln /p:Configuration=Release /p:PlatformType=x64 /p:Platform=x64 /t:external_extension_test /m /v:m

Microsoft (R) Build Engine version 14.0.25420.1
Copyright (C) Microsoft Corporation. All rights reserved.

...

  sample_extension.cpp
  LINK : /LTCG specified but no code generation required; remove /LTCG from the link command line to improve linker performance
  external_extension_test.vcxproj -> C:\Users\thor\work\repos\osquery\build\windows10\external\Release\external_extension_test.ext.exe
```
Woot! We now have a `external_extension_test.ext.exe` that'll interoperate with our osquery binaries. Now let's look at how we deploy this extensions to our enterprise and hook it up to an osquery process.

To do this one will require a deployment process for shipping the binaries, and also for setting file permissions on extensions. Below I assume you take this step, and just show what's required to get the extension talking to the osquery service on startup:
```
C:\Users\thor\work\repos\osquery [master ≡]
λ  cp .\build\windows10\external\Release\external_extension_test.ext.exe C:\ProgramData\osquery\extensions\example.exe

C:\Users\thor\work\repos\osquery [master ≡]
λ  cat C:\ProgramData\osquery\osquery.flags
--disable_extensions=false
--config_path=C:\ProgramData\osquery\osquery.conf
--config_plugin=filesystem
--logger_plugin=filesystem
--logger_path=C:\ProgramData\osquery\log
--extensions_autoload=C:\ProgramData\osquery\extensions.load

C:\Users\thor\work\repos\osquery [master ≡]
λ  . .\tools\provision\chocolatey\osquery_utils.ps1
C:\Users\thor\work\repos\osquery [master ≡]

λ  Set-DenyWriteAcl C:\ProgramData\osquery\extensions 'Add'
True

C:\Users\thor\work\repos\osquery [master ≡]
λ  osqueryi --flagfile=C:\ProgramData\osquery\osquery.flags
Using a virtual database. Need help, type '.help'
osquery> .tables
...
  => etc_services
  => example
  => file
...
osquery> select * from example;
+--------------+-----------------+
| example_text | example_integer |
+--------------+-----------------+
| example      | 1               |
+--------------+-----------------+
osquery> .quit

C:\Users\thor\work\repos\osquery [master ≡]
λ  cat C:\ProgramData\osquery\osquery.conf
{
  "options": { },
  "schedule": {
    "system_info": {
      "query": "SELECT hostname, cpu_brand, physical_memory FROM system_info;",
      "interval": 3600
    },
    "extension_sample": {
      "query": "SELECT * FROM example;",
      "interval": 5,
      "snapshot": "true"
    }
  }
}

C:\Users\thor\work\repos\osquery [master ≡]
λ  Start-Service osqueryd

C:\Users\thor\work\repos\osquery [master ≡]
λ  Get-Service osqueryd

Status   Name               DisplayName
------   ----               -----------
Running  osqueryd           osqueryd

C:\Users\thor\work\repos\osquery [master ≡]
λ  tail -f C:\ProgramData\osquery\log\osqueryd.snapshots.log
{"snapshot":[{"example_integer":"1","example_text":"example"}],"action":"snapshot","name":"extension_sample","hostIdentifier":"TESTFAC-MMFN45S","calendarTime":"Tue Sep 26 19:32:30 2017 UTC","unixTime":"1506454350","epoch":"0"}
{"snapshot":[{"example_integer":"1","example_text":"example"}],"action":"snapshot","name":"extension_sample","hostIdentifier":"TESTFAC-MMFN45S","calendarTime":"Tue Sep 26 19:32:35 2017 UTC","unixTime":"1506454355","epoch":"0"}
{"snapshot":[{"example_integer":"1","example_text":"example"}],"action":"snapshot","name":"extension_sample","hostIdentifier":"TESTFAC-MMFN45S","calendarTime":"Tue Sep 26 19:32:41 2017 UTC","unixTime":"1506454361","epoch":"0"}
{"snapshot":[{"example_integer":"1","example_text":"example"}],"action":"snapshot","name":"extension_sample","hostIdentifier":"TESTFAC-MMFN45S","calendarTime":"Tue Sep 26 19:32:46 2017 UTC","unixTime":"1506454366","epoch":"0"}
...

C:\Users\thor\Desktop\tmp
λ  l C:\ProgramData\osquery\log\


    Directory: C:\ProgramData\osquery\log


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        9/26/2017  12:32 PM            490 osqueryd.INFO.20170926-123230.10976
-a----        9/26/2017  12:17 PM              0 osqueryd.results.log
-a----        9/26/2017  12:33 PM           3632 osqueryd.snapshots.log
```

There's a lot happening up above, let's walk through some of this step-by-step.

First, we copy the extension binary we built earlier to `C:\ProgramData\osquery\extensions` (**Again Note** in your environment it's assumed you'd deploy here using Chef or Puppet). Once the binary is in place, we need to set the proper file permissions to assure the binary will load. To do this we make use of our helper Powershell libraries by 'dot' sourcing the script, and invoking the `Set-DenyWriteAcl` cmdlet with `. .\tools\provision\chocolatey\osquery_utils.ps1`, and then we call the function, `Set-DenyWriteAcl C:\ProgramData\osquery\extensions 'Add'`. 

Now that the binary is in place, we update our osquery flags file to turn on extensions, ensure the full path to our extensions binary is in our autoload file, `C:\ProgramData\osquery\extensions.load`, and then finally ensure we have some scheduled queries setup in `osquery.conf`. If you're using a TLS server for your configuration, you'd simply want to schedule a few queries against your extension there.

Lastly, we turn everything on locally, and verify we're getting logged output into our filesystem logger plugin.

Happy Hacking!