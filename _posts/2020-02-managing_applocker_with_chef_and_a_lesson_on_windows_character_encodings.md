---
layout: post
title: Managing Applocker with Chef, and a Lesson on Windows Character Encodings
---

Wanna learn about character encodings on Windows? Me neither! But unfortunately, I had to for a recent project. What follows is the generic journey I went through and how I learned a bit more than I expected about character encodings on Windows systems, and open sourced some new Chef cookbooks for managing AppLocker along the way.

## Let’s deploy blacklisting/whitelisting capabilities

The project in particular was to assess our ability to respond to vulnerabilities in applications on our Windows fleet, and our approach to vulnerability management/application control more generically. We considered a couple of possible solutions but settled on rolling AppLocker to our fleet with an eventual goal of deploying WDAC, pretty neat. In short it means that we’ll be able to enforce application blacklisting/whitelisting across our fleet leveraging Chef, with an ultimate goal of implementing a solution like Google’s Santa/Upvote, but for both Windows as well as macOS.

To approach this I started with an example use case; imagine that we’ve got some application that we’d like to prevent from running in our fleet. Perhaps this app is known to be malicious via some threat feed, or maybe it’s been decided internally that the application goes against corporate policy. The following example is very specifically tailored to trigger an issue, so bear with me here:

```cmd
C:\WINDOWS\system32> (Get-AppLockerFileInformation C:\Windows\System32\not_ntdll.dll).Publisher


PublisherName : O=MICRÖSÖFT CORPORATION, L=REDMÖND, S=WASHINGTÖN, C=US
ProductName   : MICROSOFT® WINDOWS® OPERATING SYSTEM
BinaryName    : NOT_NTDLL.DLL
BinaryVersion : 10.0.1234.567
HasPublisherName : True
HasProductName   : True
HasBinaryName : True
```

We’d like to block this from executing in our client fleet, at least for the most part. We’re not necessarily shooting for full application blockage, [as it’s relatively well known that there’s plenty of ways to bypass AppLocker](https://lmgtfy.com/?q=How+to+bypass+applocker). Rather, we’re looking for a solution that acts as a roadblock, or maybe a speed bump, to our users. We wanted a system, similar to Upvote, that would stop the user from initially executing the app as more of a sanity check and then offer a path forward if the user is sure that they really wanted to run “Totally not a virus. Trust me I’m a dolphin.exe”.

If you’ve never encountered AppLocker before, it’s got a couple of generic mechanisms for blacklisting - hash, path, or signing certificate. This list is from the weakest to the strongest mechanism, as, for example, I can bypass hash-based blacklisting by just altering the contents of the binary header, I can bypass path based blacklisting by moving the binary (assuming I have write permissions), but manipulating the signing information of the Application can sort of be a bit more difficult. Granted one can just re-sign the application with a different signing certificate to bypass the blacklist, but I digress. You can read more about AppLocker [here](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/applocker-overview), but that’s hardly the focus of this blog and moreover most of these are relatively mute points, given the number of ways one can bypass AppLocker as already mentioned :)

## Managing AppLocker rules with Chef

