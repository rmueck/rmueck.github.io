Automation Websphere Application Server (WAS)
============================================

Einleitung
----------

Ziel dieser Automation ist die Bereitstellung eines IBM WebSphere Application Servers (Version 8.5 und 9.0) auf einem RHEL 7 Linux System. Zur Implementation werden Puppet und Rundeck eingesetzt.


[gimmick:gist](5641564)



Ein weiterer Teil der Automation beinhaltet das sog. Redeployment, bei dem die VM neu installiert wird, die zweite Hardisk ebenfallls initialisiert (gelöscht) wird und anschließend eine Neuinstallation des WAS erfolgt.

Die Automation umfasst die folgenden Aspekte:

- Anlegen von Gruppen und Usern sowie sudo Berechtigungen
- Bereitstellung einer WAS spezifischen Systemkonfiguration
- Bereitstellung von:
  - Volumegroup (WASvg)
  - Logical Volumes
  - Filesystemen
  - Mountpoints
- Installation und Update des IBM Installationmanagers (IM)
- Installation des WebSphere Application Servers in den Ausprägungen
  - old
  - previous
  - current
  - unstable

>Die Versionierung und das Stagen dieser Ausprägungen liegt in der Verantwortung von OCP-MDC 

>Die benötigten Script Resourcen  werden von OPC-MDC mittels eines Git Repositories zur Verfügung gestellt und auf dem Rundeck Server via pull aktualisiert.


Implementation
--------------


Die Implementation der Automation erfolgt zweigeteilt und zwar im ersten Schritt mittels Puppet und im zweiten Schritt mittels Rundeck Jobs.

Der Gesamtprozess wird durch Camunda gestartet und mithilfe der REST APIs die von Foreman und Rundeck getriggert und der Status überwacht.

In der Regel erfolgt diese Zuordnung über einen API Call von Camunda aus indem das System in die Hostgruppe `/global/rh7/[dev|qas|prd]/default/role_was`aufgenommen wird.

Nach Aufnahme in die Hostgruppe muss ein Puppetrun durch einen REST Call gegen die Rundeck API erfolgen. Nach erfolgreicher Durchführung des Puppetruns werden die notwendigen Rundeck Jobs durch Camunda getriggert.


Puppet Module/Klassen
--------------------


Im folgenden erfolgt eine kurze Beschreibung der verwendeten Puppet Klassen. Im Anhang befindetet sich der Quellkode der Klassen.

#### `gss_serv_websphere`

Dies ist das Zentrale Puppet Modul, über das die Reihenfolge der abhängigen Module gesteuert wird und nötige Parameter gesetzt werden. Soll auf einem System WAS installiert werden, so wird **nur diese** Klasse dem System zugeordnet.

#### `gss_srv_websphere::params`

In dieser Klasse werden z.B die benötigten `rpm`Pakete definiert.

#### `gss_srv_websphere::groups`

Implementiert die für Websphere notwendigen groups. Diese werden mittels `hiera` ermittelt.

####`gss_srv_websphere::user`

Diese Klasse implementiert die für Websphere notwendigen user. Diese werden mittels `hiera` ermittelt.

#### `gss_srv_websphere::storage`

Storage

#### `gss_srv_websphere::ibminstallationmanager`

Transferiert den IM vom Repository auf `lde0002p` auf das Zielsystem entpackt es dort und führt die Installation sowie das Update auf die neueste Version durch.


Rundeck Jobs
------------

Die Aufgabe von Rundeck in der Automation umfasst folgende Jobs:

- Aktualisieren des Installationsscripts im Git Repository 
- Transfer des Installationsscripts auf das Zielsystem
- Auführung des Installationsscripts mit den Parametern
  - WAS Version (old,previous,current und unstable)
  - WAS Release (8.5 und 9.0)

#### `01_sync_repository`

Aktualisiert das Installationsscript auf dem Rundeck Server im Pfad `/opt/repo/linux`

#### `02_tranfer_script`

Kopiert das Installationsscript auf das Zielsystem in den Folder `/root`.

#### `03_fix_permissions`

Fix permissions (`+x` Flag)

#### `04_perform_installation`

Implementiert die Installation des WAS und alle weiteren benötigten Schritte mittel IBM Installationmanager (IM)



Störfallprozess und Verantwortlichkeiten
----------------------------------------

