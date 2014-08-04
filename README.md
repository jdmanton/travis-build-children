# travis-build-children
**Executive summary:** After a successful Travis build of a project, trigger a rebuild of a further project that depends on the first.

Currently, [Travis CI](https://travis-ci.org/) does not provide a way to [trigger rebuilds when dependencies are updated](https://github.com/travis-ci/travis-ci/issues/249). This script does not solve this issue, but provides a work-around for some cases by solving the inverse problem, namely rebuilding projects that depend on a parent project after a successful build of the parent. This requires that both the parent and child projects are owned by the same user/organisation, and so some forking of parent repositories may be required depending on your particular use case.

This is an unofficial tool, but has been successfully used in [nat](https://github.com/jefferis/nat) to trigger rebuilds of [flycircuit](https://github.com/jefferis/flycircuit), [nat.templatebrains](https://github.com/jefferislab/nat.templatebrains), and [nat.nblast](https://github.com/jefferislab/nat.nblast), and further in [nat.templatebrains](https://github.com/jefferislab/nat.templatebrains) to trigger a rebuild of [nat.flybrains](https://github.com/jefferislab/nat.flybrains).


## Usage
In order for builds to be triggered, it is necessary for the script to know your Travis access token so that it can authenticate successfully. The easiest way to obtain this is to [use the Travis CI command-line tool](https://github.com/travis-ci/travis.rb#token):
```
travis login
travis token
```
As adding the raw token to a publicly-accessible file would be a bad idea, you should now encrypt this and store it as a [secure environment variable](http://docs.travis-ci.com/user/encryption-keys/):
```
travis encrypt AUTH_TOKEN=<your_travis_token>
```
Now that you have your encrypted token, you should add it to your parent project's ``.travis.yml`` file, and ensure that the ``build_children.sh`` script is called after a successful build:
```
env:
	global:
		- secure: <your_encrypted_token>
after_success:
	- chmod 755 ./build_children.sh
	- ./build_children.sh

```
Finally, you should copy ``build_children.sh`` into your parent project's repository and edit it so that ``<username>`` and ``<child_project_name>`` are set to the correct values in:
```
BUILD_NUM=$(curl -s 'https://api.travis-ci.org/repos/<username>/<child_project_name>/builds' | grep -o '^\[{"id":[0-9]*,' | grep -o '[0-9]' | tr -d '\n')
```
Once this is done, a successful build of your parent project should now trigger a rebuild of your child project.


## Troubleshooting
### Help! I can't use the Travis command-line tool!
This means you are going to have to obtain your Travis access token and encrypt it manually. Any erroneous spaces, etc., will cause failure, so this may take a few attempts...

First, use a web browser to visit https://travis-ci.org and login. Once done, trigger a rebuild of one of your repositories by clicking the restart button on its overview page. By sniffing the HTTP request (for example, through Google Chrome's [Developer Tools](https://developer.chrome.com/devtools/index)), you will find a request header of the form:
```
Authorization: token 81aEalkjaEBR_N_alkjJNE
```
This is your [Travis access token](http://blog.travis-ci.com/2013-01-28-token-token-token/). Entering this directly into a publicly-accessible file would be a bad idea, so we should encrypt it first. For this, we'll require the parent project's public key, accessible through the API at ``https://api.travis-ci.org/repos/<username>/<repo_name>/key``. A request to this will produce a response of the form:
```
{"key":"-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC9y54f0+97vlwQuYb6taKEnFk5\nx1IunyU5/z8EEEXFzAztTLKmqDLWZKdsmjBlGDeLKq7Fn+DgRceH1b2Y8E0Mdsfk\nB9MRZXFcEJ8BcNo12RI5kISEFznhwutvzt27bhSOJiCg1sHAJX5Z6xCbU+3w32d4\ncY983v1X+cWRVUdrQwIDAQAB\n-----END PUBLIC KEY-----\n","fingerprint":"ab:74:15:9b:a5:f4:a1:68:8b:fe:ec:1f:5c:80:9a:00"}
```
We're only interested in the key, and so this should be reformatted into something that looks like:
```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC9y54f0+97vlwQuYb6taKEnFk5
x1IunyU5/z8EEEXFzAztTLKmqDLWZKdsmjBlGDeLKq7Fn+DgRceH1b2Y8E0Mdsfk
B9MRZXFcEJ8BcNo12RI5kISEFznhwutvzt27bhSOJiCg1sHAJX5Z6xCbU+3w32d4
cY983v1X+cWRVUdrQwIDAQAB
-----END PUBLIC KEY-----
```
and saved to a file, say called ``repokey.pem.pub``. Now you should save a file, say called ``token.txt``, containing your Travis access token, in the form:
```
AUTH_TOKEN=81aEalkjaEBR_N_alkjJNE
```
and then encrypt it using:
```
openssl base64 -in <(openssl rsautl -encrypt -pubin -inkey repokey.pem.pub -ssl -in token.txt)
```
This will return something of the form:
```
OetTChiTrKWAB9fmvu/hxpwP3HgdI/uV5CvD4xvm24ynVcZ14S6D9aWtdzZNRK7J
xDjaHf7Q2q6GflXJewuVaNK4lDcZNUygDZH4eA1zMbSlIIa+QrVEFh+Nr3skWQ3s
6Ul8g9YoQJbFoACjf2muVmNFu75U9AQv0+xWMaXhksI=
```
which you should use to modify your ``.travis.yml`` file as detailed above.


### Hey! This triggers a rebuild for projects that depend on mine, and doesn't trigger a rebuild of mine when a dependency updates!
This is detailed in the very first paragraph of the README. I can only suggest adding a comment to [this issue](https://github.com/travis-ci/travis-ci/issues/249) on Travis' issue tracker.