We just recently pushed to open source our [Chef cookbooks for managing AppLocker rules via Chef](https://github.com/facebook/IT-CPE/tree/master/itchef/cookbooks/cpe_applocker). I’m pretty proud of this cookbook as I’ve not seen many other folks managing AppLocker like this and it removes the requirement of GPO administration which is a big advantage where I work. We accomplish the configuration by having Chef convert the XML policy to a Ruby Hash and hold the state of the current AppLocker configuration in memory and then leverage Hash comparisons to determine if the configuration needs to be updated or not. Neat.

This means that if I want to ship a new AppLocker rule to my fleet, I need to make a Chef diff. We can debate the pros/cons of this approach, but once we’ve got a target rule we’d like to ship, we do the generic Chef testing flow so that we can put up our diff for review. On the first run we see the following (**Note I’ve doctored this a bit for readability**):

```chef
 * cpe_applocker[Install and configure Applocker] action configure[2020-01-09T16:05:17-08:00] INFO: Processing cpe_applocker[Install and configure Applocker] action configure (cpe_applocker::default line 25)

   [32m- update Install and configure Applocker[0m
   [32m-   set applocker_rules to {"Appx"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Dll"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Exe"=>{"mode"=>"Enabled", "rules"=>[{"type"=>"certificate", "name"=>"micrÖsoft_signed", "id"=>"40eed11e-5aa5-4e81-a9d0-630847611202", "description"=>"All binaries signed by MICRÖSÖFT are allowed.", "action"=>"Deny", "user_or_group_sid"=>"S-1-1-0", "conditions"=>[{"publisher"=>"O=MICR├ûS├ûFT CORPORATION, L=REDM├ûND, S=WASHINGT├ûN, C=US", "product_name"=>"*", "binary_name"=>"NOT_NTDLL.DLL", "binary_version"=>{"low"=>"*", "high"=>"*"}}]}}]}, "Msi"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Script"=>{"mode"=>"AuditOnly", "rules"=>[]}} 

(was {"Appx"=>{"rules"=>[], "mode"=>"AuditOnly"}, "Dll"=>{"rules"=>[], "mode"=>"AuditOnly"}, "Exe"=>{"rules"=>[], "mode"=>"AuditOnly"}, "Msi"=>{"rules"=>[], "mode"=>"AuditOnly"}, "Script"=>{"rules"=>[], "mode"=>"AuditOnly"}})
   * powershell_script[Apply updated Applocker configuration] action run[2020-01-09T16:05:18-08:00] INFO: Processing powershell_script[Apply updated Applocker configuration] action run (C:/chef/cache/cookbooks/cpe_applocker/libraries/applocker_helpers.rb line 24)
[2020-01-09T16:05:18-08:00] INFO: powershell_script[Apply updated Applocker configuration] ran successfully
```

Great. We see in our Chef run that the new polciy allowing all binaries signed by MICRÖSÖFT has been successfully applied. It’s pretty common with Chef to ensure that your cookbooks should be idempotent, so let’s do a second Chef test run to ensure that we’re not re-applying our rule:

```text
[2020-01-09T15:40:49-08:00] ERROR: Failed to parse Applocker policy from system with [#<Nokogiri::XML::SyntaxError: 1:960: FATAL: Input is not proper UTF-8, indicate encoding !
Bytes: 0x99 0x50 0x50 0x49>]

   [32m- update Install and configure Applocker[0m
   [32m-   set applocker_rules to {"Appx"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Dll"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Exe"=>{"mode"=>"Enabled", "rules"=>[{"type"=>"certificate", "name"=>"micrÖsoft_signed", "id"=>"40eed11e-5aa5-4e81-a9d0-630847611202", "description"=>"All binaries signed by MICRÖSÖFT are allowed.", "action"=>"Deny", "user_or_group_sid"=>"S-1-1-0", "conditions"=>[{"publisher"=>"O=MICR├ûS├ûFT CORPORATION, L=REDM├ûND, S=WASHINGT├ûN, C=US", "product_name"=>"*", "binary_name"=>"NOT_NTDLL.DLL", "binary_version"=>{"low"=>"*", "high"=>"*"}}]}}]}, "Msi"=>{"mode"=>"AuditOnly", "rules"=>[]}, "Script"=>{"mode"=>"AuditOnly", "rules"=>[]}}

(was nil)[0m
   * powershell_script[Apply updated Applocker configuration] action run[2020-01-09T15:40:49-08:00] INFO: Processing powershell_script[Apply updated Applocker configuration] action run (C:/chef/cache/cookbooks/cpe_applocker/libraries/applocker_helpers.rb line 24)
[2020-01-09T15:40:50-08:00] INFO: powershell_script[Apply updated Applocker configuration] ran successfully
```

Huh. That’s weird. The first Chef run we see the rules set, this is expected as we’re updating the rules to contain our whitelist of the MICRÖSÖFT publisher, but we shouldn’t see any changes in the second run as the policy has already been set. So what gives? Looking closer at the publisher information of what we’re trying to set, things don’t quite look right:

```text
"publisher"=>"O=MICR├ûS├ûFT CORPORATION, L=REDM├ûND, S=WASHINGT├ûN, C=US"
```

This seems different from our original publisher information that we’re trying to push to the fleet:

```text
PublisherName : O=MICRÖSÖFT CORPORATION, L=REDMÖND, S=WASHINGTÖN, C=US
```

So perhaps there’s some sort of encoding issue going back and forth? If we look closely at that second Chef run we see there was an issue attempting to parse the XML we got back from the system. In Chef land this part was managed by the following bit of ruby. (**Spoilers: if you check the GH repo the fix is alread in the code**):

```ruby
 begin
   current_state = powershell_out('Get-ApplockerPolicy -Effective -Xml').stdout
 rescue
   Chef::Log.warn('Failed to retrieve the effective AppLocker policy')
 end

 xml = Nokogiri::XML(current_state)
 unless xml.errors.empty?
   Chef::Log.error(
     "Failed to parse Applocker policy from system with #{xml.errors}",
   )
   # Return so we don't tank the Chef run
   return
 end
```

Windows is pretty notorious for having wide byte encodings, so maybe we can try force encoding the data when we get it back from the `powershell_out` bits to something like `UTF-8`? If we change our `powershell_out` code to force encode things before we pass to `Nokogiri::XML` this turns out to still not work. Bummer.

Examining the usage of the `Nokogiri::XML` parsing function we get another clue, perhaps we can [pass in a different encoding to `Nokogiri::XML`](https://nokogiri.org/#encoding) and have it correctly parse things out for us? This seemed like a good idea, and it might still yet have a solution, but initial attempts didn’t quite work for me.

I did the best I could Google’ing around for solutions, and found a couple of potential fixes/ideas, [this one in particular felt promising and helpful](https://www.justinweiss.com/articles/3-steps-to-fix-encoding-problems-in-ruby/), but in the end I came up short.

## So there I sat, stuck

This remained something of a problem for quite a few weeks. Well over a month actually. I hit this road block in December of 2019 and couldn’t find a way to reconcile the encoding when reading in the XML. We could deploy this recipe, but the core concept of our logic in Chef was that we maintain the config state in a hash in memory so not fixing this would mean we’d be re-applying AppLocker rules with every Chef, even worse, we’d not be able to support non-US character sets. Not really an option.

Enter a couple of weeks ago, I decided to do yet another trace of why this was happening. I decided I’d trace this all the way back to the root of where we had been getting our data and then trace the data formatting all the way up to where I was handling things in Chef. Doing some `puts` debugging with Ruby and looking at byte values of the content coming back gave me confidence that this was, indeed, an issue with invalid encodings coming back from the system calls we had been issuing, so I started digging.

My intuition here was that we’re not actually getting back stdout data from Powershell at all. The reason I felt this was, was if you look around for the default text encoding of Powershell, specifically Powershell 5.1 and higher which is what we've largely got deployed internally, you'll find that [powershell leverages the Windows-1251 standard encoding](https://docs.microsoft.com/en-us/powershell/scripting/components/vscode/understanding-file-encoding?view=powershell-6#configuring-powershell), and on Powershell 6.0 and higher, which I’ve got installed on my box, the default encoding is `UTF-8`. I had tried swapping out the encoding used by Nokogiri to various setups, `UTF-16LE`, `UTF-16`, `Windows-1251`, `Windows-1252` even! All to no avail.

This gave me the impression that something else was at play. Surely if we were getting back data encoded as aforementioned we’d be in business! So, perhaps we’re not actually dealing with Powershell? Or for that matter, we’re not dealing with something encoding the data in any modern standard?

## How does Chef shellout to the underlying system?

To apply our AppLocker policies we use the `powershell_out` helper in Chef. How does the `powershell_out` function work? Looking around github you can find `powershell_out` is a helper function in Chef Windows utilities [here](https://github.com/chef/chef/blob/master/lib/chef/mixin/powershell_out.rb#L33). This function leverages another helper defined in the same class `run_command`, which under the hood leverages [`shell_out`](https://github.com/chef/chef/blob/master/lib/chef/mixin/shell_out.rb). `shell_out` is the more generic way that one would execute shell code on the native system under the hood with Chef, packaged up by the `Chef::Mixin::ShellOut` library.

So checking out the Chef::Mixlib::ShellOut module on we are [redirected](https://github.com/chef/mixlib-shellout/blob/master/lib/mixlib/shellout.rb#L32) to the `shellout/windows` module where we _finally_ find a [definition for the `run_command` function](https://github.com/chef/mixlib-shellout/blob/master/lib/mixlib/shellout/windows.rb#L55) sitting beneath all of our shellout logic.

Here we start seeing some familiar faces of native Windows API functions like `WaitForSingleObject` which give us an idea about how Chef is doing the native code execution. Continuing to dig a bit further we find the desired calls to [`CreateProcessW`](https://github.com/chef/mixlib-shellout/blob/master/lib/mixlib/shellout/windows/core_ext.rb#L456). Woot! This is what’s interacting with Windows to get our stdout data populated.

With the knowledge that we’re leveraging native API calls underneath the hood, it’s time to figure out what the generic encodings for native Windows APIs are. Doing some searching around Google, we find that this mechanism for invoking a process and getting the output means our encoding will be relatively similar to that of cmd.exe, which points us to this [massive list of Windows Code Page Identifiers](https://docs.microsoft.com/en-us/windows/win32/intl/code-page-identifiers) (**TIL!**, well, that day I learned.)

So it looks like there’s a strong chance that the data we’re getting back from our `powershell_out` invocation is likely Code Page 437 encoded, as I’m running on a Windows host provisioned in the US. In a normal env, if one wanted to change the character encoding in their shell on a windows host they’d use the `chcp` command, so why don’t we use this in our Chef cookbook to change encodings to something a bit more sane, say `UTF-8`, via `chcp 65001`. If we drop this into our `powershell_out` invocation:

```powershell
current_state = powershell_out(
 'chcp 65001 | Out-Null; Get-ApplockerPolicy -Effective -Xml',
).stdout
```

**#money**, we’re in business, and `Nokogiri::XML` doesn't have any issues parsing our XML data! We do the multiple Chef runs, and sure enough the second run has no changes.

That being said, it still looks like there's some weird parsing errors when Ruby displays what policies are being set, but it doesn't matter as long as the AppLocker policies that are being enforced and Chef parses them to a Ruby hash the same way every time, which they do:

```xml
<FilePublisherRule Id="f921179a-bfb6-456f-9678-4f92c467d98e" Name="teamviewer_blocked_publisher" Description="Teamviewer is not approved for use on client devices. See https://fburl.com/542212880 for more details" UserOrGroupSid="S-1-1-0" Action="Deny">
 <Conditions>
   <FilePublisherCondition PublisherName="O=MICR├ûS├ûFT CORPORATION, L=REDM├ûND, S=WASHINGT├ûN, C=US" ProductName="MICROSOFT® WINDOWS® OPERATING SYSTEM" BinaryName="NOT_NTDLL.DLL">
     <BinaryVersionRange LowSection="*" HighSection="*" />
   </FilePublisherCondition>
 </Conditions>
</FilePublisherRule>
```

Overall this might have seemed relatively trivial for a more seasoned Windows OS human but I wanted to post this as I thought it was a good exercise on digging through source code to uncover truths and verify assumptions.

Keep in mind, when executing things on Windows, it could be one, of _many_, potential encodings.