| Resource                                           | Verantwortlich | Anmerkung                                                   |
| -------------------------------------------------- | -------------- | ----------------------------------------------------------- |
| Puppet Klassen                                     | GIS-CIN-UNX    |                                                             |
| Rundeck Jobs                                       | GIS-CIN-UNX    |                                                             |
| Installationscript                                 | OCP-MDC        | Im Störfall des RD Jobs `04_perform_installation`           |
| Staging und Generierung der Versionen und Releases | OCP-MDC        | old, previous, current und unstable sowie die WAS Versionen |
| Bereitstellung IM                                  | OCP-MDC        |                                                             |
| Prozessdesign und Implementation                   | OCP-MDC        | Camunda                                                     |



## Anhang


Hiera
-----


Die benötigten Gruppen und User werden mittels Hiera Lookup ermittelt. Wird einem System die Rolle `websphere` zugeordnet (*erfolgt durch Aufnahme in die Hostgroup*), so ist Puppet in der Lage die gewünschten Daten über einen automatischen Lookup zu ermitteln. 

~~~yaml
---
gss_srv_websphere::groups::groups:
  wasadm:
    ensure: present
    forcelocal: true
    gid: 304

gss_srv_websphere::users::users:
  wasadm:
    ensure: present
    forcelocal: true
    uid: 398
    gid: 304  
    home: /home/wasadm
    managehome: true  
    password_max_age: -1
    password_min_age: -1
    shell: /bin/bash
    groups:
      - initdadm
    loglevel: debug
    
gss_srv_websphere::users::logingroups:
  - kvwbtr
  - wbtr
~~~



Facts
-----

~~~ruby
# gis_im_installed.rb
Facter.add('gis_im_installed') do
    setcode do
        if File.exists? '/opt/IBM/InstallationManager/eclipse/tools/imcl'
            'true'
        else
            'false'
        end
    end
end 
~~~



Puppet Klassen
--------------

#### `gss_srv_websphere::params`

~~~ruby
# == Class gss_srv_websphere::params
#
# This class is meant to be called from gss_srv_websphere.
# It sets variables according to platform.
#
class gss_srv_websphere::params {
  case $::operatingsystemmajrelease {
    '7', '6': {
      $package_name =
        ['compat-libstdc++-33',
        'glibc',
        'glibc.i686',
        'compat-db47',
        'psmisc',
        'ksh']
      $service_name = 'gss_srv_websphere'
    }
    default: {
      fail("Operatingsystem release ${::operatingsystemmajrelease} is not supported")
    }
  }
}
~~~

#### `gis_app_websphere::groups`

~~~ruby
# == Class: gis_app_websphere::groups
#
# Manage the creation of local groups related to Websphere Application Server (WAS)
#
class gss_srv_websphere::groups (
$groups         = hiera_hash('gss_srv_websphere::groups::groups'),
$stage          = 'os-post',
){
create_resources('group',$groups)
} 
~~~



#### `gss_srv_websphere::users `

