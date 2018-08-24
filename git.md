- [Wiki home](/home)
- 


***
# Git Workflow for GIS Puppet Development

# Table of Contents
1. [Base Information](#base-information)
2. [Preparations](#preparations)
3. [Class Development](#class-development)
4. [Testing](#testing)
5. [Integrate your Development into Production](#integrate-your-development-into-production)

---

## Base Information
There are 3 Core Systems in the GIS Puppet Environment
- Puppet/Foreman for AIX: 
    - Foreman: http://lde5000p.de.top.com/users/login 
    - Gitlab Repository: http://gitlab.generali-gruppe.de:9000/ops/aix
- Puppet Foreman for Linux: 
    - Foreman: http://lde5001p.de.top.com/users/login 
    - Gitlab Repository: http://gitlab.generali-gruppe.de:9000/ops/linux
    - weekly generated class documentation: http://lde5001p.de.top.com/rdoc/index.html

**Allways work with your V-Key!!!**

## Preparations
### Create your environment
To build your own test environment:
As root:
 - Change into Puppet environment directory (`cd /etc/puppet/environments`)
 - Clone the `master` branch into your environment:
     - AIX:`git clone git@gitlab.generali-gruppe.de:ops/aix.git v999999`
     - Linux `git clone git@gitlab.generali-gruppe.de:ops/linux.git v999999`
     - This will create `/etc/puppet/environments/v999999`
 - Change owner of the created directory to puppet:adm_IB.
 - Change into the created directory (`cd v999999`)
 - Change owner of all content to your v-key and adm_IB. Don't forget the .git directory
    - e.g.  `chown -R v999999:adm_IB * .git*`
 - Log into Foreman and Import the new environment via **Configure -> Environments -> Import from $Servername**
 - Select your new Environment.

To use mcollective you need to create a copy from the dev config files
  - `cp -R modules/gis_rhel6_apps/files/sc_mcollective/dev  modules/gis_rhel6_apps/files/sc_mcollective/$envName`

Alternativly you could also import the environment over the console:
 - Restart foreman-proxy and httpd service
 - Create environment in Foreman (`hammer environment create  --name v999999`)
 - Import Puppet classes into the new environemnt (`hammer proxy import-classes --environment v999999 --id 1`)
 
### Adding test systems to your environment

Before you can start with the development in your environment you have to add one or more test systems. You can do this directly in Foreman on the system properties or via console:
~~~sh
$ hammer host update --environment v999999 --name tpm86.de.top.com
Host updated
~~~
To choose your test system go to http://gis-wiki/index.php/Linux_Testsysteme
, check for an available system and update the wiki site with your new information.


## Class Development
### Create a new locale Dev Branch
To start the work you need a branch.  (What is a branch? http://gitlab.generali-gruppe.de:9000/gss-cin-unx-documentation/git/wikis/branching )

There are two ways to get your branch: 

**Create a new branch** or **clone an existing branch** from the gitlab server.

The choice which way you create your branch depends on if your working on an existing development or not. If you start from scratch you create a new branch.

Do the following to create your branch:
```sh
# switch int your environment
$ cd /etc/puppet/environments/new_env
# at first you need to ensure that your master branch is up-to-date
$ git checkout master
$ git pull
# create your branch. Your branchname should represent the purpose of your development (e.g. fix-apache or rhel7-base)
$ git branch testbranch
# switch into your branch
$ git checkout testbranch
# push branch into Gitlab; after that "git push" will push also your branch onto the gitlab server
$ git push -u origin testbranch
```
Now you can start developing and testing on your testsystem. You don't need to commit your changes for your tests.
Be aware that puppet always uses the current branch checkout. This is important if you are working with multiple branches and testsystems in one environment.

### Prepare your Development
If you want to change or extend an existing class you can start working. 
If you want to create a new class you need to create the necesarry files:

#### AIX
- Before you start developing, Familiarize yourself with the AIX puppet directory structure (BESCHREIBUNG AN ANDERER STELLE DOKUMENTIEREN UND VERLINKEN)
- Import your new class into Foreman [see Change Class Parameters](#change-class-parameters)
- If you have changeable class parameters, change these in Foreman to overrideable. [see Change Class Parameters](#change-class-parameters)

#### Linux
- Before you start developing, Familiarize yourself with the linux puppet directory structure (BESCHREIBUNG AN ANDERER STELLE DOKUMENTIEREN UND VERLINKEN AUCH NAMENSREGELN)
- Create your class file based on the class template snippet: http://gitlab.generali-gruppe.de:9000/ops/linux/snippets/2 (copy paste)
- Import your new class into Foreman [see Change Class Parameters](#change-class-parameters)
- If you have changeable class parameters, change these in Foreman to overrideable. see [see Change Class Parameters](#change-class-parameters)

**Note**: You can also use the Puppet build-in command `puppet module generate` to build your module
structure ([Generating a Puppet module](/module_generate))

#### Filesystem structure of a Puppet module

For a functional Puppet module you need a structure like shown below. You should always have a **README.md**  and you **must** always have a `init.pp` where your classes are defined. Normaly you also will use the `files` and `templates` directories.

~~~
gis-aix_app_db2
├── files
│   ├── file1
│   └── file2
├── Gemfile
├── manifests
│   └── init.pp
├── metadata.json
├── Rakefile
├── README.md
├── spec
│   ├── classes
│   │   └── init_spec.rb
│   └── spec_helper.rb
├── templates
│   └── template1.erb
└── tests
    └── init.pp
    
Sample filesystem structure of a Puppet module
~~~


### Things you should know and do during your development

#### Change Class Parameters
- If you add or delete class Parameters, you need to (re)import your class in Foreman. *It will take a moment after you clicked on Import*

<center>
![Import1](images/foreman/foreman-class-import-001-600.png)
</center>


- Now you can see which classes are new or have changed. Check the boxes for **your environment** and click Update. 
<center>
![Import2](images/foreman/foreman-class-import-002-600.png)
</center>

- If necesarry make parameters overrideable in Foreman.
- You can later override parameters on individual host basis or in hostgroups

<table>
<center>
![Params1](images/foreman/foreman-parameter-overide-600.png)
</center>
</table>

> **Use the [Puppet Styleguide](/styleguide) for your development to ensure clean and functional puppet code!**


#### Commit your code
> Commit often but only when you think it makes sense! Do not commit in the middle of your work. Test your stuff before commiting.

> Provide a useful commit message because it helps you in identifying what you changed in that commit. Avoid overly general messages like “Fixed bugs”. If you have an issue tracker, you could provide messages like “Fixed bug #234”. 

See [ Git-Config-Usage-Guide.markdown](http://gitlab.generali-gruppe.de:9000/workshops/puppet-git/blob/master/Git-Config-Usage-Guide.markdown)

#### Keep your development branch up to date
If your work takes several days (and it will take several days) it is possible that the master branch changes during this time. So you need to sync these updates into your development branch.
```sh 
git checkout master
git pull
git checkout testbranch
git merge master
```
If the master conflicts with your development you will get a merge conflict. But this should only happen if someone else is working on the same files as you are in your development. If this happens you should consult the git documentation. (https://git-scm.com/docs/git-mergetool) 


#### Use the right documentation
If you make use of the official puppet documentation (https://docs.puppet.com/puppet/) make sure that you choose the right version of the documentation.

## Testing

### Check and validate your class syntax

#### Manual Checks

Before you test your class you should check your syntax for possible error. You can manually test you class with `puppet parser validate yourclass.pp` to check for syntax error and `puppet-lint yourclass.pp` to check for stylguide and format validation.

### Test on Testsystem
- classifiy your testsystem in the appropiate hostgroup in Foreman LINK ZUR HG BESCHREIBUNG
- assign the class you want to test on your testsystem (only necesarry if it is not already in the hostgroup) 
- set class parameters if needed
- Go on your testsystem and run 
```puppet agent -t --no-noop```
- check the log output from the puppet run and fix any errors
- Consider and test possible conflicts with other classes. e.g. Duplicate declaritons, dependencies or order of applied classes (Use Run Stages to avoid order problems).

> You should test before you commit. **Never commit broken code!**

#### Automatic Pro-Commit Checks (currently Linux only)
All modified and/or added files will be checked for syntax and formatting errors before a git commit. As long as the syntax errors are not fixed a commit is not possible. Formating errors will be reported but not enforced.

Example failure for a syntax error:
```bash
[v073385@lde5001p:/home/v073385/git/linux/modules/gis_rhel6_apps/manifests]$ git commit -m "test commit with syntax error"
Error: Could not parse for environment production: Syntax error at 'forcelocal'; expected '}' at /home/v073385/git/linux/modules/gis_rhel6_apps/manifests/clt_patrol.pp:41
Error: puppet syntax error in modules/gis_rhel6_apps/manifests/clt_patrol.pp (see above)
Error: 1 syntax error(s) found in puppet manifests. Commit will be aborted.
rspec not installed. Skipping rspec-puppet tests...
r10k not installed. Skipping r10k Puppetfile test...
Error: 1 subhooks failed. Please fix above errors.

```


## Integrate your Development into QAS and Production

### Merge your branch
- push your finshed and tested branch on the gitlab server
- Create a merge Requst in GItlab. You can comment the request and see the changes you made. 
![Merge1](images/git/merge_request_001.png)

- Select *your branch* as source and the *master branch* as target:
![Merge1](images/git/merge_request_002.png)

> When the request is accepted, you branch will be merged into the master.

### Staging your code into dev, qas and prd environments
- Accepted merge request will be pulled into the dev environment by the responsible assignee of the merge request. 
- For the further stageing please contact the responsible assigne.
- if don't contact the asignee, you canges will be pulled into dev/qas/prd during the staging of another acceppted merge request.

### Delete your dev branch
After your branch is merged and there is no futher use for your branch, you can delete it localy and on the Gitlab server.

```sh
# pull the new master branch
$ git checkout master
$ git pull
# delete branch remotely
$ git push  origin :<your-branch-name>
# delete branch locally 
$ git branch -d <your-branch-name>
# cleanup remote branch listing
$ git remote prune origin
```


