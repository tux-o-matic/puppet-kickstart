# Kickstart

[![Puppet Forge](http://img.shields.io/puppetforge/v/danzilio/kickstart.svg?style=flat)](https://forge.puppetlabs.com/danzilio/kickstart) [![Build Status](https://travis-ci.org/danzilio/puppet-kickstart.svg?branch=master)](https://travis-ci.org/danzilio/puppet-kickstart) [![Documentation Status](http://img.shields.io/badge/docs-puppet--strings-ff69b4.svg?style=flat)](http://danzilio.github.io/puppet-kickstart)

This module provides a primitive to generate a Kickstart file from Puppet data. This can be useful for Kickstarting physical or virtual machines. This module basically allows you to store your Kickstarts as structured data.

## Support

This module only parses a template and generates a file, so it's conceivably universally compatible. Given that Kickstarts are generally only used to configure RedHat-like Linux systems, I've listed CentOS and RedHat in the module metadata.

## Usage

You'll use the `kickstart` type to create Kickstart configurations. The title of the `kickstart` resource is a path where you want Puppet to place the file. The `commands` and `parititon_configuration` parameters accept a hash where the key is a string that represents the kickstart command and the value is a string or array of strings that represent arguments to that command. If you want a command to appear in the kickstart more than once, you can pass an array of arguments for each invocation. If you want a command to appear without arguments, you must set its value to `true`.

```
kickstart { '/var/www/html/kickstart.cfg':
  commands       => {
    'install'    => true,
    'text'       => true,
    'reboot'     => true,
    'skipx'      => true,
    'url'        => '--url http://mirror.centos.org/centos/6/os/x86_64',
    'lang'       => 'en_US.UTF-8',
    'keyboard'   => 'us',
    'network'    => '--device eth0 --bootproto dhcp',
    'rootpw'     => 'test1234',
    'firewall'   => '--disabled',
    'selinux'    => '--permissive',
    'authconfig' => '--enableshadow --enablemd5',
    'timezone'   => 'UTC',
    'bootloader' => '--location mbr',
  },
  parititon_configuration => {
    zerombr   => 'yes',
    clearpart => '--all --initlabel',
    part      => [
      '/boot --fstype ext3 --size 250',
      'pv.2 --size 5000 --grow',
    ],
    volgroup  => 'VolGroup00 --pesize 32768 pv.2',
    logvol    => [
      '/ --fstype ext4 --name LogVol00 --vgname VolGroup00 --size 1024 --grow',
      'swap --fstype swap --name LogVol01 --vgname VolGroup00 --size 256 --grow --maxsize 512'
    ]
  }
}
```

This would result in the following Kickstart:

```
### THIS FILE WAS GENERATED BY PUPPET ###

# Command Section
install
text
reboot
skipx
url --url http://mirror.centos.org/centos/6/os/x86_64
lang en_US.UTF-8
keyboard us
network --device eth0 --bootproto dhcp
rootpw test1234
firewall --disabled
selinux --permissive
authconfig --enableshadow --enablemd5
timezone UTC
bootloader --location mbr

# Partition Configuration
zerombr yes
clearpart --all --initlabel
part /boot --fstype ext3 --size 250
part pv.2 --size 5000 --grow
volgroup VolGroup00 --pesize 32768 pv.2
logvol / --fstype ext4 --name LogVol00 --vgname VolGroup00 --size 1024 --grow
logvol swap --fstype swap --name LogVol01 --vgname VolGroup00 --size 256 --grow --maxsize 512
```

This can get messy, so I recommend that you put this data into Hiera. You can pass arbitrary templates to the `kickstart` type for use in the `pre` or `post` installation sections using the `fragments` parameter. You must pass the `fragments` parameter a hash. The keys for the `fragments` hash must be the name of the section you want the fragment placed under (this is generally `pre` or `post`). The value for each section must be an array of paths to ERB templates.

```
kickstart { '/var/www/html/kickstart.cfg':
  fragments => {
    'post --interpreter /bin/bash' => [
      '/some/fully/qualified/path/configure_network.erb',
      'kickstart_profile/install_puppet.erb'
    ]
  }
}
```

You can pass your template fragments variables using the `fragment_variables` parameter. You can access those variables in your fragment templates using the `@fragment_variables` instance variable:

```
kickstart { '/var/www/html/kickstart.cfg':
  fragments => {
    'post --interpreter /bin/bash' => [
      '/some/fully/qualified/path/configure_network.erb',
      'kickstart_profile/install_puppet.erb'
    ]
  },
  fragment_variables => {
    'foo' => 'bar'
  }
}
```

You can pass a list of packages to install using the `packages` parameter:

```
kickstart { '/var/www/html/kickstart.cfg':
  packages => [
    '@base',
    'redhat-lsb',
    'apache',
    'mariadb-server',
    'nagios-nrpe'
  ]
}
```

or the hash version:

```
kickstart { '/var/www/html/kickstart.cfg':
  packages => {
    'package_list' => [
      '@base',
      'redhat-lsb',
      'apache',
      'mariadb-server',
      'nagios-nrpe'
    ]
  }
}
```

This would result in the following snippet:

```
# Packages Section
%packages
@base
redhat-lsb
apache
mariadb-server
nagios-nrpe
%end
```

You can pass a list of packages, with options, to install using the `packages` parameter:

```
kickstart { '/var/www/html/kickstart.cfg':
  packages => {
    'options' => '--nobase --ignoremissing',
    'package_list' => [
      '@core',
      'nagios-nrpe'
    ]
  }
}
```

This would result in the following snippet:

```
# Packages Section
%packages --nobase --ignoremissing
@core
nagios-nrpe
%end
```

You can also add repositories for use during installation using the `repositories` parameter:

```
kickstart { '/var/www/html/kickstart.cfg':
  repositories => {
    'base' => { 'baseurl' => 'http://mirror.centos.org/centos/6/os/x86_64' }
  }
}
```

This would result in the following snippet:

```
repo --name base --baseurl http://mirror.centos.org/centos/6/os/x86_64
```


This can get messy, so I recommend that you put this data into Hiera. You can pass arbitrary templates to the `kickstart` type for use in the `pre` or `post` installation sections using the `addons` parameter. You must pass the `addons` parameter a hash. The keys for the `addons` hash must be the name of the addon, and must include any options required.  The value for each addon must be a srting or an array of data required by the addon.

You can configured addon with optional data, for use during installation, using the `addons` parameter.

e.g: CentOS 7 provides the Kdump add-on by default, which adds support for configuring kernel crash dumping during the installation.  It's possible to write custom addons for Anaconda.
Refer to [documentation](https://access.redhat.com/documentation/en/red-hat-enterprise-linux/version-7.1/red-hat-enterprise-linux-71-anaconda-customization-guide/) and the [reference implementation](https://github.com/rhinstaller/hello-world-anaconda-addon).

```
kickstart { '/var/www/html/kickstart.cfg':
  addons => {
    'com_redhat_kdump --enable --reserve-mb="auto"' => []
    'my_addon_name --arg1 --arg2="value2"' => ['example1','example2','example3'],
  }
}
```

This would result in the folloiwng snippet:
```
# Addons Section
%addon com_redhat_kdump --enable --reserve-mb="auto"
%end

%addon my_addon_name --arg1 --arg2='value2'
example1
example2
example3
%end
```

## Validation

By default, this module will validate your Kickstart commands against a list of valid commands; if you pass this type an invalid Kickstart command, it will fail to compile. The `kickstart` type checks to make sure that you're passing valid Kickstart commands based on [this](https://github.com/rhinstaller/pykickstart/blob/master/docs/kickstart-docs.rst) list. If you'd like to pass a command that isn't on that list, you can set the `fail_on_unsupported_commands` parameter to `false`:

```
kickstart { '/var/www/html/kickstart.cfg':
  fail_on_unsupported_commands => false,
  commands => {
    unsupported_commands => "I'm an unsupported command!"
  }
}
```

## Development

1. Fork it
2. Create a feature branch
3. Write a failing test
4. Write the code to make that test pass
5. Refactor the code
6. Submit a pull request

We politely request (demand) tests for all new features. Pull requests that contain new features without a test will not be considered. If you need help, just ask!
