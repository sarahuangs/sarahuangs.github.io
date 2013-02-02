---
layout: post
title: "Puppet concat patterns"
---

It's common to want to configure a shared resource from many classes in your
module tree, each class defining it's own configuration parameters that then
merge into a more unified whole.

When the resource you're configuring natively supports a conf.d style
this makes the configuration management job quite simple, you just drop
out a new file from each class that's interested and call it done.

The complexity comes when you have a single file that you need to
manage.  At that point it's time to break out one of the concatenation
options available to you with puppet.

Probably the most widely-used concatenation pattern is to use
R.I.Pienaar's excellent puppet-concat module.

If you're not familiar with it here's a worked example of adding lines
to the /etc/motd file from multiple classes.

    # motd/manifests/init.pp
    class motd {
        concat { '/etc/motd': }

        concat::fragment { 'motd_header':
            target  => '/etc/motd',
            content => "Hello ${::fqdn}\n",
            order   => '01',
        }
    }

    # motd/manifests/line.pp
    define motd::line {
        concat::fragment { "motd_${title}":
            target  => '/etc/motd',
            content => "${title}\n",
            order   => '10',
        }
    }

    # chrome/manifests/init.pp
    class chrome {
        motd::line { "this machine has chrome on it": }
    }

`concat` is great when your files are line-oriented, but when you need multiple
classes to add things to the same line you need to use something a little more
complex.

The pattern I find myself using for this kind of configuration you could
describe as a 'fragment stitching' pattern. You drop data fragments into
a temporary location, then you run a script that parses those those
fragments and assembles them into your intended configuration file.

Here's a worked example of this for a nagios class that manages the
definition of hostgroups.

    # nagios/manifests/init.pp
    class nagios {
        file { '/etc/nagios/build':
            ensure  => directory,
            purge   => true,
            recurse => true,
            notify  => Exec['nagios_assemble_hostgroups'],
        }

        file { '/etc/nagios/build/assemble_hostgroups':
            source => 'puppet:///modules/nagios/assemble_hostgroups',
            mode => '0755',
        }

        exec { 'nagios_assemble_hostgroups',
            command => '/etc/nagios/build/assemble_hostgroups'
        }
    }

    # nagios/manifests/host.pp
    define nagios::host($hostgroup = $title) {
        file { "/etc/nagios/build/__HOST_${::hostname}_${hostgroup}.yml",
            content => "{ 'host': '${::hostname}', 'hostgroup': '${hostgroup}' }\n",
            notify  => Exec['nagios_assemble_hostgroups'],
        }
    }

    # nagios/files/assemble_hostgroups
    #/usr/bin/env ruby
    $fragment_dir = '/etc/nagios/build'
    $output_file  = '/etc/nagios/objects/hostgroups.cfg'

    hostgroups = {}
    Dir($fragment_dir).each do |filename|
        next unless filename =~ /\.yml$/
        fragment = YAML.load_file("#{fragment_dir}/#{filename}")
        hostgroups[fragment['hostgroup']] ||= []
        hostgroups[fragment['hostgroup']] << fragment['hostname']
    end

    assembled = ""
    hostgroups.keys.sort.each do |hostgroup|
         assembled = assembled + "
    define hostgroup {
         name    #{hostgroup}
         members #{hostgroups[hostgroup].sort.join(',')}
    }"
    end

    File.open($output_file, "w") { |f| f.write(assembled) }

    # ntp/manifests/init.pp
    class ntp {
        nagios::host { "ntp server: }
    }

This works but the major drawback is that you need to write a new assembly
script each time you use it, and there's all those intermediate
files in a build directory to manage.  In short it's complex and hard to
follow.

Having implemented this stitching pattern more than enough times I started to
think about the practicality of moving to a more data-oriented model, where the
reassembly step can be a straightforward template evaluation step.

Revisiting the nagios hostgroups pattern, with datacat it would look like this:

    # nagios/manifests/init.pp
    class nagios {
        datacat { '/etc/nagios/objects/hostgroups.cfg':
            template => 'nagios/hostgroups.cfg.erb',
        }
    }

    # nagios/manifests/host.pp
    define nagios::host($hostgroup = $title) {
        datacat_fragment { "nagios::host[${hostgroup}]":
            data => {
                $hostgroup => [ $::hostname ],
            },
        }
    }

    # nagios/templates/hostgroups.cfg.erb
    <% @data.keys.sort.each do |hostgroup| %>
    define hostgroup {
         name    <%= hostgroup %>
         members <%= @data[hostgroup].sort.join(',') %>
    }
    <% end %>

    # ntp/manifests/init.pp
    class ntp {
        nagios::host { "ntp server: }
    }

Hopefully the datacat types provide a general enough model to allow you
to manage arbritary files you build up with data, and the interface
should be clearer than dropping multiple fragment files.

I have some rough edges to sand away, which may include giving it a
better name, and much more testing to do, but datacat should be
available from the forge in a few days.