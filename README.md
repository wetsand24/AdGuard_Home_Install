# AdGuard Home 1.0 on Ubuntu Server: install, block apps with one switch, back up, update

Every command from my AdGuard Home 1.0 video, in video order. Tested in November 2026 with AdGuard Home v1.0.0 on Ubuntu Server 24.04.

▶️ Video: https://youtu.be/ylZ4mvNyj1w

AdGuard Home is a DNS blocker for the whole home network. It blocks ads and trackers for every device that uses it as its DNS server, and its **Blocked services** page blocks a whole app (TikTok, Netflix, ChatGPT and 139 more) with one switch, for the whole house or for one device.

It does **not** remove YouTube ads. YouTube serves its ads from the same names as its videos, and a DNS blocker can only refuse a whole name. AdGuard's own FAQ says the same.

## What you need

- A small server or virtual machine on your home network. The video uses Ubuntu Server 24.04 with 2 processor cores and 4 GB of memory.
- A fixed IP address for that server (a DHCP reservation in your router, or a static address on the server). Every device will ask this one machine, so its address must not change.
- A user on the server that can run `sudo`.
- A PC on the same network to work from. Windows 10 and 11 have `ssh` built in.

In the commands below, replace `10.37.0.55` with your server's IP address and `demo` with your user name.

## Step 1. Install AdGuard Home

Connect to the server from Terminal on your PC:

```bash
ssh demo@10.37.0.55
```

The first time, ssh asks whether you trust the server: type `yes`. Then type your password. Nothing shows on screen while you type it.

Install with AdGuard Home's own one-line installer:

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

`curl` fetches the installer, the pipe `|` passes it on, and `sh` runs it. It asks for your password once. It installs into `/opt/AdGuardHome`, starts the service, and ends with:

```
AdGuard Home is now installed and running
```

### Free port 53 (needed on Ubuntu)

DNS uses port 53. On Ubuntu a small DNS helper called systemd-resolved already holds that port, so the setup wizard shows **"This port is already in use"** on its DNS page and cannot finish. On Ubuntu Server 24.04 the wizard did not offer a Fix button in my tests, so run these lines before you open the wizard. They follow AdGuard's own guide:

```bash
sudo ss -lntup 'sport = :53'          # shows systemd-resolve on 127.0.0.53:53 and 127.0.0.54:53
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNS=127.0.0.1\nDNSStubListener=no\n' | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl reload-or-restart systemd-resolved
sudo ss -lntup 'sport = :53'          # now only the header line: port 53 is free
```

What they do: the first line shows who holds port 53. The middle lines write a small settings file that tells Ubuntu's helper to let go of the port, and make the server send its own lookups to AdGuard Home. The last two restart the helper and check again.

## Step 2. The setup wizard

In a browser, open `http://10.37.0.55:3000` and click **Run wizard**.

1. Username and password (typed twice). Tick the privacy policy box, or **Next** stays greyed out.
2. Web UI address: leave port 80.
3. DNS server address: port 53, with no error now.
4. The device guide: **Next**, then **Open dashboard**, and log in.

From now on the control page is at `http://10.37.0.55` (no `:3000`).

## Step 3. Make one PC use AdGuard Home

Start with one PC, so a mistake only affects that PC. On Windows:

Control Panel > Network and Internet > Network and Sharing Center > Change adapter settings > right-click Ethernet (or Wi-Fi) > Properties > Internet Protocol Version 4 (TCP/IPv4) > **Use the following DNS server addresses** > type the server's IP > OK.

Test it in Terminal on that PC:

```powershell
nslookup doubleclick.net
```

You should see your server's address under `Server:` (the name `UnKnown` is normal) and all-zero addresses for the name:

```
Addresses:  ::
          0.0.0.0
```

In AdGuard Home, open the menu > **Query log** and search for `doubleclick`. The line has the reason **Blocked by filter**.

**The Server line shows a long address with colons?** Your router also hands out an IPv6 DNS server, and Windows asks that one first, so the PC skips AdGuard Home. In the same Properties window, untick **Internet Protocol Version 6 (TCP/IPv6)**, click OK, and run the test again. When you switch the whole house over (Step 7), look for an IPv6 DNS setting in your router as well.

## Step 4. Block an app with one switch

**Filters > Blocked services.** The page lists 142 services (AdGuard Home v1.0.0). Turn on the switch next to an app. There is no Save button; it applies at once.

The PC and the browser both remember the old address for a while. To see the block right away:

```powershell
ipconfig /flushdns
```

Then close the browser and open it again. For a blocked site, Edge may first say **"This site doesn't support a secure connection"**; after **Continue to site** it shows **"Hmmm... can't reach this page"**. That is what a blocked app looks like.

One switch covers every name the app uses. The ChatGPT switch, for example, blocks `chatgpt.com`, `openai.com`, `oaistatic.com` and `oaiusercontent.com`, and every address that ends in one of those.

The Query log shows these lookups with the reason **Blocked services** and the app's name.

## Step 5. Block an app on one device only

**Settings > Clients > Add client.**

