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
