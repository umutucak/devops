# 2- Automate build process


## Assumptions/Pre-requisites

### Software
1. Ansible (v 2.7.5, or higher)
* Instructions to install here: https://docs.ansible.com/
* Check installation with the command `ansible --version`


## Create integration server

The goal is to create a VM that acts as integration server. <br>
At this stage the integration server should be used to hold a VCS where developers 
can publish their changes. Moreover, every time a change is detected the build process 
should be triggered. 

1- Get to the working directory

`cd ~/<git_root_folder>/devops/pipeline/s2-automate-build/integration-server`

2- Vagrant is used to create a VM which acts as integration server.

`vagrant up`

**Notes**

* The VM is automatically provisioned using Ansible playbooks. 

* These playbooks install GitLab and Docker. 

* GitLab is used as VCS and CI, whereas Docker is used to handle the integration environments.


**Exercise 1**

* 1.a Analyse how the provisioning is done.

delivered through an ansible playbook. config defines 2 [roles](https://docs.ansible.com/projects/ansible/latest/reference_appendices/playbooks_keywords.html#role), for gitlab and docker. they then execute their own installations through [tasks](https://docs.ansible.com/projects/ansible/latest/reference_appendices/playbooks_keywords.html#task). the tasks are calling [modules](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/), predefined sets of instructions that can be structured into yaml.

* 1.b Is there any "qualities" being addressed by the way the provisioning is done? Justify. 

**repeatability** & **[idempotency](https://www.compilenrun.com/docs/devops/ansible/ansible-best-practices/ansible-idempotency/)**: found this word while reading about ansible. i'd say this is the biggest due to how playbooks work if there is nothing to update. they do not necessarily target actions, but the outcomes of those actions. changes are made if we arent in the target state we want to be in. the article extends idempotency to more qualities -> **safety**, **reliability**, **predictability**, and **efficiency**. i really like the idempotency of ansible, but would argue that many of these qualities the article proposes are already provided by the notion on provisioning.

**maintainability**: provisioning is delegated outside of the vagrantfile, now all provisioning tasks can be handled in isolation without touching the rest of the vagrant config which touches vm configuration too (networking, box, etc).

**simplicity**: instead of writing lots of bashcommands into the vagrant file, we use a higher level management system through the playbook. this also aids in maintainability, as bash commands/CLI interactions can be deprecated, which could potentially be better maintained by the ansible module maintainers.
 
* 1.c Add package "nmap" to the playbook.

first, confirmed that nmap was on `apt` via their [website](https://nmap.org/book/inst-linux.html). although main site recommends to build from source, `apt` offers the package. then, all i had to do was install via the [apt module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/apt_module.html#ansible-collections-ansible-builtin-apt-module) using the [name parameter](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/apt_module.html#parameter-name). to confirm: 1) `vagrant ssh`, 2) `nmap --version`

kept the `apt update` & `apt upgrade` steps, even though they are redundant and are ran by gitlab and docker already. i kept it intentionally to not build dependencies between the roles. we could drop docker and gitlab one day, then those steps would be omitted. if i were to touch them: i would keep `update`, so that `nmap` fetches its own updated package, and would drop `upgrade`, as i dont think it's `nmap`'s task to upgrade system packages. if we want `nmap` to keep itself up to date, i would add `state: latest` to it.

## Configure GitLab

The goal is to make GitLab accessible as a service.

Note that the installation of GitLab was already made upon starting of the VM (via provisioning).
It only remains to make its configuration.


1- Edit `/etc/gitlab/gitlab.rb` and replace 


`external_url http://hostname`

by

`external_url 'http://192.168.56.9/gitlab' `

and

`# puma['port'] = 8080`

by

`puma['port'] = 8088`


(Don't forget to uncomment)



2- Reconfigure and restart gitlab

`sudo gitlab-ctl reconfigure`

`sudo gitlab-ctl restart puma`

`sudo gitlab-ctl restart`


3- Connect to GitLab

Open `http://192.168.56.9/gitlab` in your browser.

You will be asked to provide a password (refered as $YOUR_PASSWORD later) for the root credentials.

To further login with root the credentials are:<br>
Login: root<br>
Password: $YOUR_PASSWORD<br>

**i was not shown this root setup page, so i have my default root pass: `rrLa2/oBs4kvdstScR0gJ8O5F1TfcquZr18gp21dsFI=`**

 
   
**Notes:** 

* Upon starting the service up yet another user is requested to be created. 
Keep the credentials of this new user, as you'll need it for creating repositories.

new user credentials
login: test
password: this1password2

* In case you need to reset the password of a GitLab user follow the instructions given in this [link](https://docs.gitlab.com/user/profile/user_passwords/)


## Configure Docker

The objective of this step is to be able to use the CLI with the current user ($YOUR_USERNAME = vagrant).<br>
Notice that the basic docker daemon and docker CLI have been already installed on the VM upon starting (via provisioning).

1- Add a user to the docker group ot be able to access the docker CLI

`sudo usermod -aG docker $YOUR_USERNAME`

**not actually needed, this ran during provision**


2- Validate the installation and access by running a hello word container.

`docker run --name hello-world hello-world`


The expected output should be.

`Hello from Docker!`

**done**

3- Remove the docker container and image:

`docker rm hello-world`

`docker rmi hello-world`

**done**


## Use GitLab as VCS

The goal of this step is to make use of GitLab as a VCS. 

1- Create HelloWorldMaven project (in GitLab a project is a repository).<br>
Follow the instructions to create the remote and local repositories.


2- Commit and push to the remote repository.


**Notes:** 

* do NOT use the GitLab's root user to create the repo and to push to it<br>

* use the project placed at:<br>
`~/<git_root_folder>/devops/pipeline/s1-create-skeleton/MavenHelloWorldProject`

* ensure the file `pom.xml` is at the root of the local repository. 

* create a file '.gitignore' to avoid committing logs, dev settings, and binaries. 


done: https://imgur.com/a/OSFkg2p



## Automate build

The objective of this stage is to configure the GitLab project such that
every time a developer publishes a change on the remote repository, the build process is 
automatically started. More precisely, upon detection of a change on the remote repository, GitLab
has to:

1. check out/update source code
2. run the automated build script
3. store the binaries where they can be accessed by the team. 



### [GitLab Runner](https://docs.gitlab.com/runner/) setup



#### Install Git Lab Runner

1. Ensure you are in the directory<br>
`~/<git_root_folder>/devops/pipeline/s2-automate-build/integration-server`

2. Then ssh to the VM by doing

`vagrant ssh`

3. Once you have entered into the VM, you need to install the GitLab Runner on the integration server.
This is done by executing the following commands:

`curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash`

`sudo apt-get install gitlab-runner`



#### Register the Runner

1. First of all, you need to execute the following command:

`sudo gitlab-runner register`

2. Then enter the requested information as follows:

For GitLab instance URL enter:<br>
`http://192.168.56.9/gitlab/`


For the gitlab-ci token enter the generated token.<br>
~~`Example: 85y84QhgbyaqWo38b7qg`~~
**mine is `DxrrfEYtXq_DMuBv-cDG`**

For a description for the runner enter:<br>
`[integration-server] docker`

For the gitlab-ci tags for this runner enter:<br>
`integration`

For the executor enter:<br>
`docker`

For the Docker image (eg. ruby:2.1) enter:<br>
`alpine:latest`


3. Restart the runner:

`sudo gitlab-runner restart`


4. Finally, in GitLab change the configuration of the runner to accept jobs without TAGS.

done: https://imgur.com/a/srIcKsV

**Exercise 2** 

* 2.a Collect the manual steps you have done up to now. 

edit gitlab config file

reconfigure and restart gitlab


* 2.b List those that could be added to a playbook.

**edit gitlab config file** -> automated

**reconfigure and restart gitlab** -> automated

* 2.c Add (at least) half of them to either a new or existent playbook.



### Create GitLab CI

1. Create file named .gitlab-ci.yml at the root of the repository.

2. Add the following lines into the file:

```
image: maven:latest

stages:
  - build
  - test
  - run

cache:
  paths:
    - target/

build_app:
  stage: build
  script:
    - mvn compile

test_app:
  stage: test
  script:
    - mvn test

run_app:
  stage: run
  script:
    - mvn package
    - mvn exec:java -Dexec.mainClass="com.jcg.maven.App"

```

**Exercise 3** 

* 3.a Use GtiLab to analyse what has just happened?<br>
**Tip:** have a look on the CI/CD -> Pipelines of the project.

* 3.b Try to run again the CI and inspect the intermediate results of each stage.

* 3.c Where is the .jar file generated as result of the build?



### Storing the "binary"

The last step of the automatic build is to store the resulting artefact such that it 
can be accessed by the team. 

Add to the file .gitlab-ci.yml (in the stages section, afer "- run") the line:

`- deploy` 

and, add at the bottom of the file the following block

```

deploy_app:
    stage: deploy
    script:
    - echo "Deploy review app"
    artifacts:
        name: "my-app"
        paths:
        - target/*.jar
        
```
	

**Exercise 4**
 
* 4.a Find out what is the result of this modification.<br>
**Tip:** have a look on the CI/CD -> Pipelines of the project.






   
