# NIDS with Snort + Snorby

Learning project; setting up a Network Intrusion Detection System on a local VM using Snort to sniff and analyze traffic, with Snorby as the web frontend to visualize alerts. Done as part of getting hands-on with network security monitoring concepts.

## What it does

Snort sits on the network interface in passive mode, inspecting every packet against a ruleset. When traffic matches a rule; a port scan, a known exploit signature, suspicious HTTP patterns; it raises an alert and logs it. Snorby reads those logs and gives you a dashboard where you can browse, filter, and acknowledge alerts without staring at raw log files.

```
Network traffic
      │
   [Snort]  ←  snort.conf (rules, preprocessors, network vars)
      │
  snort.log (unified2 format)
      │
   [Barnyard2]  ←  reads unified2, feeds into MySQL
      │
   [Snorby]  ←  Rails web app, reads MySQL, shows the dashboard
```

Barnyard2 is the bridge between Snort's binary log format and the database Snorby needs; Snort doesn't write to a database directly.

## Environment

- Ubuntu/Debian VM (tested on Ubuntu 20.04)
- Snort 2.9.x
- Snorby 2.6.x
- Barnyard2 2.x
- MySQL 5.x
- Network interface: `ens33` (VMware NAT); change this to yours
- Home network: `192.168.182.0/24`

## Files

`snort.conf`; the main Snort config. The parts that actually matter and that you'd change per environment are at the top: `HOME_NET`, the port variables, and the rule includes at the bottom. Everything in the middle (preprocessors) is mostly left at sane defaults.

`docs/screenshots/`; proof it worked.

## Setup

### 1. Install Snort

```bash
sudo apt update
sudo apt install snort -y
```

The installer will ask for your home network range; put in your subnet (e.g. `192.168.182.0/24`).

### 2. Drop in the config

```bash
sudo cp snort.conf /etc/snort/snort.conf
```

Then edit the top section to match your network:

```bash
sudo nano /etc/snort/snort.conf
```

The two lines you definitely need to update:

```
ipvar HOME_NET 192.168.182.0/24   # your actual subnet
```

And make sure the interface in your startup command matches what `ip a` shows you.

### 3. Validate the config

```bash
sudo snort -T -c /etc/snort/snort.conf -i ens33
```

`-T` is test mode; it loads the full config and rules, validates everything, and exits without sniffing any traffic. If you see `Snort successfully validated the configuration!` you're good. See `docs/screenshots/snort-config-validation.png`.

### 4. Run Snort

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i ens33
```

`-A console` prints alerts to the terminal so you can see them in real time. `-q` suppresses the banner. For production/background use, drop `-A console` and let it log to `/var/log/snort/`.

### 5. Install and configure Snorby

Snorby needs Ruby, Rails, MySQL, and Barnyard2. This takes a while.

```bash
# install dependencies
sudo apt install -y mysql-server ruby ruby-dev rails libmysqlclient-dev git

# install barnyard2 (reads snort's unified2 logs into mysql)
sudo apt install -y barnyard2

# clone snorby
git clone https://github.com/Snorby/snorby.git /opt/snorby
cd /opt/snorby
bundle install
```

Set up the database:

```bash
cp config/snorby_config.yml.example config/snorby_config.yml
# edit snorby_config.yml with your mysql credentials
bundle exec rake snorby:setup
```

Start it:

```bash
bundle exec rails server -e production
```

Snorby runs on port 3000 by default; hit `http://localhost:3000` in a browser.

## The config file breakdown

The `snort.conf` is structured in 9 steps (Snort's own numbering). The ones worth knowing:

**Step 1; network variables.** `HOME_NET` is your protected network. `EXTERNAL_NET` is everything else (left as `any`). These get referenced throughout the ruleset; a rule like "alert if external tries to connect to SSH on home" uses these vars.

**Step 5; preprocessors.** These are plugins that run before rules are evaluated. `stream5` does TCP stream reassembly so Snort can detect attacks spread across multiple packets. `http_inspect` normalizes HTTP traffic so evasion tricks (double encoding, weird whitespace) don't bypass rules. `frag3` handles IP fragmentation reassembly.

**Step 6; output.** Set to `unified2` format which is Snort's binary log format; compact and fast. Barnyard2 reads this and writes to MySQL.

**Step 7; rules.** The `include` lines at the bottom load rule files from `/etc/snort/rules/`. Commented-out ones either weren't available in the Debian package or weren't needed for this setup.

## Screenshots

`docs/screenshots/snort-install.png`; fresh Snort install confirming version.

`docs/screenshots/snort-interface-config.png`; binding Snort to the network interface.

`docs/screenshots/snort-config-validation.png`; successful config validation run.

## What I learned

Snort's rule syntax is straightforward once you read a few; `alert tcp any any -> $HOME_NET 22` is basically English. The harder part is the preprocessor config and understanding why reassembly matters for detection. Snorby's setup is the most painful part of this stack; Ruby dependency hell is real. Barnyard2 as a separate process between Snort and the database is an odd design but it keeps Snort focused on packet processing without blocking on DB writes.
