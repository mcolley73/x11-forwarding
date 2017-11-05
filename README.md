## X11 Forwarding Experiment
The desire is to be able to go through a bastion/jump server to a target and allow
X11Forwarding back through the bastion to a client.

```sh
  local --> bastion --> target
```

This would allow us to execute a GUI application on `target` but interact with its screen from
our `local`, so that we need only expose the secure SSH port through the relevant pieces of
the network.

All Linux instances are using RHEL-7. All testing was done from a MacMini running
v10.12.6 of macOS Sierra.

## XQuartz (Mac OSX)
You will need an XWindows client so you can see the exported display. I was working on
a MacMini, so I used XQuartz:
https://www.xquartz.org/

## Inventories
The VPC and Subnet creation is left up to the reader. Variables in the inventory
should be changed to work in your environment. Of note:
* `aws_profile` - The name of the profile in `~/.aws/credentials` to use
* `aws_ssh_key_pair` - The AWS Key Pair to use when spinning up instances
* `target_vpc` - The VPC into which we are building our instances
* `target_subnet` - The Subnet we are to use
* `target_subnet_cidr_block` - The CIDR block of the subnet
* `trusted_network_cidr` - Your house or corporate network, perhaps

## Execution
This project is divided into a few key playbooks and associated Ansible Roles.
* `build-bastion.yml`
  As you would expect, this stands up the bastion.
* `build-target.yml`
  This builds our final target instance.

Run each of the above in turn. Doing so will result in the appropriate Security Group
being created as well.

## X11 `sshd` settings
There are some key `sshd` settings for the bastion and the target to make things work with
forwarding.

Bastion (see `roles/bastion-config/files/sshd_config`):

```sh
GatewayPorts yes
X11Forwarding yes
X11DisplayOffset 10
AllowTcpForwarding yes
```

Target (see `roles/target-config/files/sshd_config`). Note the extra setting:

```sh
GatewayPorts yes
X11Forwarding yes
X11DisplayOffset 10
AllowTcpForwarding yes

# Extra setting for final target
X11UseLocalhost no
```

## Testing
After Execution, we should be able to test using XClock. By convention, I will label the
various command line executions with `local`, `bastion`, and `target`.

*NOTE:* You can use Private IPs if you have a VPN. I did not, so I used Public and made
sure to lock port 22 down well using my Security Group.

Fire up XQuartz on your Mac (or some equivalent on your system) so you can be ready to receive
the XWindows forwards.

*NOTE:* I had to Log Out of my Mac Mini and back in, then launch XQuartz for it to work.

We should be able to SSH directly to the bastion and use XClock. We need the `-Y` argument for
our X11 forwarding to work.

```sh
local $ ssh -Y ec2-user@<bastion-public-dns>
Last login: Sat Nov  4 19:58:42 2017 from <local-ip>
/usr/bin/xauth:  file /home/ec2-user/.Xauthority does not exist
[ec2-user@<bastion-private-ip> ~]$ xclock
```

We should be able to do the exact same test by SSHing directly to the final target and
using XClock from there. Still need the `-Y` argument.

```sh
local $ ssh -Y ec2-user@<target-public-dns>
Last login: Sat Nov  4 19:58:42 2017 from <local-ip>
/usr/bin/xauth:  file /home/ec2-user/.Xauthority does not exist
[ec2-user@<target-private-ip> ~]$ xclock
```

But our goal is to go *through* the bastion to the target. So what we'd like to do is
SSH to the Private DNS of the target from our local. But if we try it, it won't work:

```sh
local $ ssh -Y ec2-user@<target-private-dns>
ssh: Could not resolve hostname <target-private-dns>: nodename nor servname provided, or not known
local $
```

*NOTE:* If you're on a VPN, you might be able to hit the Private DNS directly. But that's not what
this experiment is about =)

### SSH Trickery
We can use some SSH configuration to overcome this issue. Within our SSH Config in `~/.ssh/config` we can set up our bastion as the bridge between our local and our targets.

My Subnet was `172.29.32.0/20`, as you could have seen in `./inventories/sandbox` for the value
of the `target_subnet_cidr_block` variable. That should help you understand the below config,
in which I tried to handle prepping for either Private DNS or Private IPs:

```sh
Host bastion
  User ec2-user
  HostName <bastion-public-dns>
  IdentityFile ~/.ssh/<aws-ssh-key-pair-file>.pem

Host ip-172-29*
  ProxyCommand ssh -Y ec2-user@bastion -W %h:%p 2>/dev/null

Host 172.29.*
  ProxyCommand ssh -Y ec2-user@bastion -W %h:%p 2>/dev/null
```

Now, whenever we try to SSH to something in the `172.29.*.*` range, which is the range
of our *private* IPs in the Subnet, SSH will proxy our traffic through our bastion using
the *public* DNS of the bastion itself. We run the same command again but we get better
results:

```sh
local $ ssh -Y ec2-user@<target-private-dns>
The authenticity of host '<target-private-dns> (<no hostip for proxy command>)' can\'t be established.
ECDSA key fingerprint is SHA256:8dkd8dkd8dkd8dkd8dkd8dkd8dkd8dkd8dkd8dkd+8dkd.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '<target-private-dns>' (ECDSA) to the list of known hosts.
Last login: Sat Nov  4 22:15:59 2017 from <local-ip>
[ec2-user@<target-private-ip> ~]$ xclock
```

It works!

---

## Chrome
XClock is a perfect tool for testing things out, but what about a Browser? The code in our
`target-config` Role installed both Google Chrome and Firefox.

Fire up Chrome:

```sh
local $ ssh -Y ec2-user@<target-private-dns>
[ec2-user@<target-private-ip> ~]$ google-chrome https://www.google.com
```

It takes a while, but it'll come up. And we're now running a Browser in our Linux `target`
and exporting its display all the way back to our `local` screen.

---

## Firefox
Firefox wouldn't run without crashing. At first, it would error out whenever I launched a tab.

Fire up Firefox:

```sh
local $ ssh -Y ec2-user@<target-private-dns>
[ec2-user@<target-private-ip> ~]$ firefox https://www.google.com
<snip>
boatload of errors
</snip>
```

Eventually, I got it to work by going into the Firefox settings with HamburgerMenu->Preferences.

THEN I was able to change the URL of that tab to `about:config`, and scroll down
to change 2 settings to false (double click on them in the GUI):

```sh
browser.tabs.remote.autostart = false
browser.tabs.remote.autostart.2 = false
```

I got this idea from this URL:
https://github.com/SeleniumHQ/docker-selenium/issues/388

After changing these settings, you can open a new tab, and even close and re-start, and
firefox will run clean:

```sh
local $ ssh -Y ec2-user@<target-private-dns>
[ec2-user@<target-private-ip> ~]$ firefox https://www.google.com
<browse for a while, then exit or ctrl-c>
[ec2-user@<target-private-ip> ~]$
```

---

HTH