~~~ruby
# == Class: gss_srv_websphere::users
#
class gss_srv_websphere::users (
  # Users are derived from hiera
  # The system must have the role websphere assigned in Foreman!
  $users             = hiera('gss_srv_websphere::users:users'),
  $logingroups       = hiera('gss_srv_websphere::users:logingroups'),
  $stage             = 'os-post',
){

create_resources('user',$users)

# execute password generator script
exec { 'create_passw_wasadm':
  command => '/root/scripts/setPassword.sh wasadm',
  creates => '/home/wasadm/.donttouchthis',
  } ->

exec { 'create_passw_logrdr':
  command => '/root/scripts/setPassword.sh logrdr',
  creates => '/home/logrdr/.donttouchthis',
  } ->

# Create needed files & directories
file { '/home/wasadm/.profile':
  ensure =>  'link',
  target => '/home/wasadm/.bash_profile',
  group  => 'wasadm',
  owner  => 'wasadm',
  } ->

  file { '/home/wasadm/.ssh':
  ensure => 'directory',
  group  => 'wasadm',
  owner  => 'wasadm',
  mode   => '0700',
  } ->

ssh_authorized_key { 'sshkey_wasadm':
  ensure  => 'present',
  require => File['/home/wasadm/.ssh'],
  name    => 'wasadm',
  key     => 'AAAAB3NzaC1yc2EAAAADAQABAAABAQC5tuMKVlseSZB2vWzNIKLyBLW/SyovtdVCC+bdwMRWgLBXAJSj8/t2vyVUa/wLA13LD5HN+NW86hFVj7pvwHpvatXWvO2RLfh8agRq89KLj9qKzUs7i91sdWxwQJDR+ZXygDzKvkFI56+O8ocfNLdTod3TDLSI0W409p8v3MRRAZII7nRbNC/fNRBahEMRDD8J9eBW97n5z4hUnLV/ZQyt4ki8bfowjB8NPjk7kDVzys6gepawDZwFuRAR9R+luw8DPDibFkVNqzw0lUPn90WGcZ2LiGUVax7BruTt4PLNEU3zrW5jAmk9Z4+hnnnn+Nwh9gyJT+jv3oq/XwcDI4Oz',
  type    => 'ssh-rsa',
  user    => 'wasadm',
  } ->

# Allow groups defined in hiera to login
manage_login { $logingroups :
  ensure => 'present',
  } ->

# sudoers config for Websphere admins
file { '/etc/sudoers.d/60-websphere':
    ensure  => 'file',
    content => template('gis_standards/puppetwarn_hash.erb','gss_srv_websphere/etc/sudoers.d/60-websphere.erb'),
    mode    => '0440',
    owner   => '0',
    group   => '0',
} ->

# limits config for Websphere
file { '/etc/security/limits.d/60-websphere.conf':
ensure  => 'file',
  content => template('gis_standards/puppetwarn_hash.erb','gss_srv_websphere/etc/security/limits.d/60-websphere.conf.erb'),
  mode    => '0644',
  owner   => '0',
  group   => '0',
  }
} 
~~~



#### `gss_srv_websphere::storage `

