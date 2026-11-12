# SmokePing Setup and Target Management

SmokePing is a deluxe latency measurement tool. It can measure, store and display latency, latency distribution and packet loss. SmokePing uses RRDtool to maintain a longterm data-store and to draw pretty graphs, giving up to the minute information on the state of each network connection.

## 1. Environment

  Component                Configuration
  ------------------------ ----------------------------------------
  Host                     `home-server01`
  Hardware                 Raspberry Pi 3 Model B Rev 1.2
  OS                       Debian 13 (Trixie), ARM64
  SmokePing                Native package installation
  Web URL                  `http://192.168.55.10:8080/smokeping/`
  Apache                   Port `8080`
  Probe                    `FPing`
  Measurement interval     5 seconds
  Probes per measurement   5

> Replace the example IP address and hostnames if your network
> configuration changes.

## 2. Install SmokePing and dependencies

Update package metadata and install the required packages:

``` bash
sudo apt update
sudo apt install -y smokeping apache2 rrdtool fping
```

Check the installed versions and service status:

``` bash
smokeping --version
rrdtool --version
fping --version
sudo systemctl status smokeping
sudo systemctl status apache2
```

## 3. Configure Apache on port 8080

Edit the Apache ports configuration:

``` bash
sudo nano /etc/apache2/ports.conf
```

Ensure it contains:

``` apache
Listen 8080
```

Edit the default virtual host:

``` bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Set the virtual host to:

``` apache
<VirtualHost *:8080>
    # Keep the existing DocumentRoot and other site directives.
</VirtualHost>
```

Restart Apache and verify that it is listening:

``` bash
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo ss -lntp | grep ':8080'
```

Open SmokePing in a browser:

``` text
http://192.168.55.10:8080/smokeping/
```

## 4. SmokePing configuration layout

The main configuration file is:

``` text
/etc/smokeping/config
```

It includes the configuration fragments:

``` text
@include /etc/smokeping/config.d/General
@include /etc/smokeping/config.d/Alerts
@include /etc/smokeping/config.d/Database
@include /etc/smokeping/config.d/Presentation
@include /etc/smokeping/config.d/Probes
@include /etc/smokeping/config.d/Slaves
@include /etc/smokeping/config.d/Targets
```

The main files used for routine administration are:

  ------------------------------------------------------------------------
  File                                 Purpose
  ------------------------------------ -----------------------------------
  `/etc/smokeping/config`              Includes the configuration
                                       fragments

  `/etc/smokeping/config.d/Database`   Measurement interval and historical
                                       data retention

  `/etc/smokeping/config.d/Probes`     Probe configuration

  `/etc/smokeping/config.d/Targets`    Hosts and network targets

  `/etc/smokeping/config.d/Alerts`     Alert definitions
  ------------------------------------------------------------------------

Back up a configuration file before editing it:

``` bash
sudo cp -a /etc/smokeping/config.d/Targets \
  /etc/smokeping/config.d/Targets.bak.$(date +%F-%H%M%S)
```

## 5. Configure five-second measurements and retention

Edit the database configuration:

``` bash
sudo nano /etc/smokeping/config.d/Database
```

Example configuration:

``` text
*** Database ***

step     = 5
pings    = 5

# Archive format: consolidation function, minimum data fraction,
# number of primary data points, total rows.

# 5-second resolution: 7 days
AVERAGE  0.5    1  120960

# 1-minute resolution: 1 year
AVERAGE  0.5   12  525600
MIN      0.5   12  525600
MAX      0.5   12  525600

# 5-minute resolution: 5 years
AVERAGE  0.5   60  525600
MIN      0.5   60  525600
MAX      0.5   60  525600
```

Retention calculations:

  Resolution     Data points per row      Rows   Approximate retention
  ------------ --------------------- --------- -----------------------
  5 seconds                        1   120,960                  7 days
  1 minute                        12   525,600                  1 year
  5 minutes                       60   525,600                 5 years

`pings = 5` sends five probe packets for each measurement cycle. The
`step = 5` setting schedules a measurement cycle every five seconds.

### Important: changing the RRD step

RRD files are created with a fixed step size. Existing files created
with `step = 300` cannot simply be reused with `step = 5`. SmokePing may
report an error similar to:

``` text
RRD parameter mismatch
Wrong value of step: ... has 300, create string has 5
```

If you are intentionally changing from 300 seconds to 5 seconds, back up
the old data and start with a new RRD directory. This preserves the old
history but does not automatically convert it:

``` bash
sudo systemctl stop smokeping

sudo mv /var/lib/smokeping \
  /var/lib/smokeping-300s-backup

sudo install -d -o smokeping -g smokeping /var/lib/smokeping
sudo install -d -o smokeping -g smokeping /var/lib/smokeping/__cgi