1. **Client name**: any name, for example `living-room-pc`.
2. **Identifiers**: that device's IP address. Reserve the address in your router so it stays the same.
3. Switch **on** "Use global settings for Protection". In v1.0.0 it is **off** on a new client, and a client saved that way gets **no ad blocking at all**.
4. Open **Blocked services** inside the client. "Use global blocked services" is off there, which gives this device a list of its own. Turn on the apps for this device.
5. Go back to the Add client form and click **Save**.

Good to know: a device with a list of its own **ignores** the switches on the main Blocked services page completely. Whatever that device should lose has to be on its own list.

**A device can skip AdGuard Home** by asking another DNS server:

```powershell
nslookup tiktok.com 1.1.1.1
```

That returns the real address even while TikTok is blocked. Stopping it takes a firewall rule on the router that only lets your DNS server send DNS out. I explain that rule in my Technitium video: https://youtu.be/MQcR3S3qYVM

## Step 6. When a link you want stops working

Sooner or later the blocklist stops a name you want. Query log > search for the device or the name > the three dots on the blocked line > **Unblock**. AdGuard Home adds an allow rule like this:

```
@@||ad.doubleclick.net^$important
```

It is listed under **Filters > Custom rules**, where you can delete it again. Then run `ipconfig /flushdns` on the PC and try again.

If the thing that stopped working is an app you blocked on the Blocked services page, turn its switch off.

## Step 7. Switch the whole house over

In your router's DHCP or DNS settings, enter the IP address of the AdGuard Home server as the DNS server. Devices pick it up when they reconnect. If a Pi-hole's address is in that field today, the AdGuard Home address replaces it.

Before you save that setting: from then on the whole house depends on this server. While it is switched off, no device can look up a site. Some routers do not accept a custom DNS server; then set each device by hand as in Step 3. AdGuard Home also has a built-in DHCP server for that case, which the video does not cover.

## Step 8. Back up, restore, update

AdGuard Home has no backup button. Everything is in `/opt/AdGuardHome`: the settings file `AdGuardHome.yaml` and the folder `data`.

**Back up** (nothing gets DNS answers while AdGuard Home is stopped, so pick a quiet moment):

```bash
sudo /opt/AdGuardHome/AdGuardHome -s stop
sudo tar -czf ~/adguard-backup.tar.gz -C /opt/AdGuardHome AdGuardHome.yaml data
sudo /opt/AdGuardHome/AdGuardHome -s start
ls -lh ~
```

Copy `adguard-backup.tar.gz` off the server to keep it safe.

**Restore:**

```bash
sudo /opt/AdGuardHome/AdGuardHome -s stop
sudo tar -xzf ~/adguard-backup.tar.gz -C /opt/AdGuardHome
sudo /opt/AdGuardHome/AdGuardHome -s start
```

**Update:** when a new version is out, a line appears at the top of the control page with an update button. I tested it from v0.107.80 to v1.0.0: a few seconds, and blocked services, clients and custom rules were kept. Make a backup first anyway.

A server that was upgraded keeps its old upstream DNS server (Quad9 on an old install). A fresh 1.0 install uses Cloudflare. You can change it under Settings > DNS settings.

**Service control:**

```bash
sudo /opt/AdGuardHome/AdGuardHome -s status
sudo /opt/AdGuardHome/AdGuardHome -s restart
```

## Troubleshooting

- **"This port is already in use" in the wizard.** Run the port 53 lines from Step 1, then reload the wizard page.
- **`sudo: unable to resolve host ...` after the port 53 lines.** The server's name is missing from `/etc/hosts`. It is harmless; this adds it:
  ```bash
  echo "127.0.1.1 $(hostname)" | sudo tee -a /etc/hosts
  ```
- **ssh does not connect.** Check the server's address at its own screen with `ip a`. If ssh was not installed with Ubuntu, install it there: `sudo apt update && sudo apt install -y openssh-server`.
- **An app still opens after I turned its switch on.** Run `ipconfig /flushdns` and restart the browser (Step 4). Check with `nslookup tiktok.com` that the PC really asks AdGuard Home (Step 3). A device that has a client entry with its own list ignores the main switches (Step 5).
- **A browser or phone keeps skipping AdGuard Home.** "Secure DNS" in a browser and iCloud Private Relay send lookups elsewhere. AdGuard Home already tells Firefox to turn its automatic secure DNS off; for iCloud Private Relay there is a switch on the Blocked services page.
- **YouTube ads are still there.** Expected: DNS blocking cannot remove them.

## Uninstall

From AdGuard's documentation; I did not run this in my lab. Point your devices and router at another DNS server first.

```bash
cd /opt/AdGuardHome
sudo ./AdGuardHome -s uninstall
cd ~ && sudo rm -r /opt/AdGuardHome
```

To give port 53 back to Ubuntu's helper afterwards:

```bash
sudo rm /etc/systemd/resolved.conf.d/adguardhome.conf
sudo rm /etc/resolv.conf && sudo mv /etc/resolv.conf.backup /etc/resolv.conf
sudo systemctl reload-or-restart systemd-resolved
```

## Links

- AdGuard Home (open source, GPL-3.0): https://github.com/AdguardTeam/AdGuardHome
- AdGuard Home FAQ, known limitations: https://adguard-dns.io/kb/adguard-home/faq/
