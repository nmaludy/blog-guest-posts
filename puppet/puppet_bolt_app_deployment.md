# Deploying An Application with Bolt

You've been iterating on some code and finally have that new feature working. You submit a 
Pull Request, there are a few comments that you address quickly then the PR gets merged.
What happens next? It's time for deployment. Read on if you're intersted in learning how 
Bolt can be used to deploy your application.

## Background

This post goes into details on how we at Encore Technologies deploy our application code
using Bolt. Before we get started showing off the Bolt code, we should probably go over
what our application is.

We use a tool called [StackStorm](https://stackstorm.com/) for orchestrating our automation 
workflows. StackStorm is an event-driven automation platform that we use to integrate the 
best-of-breed tools across the DevOps landscape. These integrations are possible using 
StackStorm's plugin mechanism called "packs" (short of Packages). Packs are Git repos
that contain automation code written in a variety of different languages: `bash/sh`,
`PowerShell`, `Python`, anything executable, and Orquesta workflows (YAML).

Along side these packs we need to deploy automation code for the external tools that StackStorm
integrates with:
 - Puppet/Bolt (`/etc/puppetlabs/code`)
 - Terraform (`/opt/terraform`)
 - Packer (`/opt/packer`)
 - Many others, check out the [StackStorm Exchange](https://exchange.stackstorm.org/)

StackStorm's packs then either execute scripts, make API calls or invoke an external tool
(Bolt, Puppet, Terraform, Packer, etc) to automate various tasks.

## Deployment Process

From a high-level, pack deployment consists of the following steps:
 - Clone Git repos
 - Create a deployment archive
 - Copy the archive to the StackStorm host
 - Extract the archive
 - Put automation code in its proper spot
 - Register packs with StackStorm

There are two nodes in this process, the Bolt control node where the deployment is triggered
from and the StackStorm node(s) that the code will be deployed to. Our Bolt control node is 
either our Jenkins server or a developers laptop. The Bolt plan is initiated from the
control node where it does some prep work, copies the files over to the StackStorm nodes,
and then performs the deployment steps on those nodes.

### Cloning Git repos

In order for us to deploy code, we need to have the code first. Logically, the first step
in the process is cloning all of the Git repos we need for our deployment.  These Git repos
contain code for interacting with other tools such as Puppet, Bolt, Terraform, Packer, etc.
Cloning the repos is accomplished using the following plan:

```puppet
plan encore_st2::clone_repo(
  TargetSpec $nodes,
  String $path,
  String $url,
  String $repo,
  String $revision = undef,
) {
  $path_abs = encore_st2::join_path(encore_st2::resolve_path($path), $repo)
  $res = apply($nodes) {
    vcsrepo { $path_abs:
      ensure     => present,
      provider   => git,
      source     => $url,
      revision   => $revision,
      submodules => true,
    }
  }
  return $res
}
```

In this plan we utilize the [vcsrepo](https://forge.puppet.com/puppetlabs/vcsrepo) forge module
in an apply block. This saves us the hassle of having to artisinally craft the correct `git`
command. You may ask why we love Bolt? It's because we can reuse all of the awesome modules
from the [Forge](https://forge.puppet.com)!

This code clones our Git repos on our Bolt control node, in the `extern/` directory, and now
we have all of the code needed to perform a deployment.

### Creating an Archive

Next we need to archive all of the various files we need for the deployment:

```puppet
plan encore_st2::pkg_build(
  Hash $config,
  String $release,
  TargetSpec $nodes,
) {
  $pwd = encore_st2::pwd()

  # list of files under git control
  $ls_files_result = run_command("pushd $pwd &>/dev/null; git ls-files", $nodes)
  $ls_files = split($ls_files_result.first.value['stdout'], '\n')

  # list of extern/ directories (bolt, puppet, terraform, etc)
  $extern_files_result = run_command("pushd $pwd &>/dev/null; find extern/ -mindepth 1 -maxdepth 1 -type d",
                                     $nodes)
  $extern_files = split($extern_files_result.first.value['stdout'], '\n')

  # combine files
  $files = concat($ls_files, $extern_files)
  $files_str = join($files, ' ')

  # create archive
  $archive = "/tmp/${release}.tar.gz"
  $archive_result = run_task('encore_st2::tar', $nodes,
                             pwd     => $pwd,
                             archive => $archive,
                             files   => $files_str)
  return $archive
}
```

This plan uses `git ls-files` to list all of the files in our repo (all of the StacKStorm Packs)
and combines it with all of the external tool code that we cloned into the `extern/` directory.
The files list is then used to create a `.tar.gz` archive in the `/tmp` directory.

`encore_st2::tar` task:

```json
{
  "description": "Create a tar archive",
  "parameters": {
    "pwd": {
      "type": "String",
      "description": "pwd when creating the archive"
    },
    "archive": {
      "type": "String",
      "description": "Name of the archive file"
    },
    "files": {
      "type": "String",
      "description": "List of files to put in the archive"
    }
  }
}
```

```sh
#!/bin/sh
if [ ! -z "$PT_pwd" ]; then
  pushd "$PT_pwd" &> /dev/null
fi

if [ -e "$PT_archive" ]; then
  rm -f "$PT_archive"
fi

tar -zcf "$PT_archive" $PT_files
```

### Copy the Archive

The archvie now needs to make its way to the StackStorm nodes, to do this we utilize the built-in `upload_file()` [function](https://puppet.com/docs/bolt/1.x/plan_functions.html#upload-file).

```puppet
upload_file($pkg_path, $pkg_dst, $nodes)
```

### Extract the archive

Extracting the archive is as simple as running a `tar` command:

```
run_command("tar -xvf ${pkg_dst} -C ${release_dst}", $nodes)
```

### Place automation code in its proper spot

Next we need to put the code for our external different tools in the correct directories
so those tools can utilize them affectively. We do this using symlinks. Alternatively we
could copy the code into the desired directory.

For Puppet/Bolt code we create a symlink from `/opt/encore/stackstorm/current/extern/puppet` to
`/etc/puppetlabs/code/environments/production`.

For Packer code we create a symlink from `/opt/encore/stackstorm/current/extern/packer` to
`/opt/packer`.

```puppet
  # create puppet symlink
  if $cfg['puppet'] {
    # /opt/encore/stackstorm/current/extern/puppet
    $puppet_path = encore_st2::join_path($stackstorm_path,
                                         'current',
                                         $cfg['puppet']['deploy_path'],
                                         $cfg['puppet']['name'])
    # /etc/puppetlabs/code/environments/production
    $puppet_dir = $cfg['stackstorm_puppet_dir']
    apply($nodes) {
      file { $puppet_dir:
        ensure => link,
        target => $puppet_path,
      }
    }
  }

  # create packer symlink
  if $cfg['packer_repo'] {
    # /opt/encore/stackstorm/current/extern/puppet
    $packer_path = encore_st2::join_path($stackstorm_path,
                                         'current',
                                         $cfg['packer_repo']['deploy_path'],
                                         $cfg['packer_repo']['name'])
    # /opt/packer
    $packer_dir = $cfg['stackstorm_packer_dir']
    apply($nodes) {
      file { $packer_dir:
        ensure => link,
        target => $packer_path,
      }
    }
  }
```

### Register packs with StackStorm

In the final step of our deployment we need to move the StackStorm pack code into
the directory where StackStorm expects (`/opt/stackstorm/packs`), then register the packs
(StackStorm loads the data from the filesystem into its database) and finally setup a Python
`virtualenv` for each pack:

```puppet
plan encore_st2::pack_install_nested(
  Hash $config,
  Array[Hash] $packs,
  TargetSpec $nodes,
) {
  $stackstorm_path = $config['stackstorm_path']
  $stackstorm_packs_dir = '/opt/stackstorm/packs'

  # create paths for the packs
  $packs_data = $packs.map |$p| {
    {
      'path_src' => encore_st2::join_path($stackstorm_path, 'current', $p['src']),
      'path_dst' => encore_st2::join_path($stackstorm_packs_dir, $p['name']),
      'name'     => $p['name'],
    }
  }

  # copy packs into proper directory
  # set proper permissions on pack directories
  $res = apply($nodes) {
    $packs_data.each |$p| {
      file { $p['path_dst'] :
        ensure  => directory,
        recurse => true,
        source  => "file:///${p['path_src']}",
        group   => 'st2packs',
      }
    }
  }

  # register packs
  $packs_path_dst = $packs_data.map |$p| {$p['path_dst']}
  $packs_path_dst_str = join($packs_path_dst, ' ')
  run_command("st2 pack register ${packs_path_dst_str}", $nodes)

  # refresh virtualenvs
  $packs_name = $packs_data.map |$p| {$p['name']}
  $packs_name_str = to_json($packs_name)
  run_command("st2 run packs.setup_virtualenv update=true packs='${packs_name_str}'", $nodes)
}
```

For copying the code we again utilize an `apply` block with a `file` resource to copy
the pack's code from `/opt/encore/stackstorm/packs/*` to `/opt/stackstorm/packs`. Next we
run the `st2 pack register` command to load all of the Pack information from the filesystem
into the StackStormd database. Finally we run the `st2 run packs.setup_virtualenv` action
to create a Python `virtualenv` for each pack.

## Conclusion

Bolt is an amazing tool with a ton of use cases. In this post we've demonstrated how the
DevOps team at Encore Technologies utilizes Bolt to package and deploy our automation code
to StackStorm hosts. If you're inspired by what you've seen here you can get started with 
Bolt by checking out the [documentation](https://puppet.com/docs/bolt/1.x/bolt.html). 
Sometimes documentation doesn't answer all of your questions, or maybe you just need to 
talk to a real person, in this case come visit the
[Puppet Slack](https://slack.puppet.com/). We'll be hanging out in the `#bolt` channel.


