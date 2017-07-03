# Running PowerShell on AWS Lambda

## Why?

So you may be asking yourself, "Self, why would anyone want to run PowerShell on AWS Lambda?"

PowerShell is a great tool to manage infrastructure.  Traditionally, it's been a Windows-only
tool in that it could only run on a Windows-based operating system.  But with Microsoft's
*Openness Renaissance* of the last few years, PowerShell has [hopped on](https://azure.microsoft.com/en-us/blog/powershell-is-open-sourced-and-is-available-on-linux/)
the open source and cross-platform train and the future looks very promising.

With *classic* PowerShell (that is, Windows PowerShell or the more recently retrofitted moniker, *PowerShell Desktop edition*), you always had great power to manage local Windows hosts.  But even then, you also
had the ability to remotely manage infrastructure components, such as [virtualization](https://code.vmware.com/web/dp/tool/vsphere-powercli/6.5.1), [networking](http://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/sw/msft_tools/installation_guide/powertool/b_Pwrtool_Install_and_Config/b_Install_and_Config_chapter_01.html#id_16505), [storage](http://community.netapp.com/t5/Microsoft-Cloud-and-Virtualization-Discussions/NetApp-PowerShell-Toolkit-4-0-containing-DataONTAP-PowerShell-Toolkit-3-3/m-p/109322) and general [datacenter administration](https://technet.microsoft.com/en-us/library/dn265975).  Microsoft products like Exchange and Sharepoint
have made PowerShell the primary vehicle for their administration, both [on-prem](https://practical365.com/powershell/) and in the [cloud](https://technet.microsoft.com/en-us/library/cb90889b-9c1e-4cec-ab0f-774be623022f).

And I should make special mention, especially befitting to this article, the excellent
support for managing all things AWS with the [`AWSPowerShell` module](http://docs.aws.amazon.com/powershell/latest/userguide/pstools-getting-set-up.html).


### Why Serverless?

### Why AWS Lambda?

Support for PowerShell on a Serverless (or FaaS) platform already exists in Azure, so why
reproduce this in AWS?  Simple, if your infrastructure is already hosted in AWS, then it
make sense to host infrastructure management and suppot there as well.  Especially when
you consider the special integrations and support that Lambda provides for private
networking (VPC) and security (IAM), running PowerShell tasks on Lambda gives you
special support to automate!

### Why?  Because you can!

This exercise attempts to put together a number of well-established, as well as still
evovling, tools and techniques to prove you can do something that probably would have
been unthinkable just a short couple of years ago.

It brings together technologies from different disciplines and different vendors to
provide something that could be genuinely useful and gives a hint at what could be
possible in the future.

## What?

### Serverless Infrastructure Management

In the traditional approach, you would be executing POSH to manage your infrastructure
from a Windows host -- and that's all well and good if you're doing one-off transactions
or interactive sessions.  But if you're running frequent tasks and automations, you would
normally need to have a server (or group of servers, redundancy, right?) for hosting your
scripts and executing on some set schedule or response to some event.

This is where Lambda comes in.  In the spirit of the whole *serverless* movement, Lambda
gives you the ability to define your logic, publish it with some configuration data, and
let it run.  You don't worry about the underlying sever provisioning, scaling and redundancy,
and you only pay for what you use, i.e. the execution time.  There's no need to manage and
maintain the host and OS, just worry about your code.

With PowerShell we get the 


## General Steps

- Build PS6 Core
- publish for target Runtime "centos.7-x64"
  e.g. `Start-PSBuild -Runtime centos.7-x64`
  
- add the `PackageManagement` and `PowerShellGet` modules into the publish/Modules
  folder
- capture the build output in the "publish" sub-dir as a "bin" dir
  e.g. src/powershell-unix/bin/Linux/netcoreapp2.0/centos.7-x64/publish
- Write a Node.js handler:
```javascript
const exec = require('child_process').exec;

exports.handler = function(event, context) {
    const child2 = exec('./bin/powershell ./main.ps1', (result) => {
        // Resolve with result of process
        context.done(result);
    });

    // Log process stdout and stderr
    child2.stdout.on('data', console.log);
    child2.stderr.on('data', console.error);
}
```
- Write your main.ps1:
```powershell
Write-Output "Hello World!  From PS6 on Lambda"
```
- Copy the libunwind libraries from lib64:
```
cp /usr/lib64/libunwind.so* .
```
- Layout the components into a folder structure:
  * `bin/` - contains the contenst of the PS6 publish folder from above
  * main.ps1
  * main.js
  * libunwind.so.8
  * libunwind.so.8.0.1
- combine into a zip:
```bash
zip -r ../ps6.zip ./bin ./main.ps1 ./main.js ./libunwind*
```

## Observations
* Shelling out from Node.js to do whoami -- less than 100ms
* With 256MB of memory:
  * Shelling out from Node.js to do powershell -version -- ~3000ms
  * Shelling out form Node.js to do powershell main.ps1 (the sample above) -- ~15000ms
* With 512MB of memory:
  * Shelling out from Node.js to do powershell -version -- ~1600ms
  * Shelling out form Node.js to do powershell main.ps1 (the sample above) -- ~7700ms
* With 1024MB of memory:
  * Shelling out from Node.js to do powershell -version -- ~850ms
  * Shelling out form Node.js to do powershell main.ps1 (the sample above) -- ~3800ms
* With 1536MB (MAX) of memory:
  * Shelling out from Node.js to do powershell -version -- ~550ms
  * Shelling out form Node.js to do powershell main.ps1 (the sample above) -- ~2600ms
  
  
## Considerations

AWS Lambda Limits (from the [doc](http://docs.aws.amazon.com/lambda/latest/dg/limits.html)):
AWS Lambda Deployment Limits

| Item                                                                                               | Default Limit
|----------------------------------------------------------------------------------------------------|--------
| Lambda function deployment package size (compressed .zip/.jar file)                                | 50 MB
| Total size of all the deployment packages that can be uploaded per region                          | 75 GB
| Size of code/dependencies that you can zip into a deployment package (uncompressed .zip/.jar size) | 250 MB
| Total size of environment variables set                                                            | 4 KB

Current ps6.zip file size is about 41MB
Current expanded ps6 size is about 120MB
AWSPowerShell Module (Windows) on disk is about 50MB
AWSPowerShell Module (Windows) compressed is about 7MB

## References

* https://aws.amazon.com/blogs/developer/introducing-aws-tools-for-powershell-core-edition/
* https://www.powershellgallery.com/packages/AWSPowerShell.NetCore

Other Infrastructure Management:
* https://msdn.microsoft.com/en-us/powershell/wmf/5.0/networkswitch_overview


## Something mildly more useful

Let's try this script:
```PowerShell
ipmo AWSPowerShell.NetCore
Get-LMFunctionList | select FunctionName
```

Before we get to Lambda, here are some timings to execute this directly on a VM of type `t2.micro`:
```
[ec2-user@ip-10-50-0-5 ~]$ time ./ps6/bin/powershell ./ps6/main.ps1

FunctionName
------------
TugDscLambda
test-TugDscLambda
TugDscLambda2
ps6
FixUpDscHeaders
dsc-service-DscPullLambda-WQBCHOCAEWHM



real    0m13.339s
user    0m12.776s
sys     0m0.424s
```

## Observations
* Initially the Lambda call failed with a timeout -- we had this configured with the default 30s timeout.
* Lambda allows up to 5m0s timeout, so we bumped it up to the max.
* With 512MB of memory:
  * List Lambda Functions -- ~42000ms, MaxMemoryUsed=174MB
* With 1024MB of memory:
  * List Lambda Functions -- ~21000ms, MaxMemoryUsed=174MB
* With 1536MB (MAX) of memory:
  * List Lambda Functions -- ~14200ms, MaxMemoryUsed=174MB