sudo systemctl start smokeping
```

Only do this once, when deliberately migrating the RRD step. Do not
repeat it when adding or removing targets.

## 6. Configure probes

Check the probe configuration:

``` bash
sudo nano /etc/smokeping/config.d/Probes
```

The targets configuration selects the probe with:

``` text
probe = FPing
```

Confirm that `fping` is installed and can reach a target:

``` bash
fping -c 5 8.8.8.8
```

DNS resolution can be checked separately:

``` bash
getent hosts vpn-gw-ben1.hertshtengroup.com
getent hosts vpn-gw-hyd1.hertshtengroup.com
getent hosts vpn-gw-gur1.hertshtengroup.com
```

A successful DNS lookup does not prove that the host responds to ICMP.
Use `fping` to test reachability.

## 7. Configure monitoring targets

Edit the targets file:

``` bash
sudo nano /etc/smokeping/config.d/Targets
```

Example configuration for public internet and VPN gateway monitoring:

``` text
*** Targets ***

probe = FPing

menu = Top
title = Network Latency Grapher
remark = Network latency and packet-loss monitoring.

+ Internet
menu = Internet
title = Internet Connectivity

++ Google
menu = Google DNS
title = Google DNS
host = 8.8.8.8

++ Cloudflare
menu = Cloudflare DNS
title = Cloudflare DNS
host = 1.1.1.1

++ Quad9
menu = Quad9 DNS
title = Quad9 DNS
host = 9.9.9.9

+ VPN
menu = VPN Gateways
title = Hertshten VPN Gateway Monitoring

++ BEN
menu = Bangalore VPN
title = Bangalore VPN Gateway
host = vpn-gw-ben1.hertshtengroup.com

++ HYD
menu = Hyderabad VPN
title = Hyderabad VPN Gateway
host = vpn-gw-hyd1.hertshtengroup.com

++ GUR
menu = Gurgaon VPN
title = Gurgaon VPN Gateway
host = vpn-gw-gur1.hertshtengroup.com
```

The hierarchy controls how targets are grouped in the SmokePing
interface:

-   `+` creates a top-level group.
-   `++` creates a target beneath that group.
-   `menu` is the navigation label.
-   `title` is the displayed target title.
-   `host` is the IP address or DNS hostname to probe.

Use unique target names within their parent group. The RRD path is based
on the target hierarchy and name.

## 8. Add a new target

1.  Back up the targets file.
2.  Edit `/etc/smokeping/config.d/Targets`.
3.  Add a new `++` target under the appropriate group.
4.  Give it a unique name, menu label, title, and host.
5.  Test DNS and ICMP reachability.
6.  Restart SmokePing.
7.  Check the service logs and confirm that a new RRD file is created.

Example: add a new target under the VPN group:

``` text
++ MUM
menu = Mumbai VPN
title = Mumbai VPN Gateway
host = vpn-gw-mum1.example.com
```

Commands:

``` bash
getent hosts vpn-gw-mum1.example.com
fping -c 5 vpn-gw-mum1.example.com

sudo systemctl restart smokeping
sudo systemctl status smokeping --no-pager
sudo journalctl -u smokeping -n 100 --no-pager
```

Check the RRD files:

``` bash
sudo find /var/lib/smokeping -type f -name '*.rrd' -print
```

A new RRD is created when SmokePing processes and probes the target
successfully enough to initialize its data file. Do not manually create
an empty RRD file.

## 9. Remove a target

1.  Back up `/etc/smokeping/config.d/Targets`.
2.  Remove the complete target block from the file.
3.  Restart SmokePing.
4.  Verify that the target no longer appears in the web interface.

Example: remove the Mumbai target by deleting its complete block:

``` text
++ MUM
menu = Mumbai VPN
title = Mumbai VPN Gateway
host = vpn-gw-mum1.example.com
```

Restart and verify:

``` bash
sudo systemctl restart smokeping
sudo systemctl status smokeping --no-pager
```

### What happens to historical data?

Removing a target from the configuration does not automatically delete
its RRD file. Its historical data remains on disk.

Find the RRD file before deciding what to do:

``` bash
sudo find /var/lib/smokeping -type f -name '*.rrd' -print
```

If you want to retain the history, leave the RRD file in place or
archive it. If you are certain that the history is no longer required,
stop SmokePing before deleting the specific file:

``` bash
sudo systemctl stop smokeping

# Replace this example path with the exact RRD file you identified.
sudo rm -i /var/lib/smokeping/VPN/MUM.rrd

sudo systemctl start smokeping
```

Do not delete the entire `/var/lib/smokeping` directory when removing an
individual target.

## 10. Configure packet-loss alerts

Edit:

``` bash
sudo nano /etc/smokeping/config.d/Alerts
```

Example alert definitions:

``` text
*** Alerts ***

to = YOUR_EMAIL@example.com
from = smokeping@home-server01