~~~Ruby
# == Class:  gss_srv_websphere::storage
#
class gss_srv_websphere::storage(
  $stage            = 'app-pre',
  $is_disabled      = false,
  $uninstall        = false,
  $vgname           = 'WASvg',
  $physical_volumes = [ '/dev/sdb' ],
  $waslv            = '10G',

){
# don't change anything if the class is disabled
if ($is_disabled == false){

# Create volume and directory structure
volume_group { $vgname :
  ensure           => present,
  createonly       => true,
  physical_volumes => $physical_volumes,
} ->
#########################################
    logical_volume { 'waslv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => $waslv,
      #require      => Volume_group['WASvg'],
    } ->
    logical_volume { 'wasbaklv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '1G',
    } ->
    logical_volume { 'ibmimlv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '1G',
    } ->
    logical_volume { 'ibmimsharelv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '3G',
    } ->
    logical_volume { 'dynatracelv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '1G',
    } ->
    logical_volume { 'wasdatlv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '1G',
    } ->
    logical_volume { 'waslogslv':
      ensure       => present,
      volume_group => 'WASvg',
      initial_size => '1G',
    } ->

    filesystem { '/dev/WASvg/waslv':        ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/wasbaklv':     ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/ibmimlv':      ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/ibmimsharelv': ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/dynatracelv':  ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/wasdatlv':     ensure  => present, fs_type => 'ext4',} ->
    filesystem { '/dev/WASvg/waslogslv':    ensure  => present, fs_type => 'ext4',} ->

    exec { 'create_opt_was': command => '/bin/mkdir -p /opt/was', creates => '/opt/was', } ->
    mount { '/opt/was':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-waslv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/was': ensure => 'directory', group => 'wasadm', mode => '0755', owner => 'wasadm',} ->

    exec { 'create_opt_wasbackup': command => '/bin/mkdir -p /opt/wasbackup', creates => '/opt/wasbackup', } ->
    mount { '/opt/wasbackup':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-wasbaklv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/wasbackup': ensure => 'directory', group => 'wasadm', mode => '0755', owner => 'wasadm',} ->

    exec { 'create_opt_ibminst': command => '/bin/mkdir -p /opt/IBM/InstallationManager', creates => '/opt/IBM/InstallationManager', } ->
    mount { '/opt/IBM/InstallationManager':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-ibmimlv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/IBM/InstallationManager': ensure => 'directory', group => '0', mode => '0755', owner => '0',} ->

    exec { 'create_opt_ibmims': command => '/bin/mkdir -p /opt/IBM/IMshared', creates => '/opt/IBM/IMshared', } ->
    mount { '/opt/IBM/IMshared':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-ibmimsharelv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/IBM/IMshared': ensure => 'directory', group => '0', mode => '0755', owner => '0',} ->

    exec { 'create_opt_dynatrace': command => '/bin/mkdir -p /opt/dynatrace', creates => '/opt/dynatrace', } ->
    mount { '/opt/dynatrace':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-dynatracelv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/dynatrace': ensure => 'directory', group => 'wasadm', mode => '0755', owner => 'wasadm',} ->

    exec { 'create_opt_data': command => '/bin/mkdir -p /opt/data', creates => '/opt/data', } ->
    mount { '/opt/data':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-wasdatlv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/data': ensure => 'directory', group => 'wasadm', mode => '0775', owner => 'wasadm',} ->

    exec { 'create_opt_datalogs': command => '/bin/mkdir -p /opt/data/logs', creates => '/opt/data/logs', } ->
    mount { '/opt/data/logs':
      ensure  => 'mounted',
      device  => '/dev/mapper/WASvg-waslogslv',
      dump    => '1',
      fstype  => 'ext4',
      options => 'defaults',
      pass    => '2',
      target  => '/etc/fstab',
    } ->
    file { '/opt/data/logs': ensure => 'directory', group => 'wasadm', mode => '0755', owner => 'wasadm',}

    # file { '/opt/dynatrace': ensure => 'link', target => '/opt/wily'}

# end of volume and directory structure
#########################################

  } # close is_disabled
} # close class 
~~~



#### `gss_srv_websphere_ibminstallationmanager `

~~~ruby
# == Class: gss_srv_websphere_ibminstallationmanager
#
# === Authors
#
# Author Name <ruediger.mueck@generali.com>
#
# === Copyright
#
# Copyright 2018 GSS/Ruediger Mueck
#
class gss_srv_websphere::ibminstallationmanager (

    # general variables
    $im_basedir         = '/opt/IBM/InstallationManager',
    $im_tooldir         = '/opt/IBM/InstallationManager/eclipse/tools',
    $im_respdir         = '/opt/IBM/InstallationManager/eclipse/responsefiles',
    $im_dir             = '/opt/IBM/InstallationManager',

    # repository
    $repo_baseurL       = 'http://lde0002p.de.top.com/pub/repos/ibm/PROD',
    $repo_user          = 'repuser',
    $repo_passwd        = 'login123',
    $repo_baseurl       = 'http://lde0002p.de.top.com/pub/repos/ibm/PROD',
    $stage              = 'app-pre',
){
# defaults for exec resources
Exec {
      path        => '/usr/bin:/bin:/usr/sbin:/sbin',
      logoutput   => false,
      timeout     => '3600',
      }

if str2bool($gis_im_installed) != true {
  notify {"IM STATUS IS:  ${gis_im_installed}":} ->

  archive { '/tmp/IM_inst/IM173_x86.zip':
    ensure        => present,
    source        => 'http://lde0002p.de.top.com/pub/repos/ibm/PROD/IM173_x86.zip',
    username      => 'repuser',
    password      => 'login123',
    extract_path  => '/tmp/IM_inst',
    checksum      => '01c079c750476b670454603c4dcd7e52337349d6',
    checksum_type => 'sha1',
    user          => '0',
    group         => '0',
    extract       => true,
    cleanup       => true,
  } ->

  notify {'Installing Installation Manager':} ->
  exec { 'install_ibm_im':
    command => '/tmp/IM_inst/tools/imcl install \'com.ibm.cic.agent\' -repositories /tmp/IM_inst  -installationDirectory /opt/IBM/InstallationManager/eclipse -accessRights admin -acceptLicense 2>&1 > /tmp/instInstallManager.log ',
    unless  => 'ls /opt/IBM/InstallationManager/eclipse/tools/imcl',
    } ->

  notify {'Saving Installation Manager credentials':} ->
  exec { 'save_ibm_im_credentials1':
    command  => "${im_tooldir}/imutilsc saveCredential -url ${repo_baseurl}/IM -userName ${repo_user} -userPassword ${repo_passwd}",
    unless => 'ls /var/ibm/InstallationManager/secure_storage',
    } ->

  notify {'Updating Installation Manager':} ->
  exec { 'update_ibm_im':
    command   => "${im_tooldir}/imcl install 'com.ibm.cic.agent' -repositories  ${repo_baseurl}/IM -preferences 'offering.service.repositories.areUsed=false' -sVP",
      }
#  exec { 'remove_instdir':
#    command   => 'rm -rf /tmp/IM_inst'
#    }
  }
} 
~~~