+packetloss
type = loss
pattern = >0%,>0%,>0%
comment = Packet loss detected across consecutive measurements

+internetdown
type = loss
pattern = >99%,>99%,>99%
comment = Target appears unreachable across consecutive measurements
```

Replace the email addresses with valid addresses. The alert definitions
alone do not guarantee email delivery. A working local mail transfer
agent or SMTP relay must be configured and tested on the Raspberry Pi.

Attach alerts to individual targets by adding the following line inside
each target block:

``` text
alerts = packetloss,internetdown
```

For example:

``` text
++ Google
menu = Google DNS
title = Google DNS
host = 8.8.8.8
alerts = packetloss,internetdown
```

Validate actual alert behaviour and mail delivery before relying on
these notifications. Alert pattern semantics depend on SmokePing's loss
history and should be tested against the installed version.

## 11. Service management

Common commands:

``` bash
# Restart SmokePing after configuration changes
sudo systemctl restart smokeping

# Check service status
sudo systemctl status smokeping --no-pager

# Follow SmokePing logs
sudo journalctl -u smokeping -f

# Show the last 100 log lines
sudo journalctl -u smokeping -n 100 --no-pager

# Stop and start
sudo systemctl stop smokeping
sudo systemctl start smokeping

# Enable at boot
sudo systemctl enable smokeping
```

## 12. Troubleshooting

### A. Target graph reports that an RRD file is missing

Example:

``` text
ERROR: opening '/var/lib/smokeping/VPN/BEN.rrd': No such file or directory
```

Check the following:

``` bash
# Confirm the target is in the active configuration
sudo grep -nE '^(\\+|\\+\\+|host|menu|title|probe)' \
  /etc/smokeping/config.d/Targets

# Check service status and recent logs
sudo systemctl status smokeping --no-pager
sudo journalctl -u smokeping -n 150 --no-pager

# List existing RRD files
sudo find /var/lib/smokeping -type f -name '*.rrd' -print
```

If necessary, run SmokePing in debug mode to see which targets are
loaded and probed. Stop the systemd service first so two SmokePing
processes do not run simultaneously:

``` bash
sudo systemctl stop smokeping

sudo install -d -o smokeping -g smokeping /run/smokeping

sudo smokeping --config=/etc/smokeping/config --debug
```

Review the output, then press `Ctrl+C` and restart the service:

``` bash
sudo systemctl start smokeping
```

Do not run debug mode concurrently with the normal daemon.

### B. `/run/smokeping` does not exist

`/run` is temporary and is recreated at boot. For a manual debug run,
create the runtime directory:

``` bash
sudo install -d -o smokeping -g smokeping /run/smokeping
```

### C. RRD step mismatch

If logs show that an existing RRD has a different step size, check
whether the database step was changed. RRD step size is fixed at file
creation. Use the migration procedure in Section 5 only if you
intentionally want to start a new RRD history at the new interval.

### D. DNS resolves but no graph data appears

Test ICMP reachability:

``` bash
getent hosts vpn-gw-ben1.hertshtengroup.com
fping -c 5 vpn-gw-ben1.hertshtengroup.com
```

Then inspect the SmokePing logs. DNS success alone does not guarantee
that the target allows ICMP traffic.

### E. Check directory ownership

``` bash
sudo ls -ld /var/lib/smokeping
sudo ls -ld /var/lib/smokeping/__cgi
sudo find /var/lib/smokeping -maxdepth 2 -type d -printf '%M %u:%g %p\\n'
```

The SmokePing service must be able to write its RRD files and runtime
data. Correct ownership only where required; do not recursively change
permissions without checking the package's expected layout.

## 13. Operational checklist

After adding or changing targets:

-   [ ] Back up the targets configuration.
-   [ ] Confirm the target hostname resolves.
-   [ ] Confirm ICMP reachability with `fping`.
-   [ ] Confirm the target block is correctly nested and uniquely named.
-   [ ] Restart SmokePing.
-   [ ] Check `systemctl status smokeping`.
-   [ ] Review recent SmokePing logs for errors.
-   [ ] Confirm the expected RRD file exists.
-   [ ] Confirm the graph loads in the web interface.
-   [ ] Test alert delivery if alerts are enabled.

## 14. Useful paths and commands

  Item                 Path or command
  -------------------- ----------------------------------------
  Main configuration   `/etc/smokeping/config`
  Targets              `/etc/smokeping/config.d/Targets`
  Database settings    `/etc/smokeping/config.d/Database`
  Alert settings       `/etc/smokeping/config.d/Alerts`
  RRD data             `/var/lib/smokeping/`
  Service logs         `sudo journalctl -u smokeping`
  Service status       `sudo systemctl status smokeping`
  Web interface        `http://192.168.55.10:8080/smokeping/`
