# 🐭 Postfix Relay Compose

A small locked-down SMTP relay for home LAN, built on [boky/postfix](https://github.com/bokysan/docker-postfix) and packaged as a Docker Compose file. Insecure and old devices on your network (printers, NAS, cameras, NVR) isolated from the internet send e-mail to the relay, and the relay forwards it to your real SMTP provider over verified TLS.

Tested on **[QNAP Container Station](https://www.qnap.com/en-uk/how-to/tutorial/article/container-station-quick-start-guide)**, but it's plain Docker Compose and should work on any host with Docker (see [Known limitations](#known-limitations)).

#### What it does

- Accepts e-mail on a single port (`2525`) from a short list of allowed clients by their local-IPs.
- Relays it to your provider over SMTP (implicit TLS) with certificate verification.

## Requirements

- A host running Docker with Compose (QNAP: Container Station).
- An SMTP provider that supports SMTPS on port 465 (with 587/STARTTLS, then remove `POSTFIX_smtp_tls_wrappermode`).
- Fixed IPs (or DHCP reservations) for the clients that will send e-mail messages.

## Quick start

1. Copy the sample variables file and fill it in:

   ```bash
   cp vars.sample.env vars.env
   chmod 600 vars.env
   ```

2. Edit `vars.env` with your provider details and the client/recipient restrictions (see [Configuration](#configuration)).

3. Edit `docker-compose.yml`:
   - Set the `ports` binding to your host's LAN IP
   - Adjust the `env_file` path to point to your `vars.env`

4. Start it:

   ```bash
   docker compose up -d
   ```

### QNAP Container Station

1. Put `vars.env` somewhere persistent on the NAS (e.g. `/share/Container-Configs/postfix-relay/vars.env`) and set that path in `env_file`
2. In Container Station, create a new application and import the file or paste the contents of `docker-compose.yml`
3. Review the settings above
4. [Optional] Set resource limits (_Advanced settings_ -> _Resources_)
   - Set _CPU limit_ to _Limited_ with 1 core
   - Set _Memory limit_ to _Limited_ with 64 MB
   - Set _Memory reservation_ to _Limited_ with 10 MB
5. Create/start the application

## Configuration

Variables in `vars.env`:

| Variable                               | Description                                                                                                            |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `RELAYHOST`                            | Provider SMTP host and port, in brackets to skip MX lookup, e.g. `[smtps.example.com]:465`                             |
| `RELAYHOST_USERNAME`                   | Account used to authenticate to your provider                                                                          |
| `RELAYHOST_PASSWORD`                   | Password of that account (use a dedicated account or app password if possible)                                         |
| `POSTFIX_mynetworks`                   | Clients allowed to relay, by IP, e.g. `"127.0.0.0/8, 192.168.1.2/32"`                                                  |
| `POSTFIX_smtpd_recipient_restrictions` | Allowed recipient(s), e.g. `check_recipient_access inline:{you@example.com=OK}, reject` -> rejects any other recipient |

The settings in `docker-compose.yml` you will most likely want to change are the `ports` binding and `POSTFIX_message_size_limit`.

## Testing

Send a test message from an **allowed** client using 'nc' (netcat):

```bash
nc 192.168.1.18 2525

EHLO test
MAIL FROM:relayhost-username@example.com
RCPT TO:smtpd-recipient-restriction-contact@example.com
DATA
From: Test From-Relay <relayhost-username@example.com>
To: smtpd-recipient-restriction-contact@example.com
Message-ID: <f501abd8-b569-4f06-b38a-175f298bfe1d@example.com>
Subject: Test Postfix-Relay

This is a test.
.

QUIT
```

### Verify the relay is not open

This is the test that matters. From a device whose IP is **not** in `POSTFIX_mynetworks`, try the same command. It must be rejected:

```
554 5.7.1 <unknown[192.168.1.xxx]>: Client host rejected: Access denied
```

Also check that the logs show the **real client IP** in `connect from ...` lines. If you see the Docker gateway (`172.x.x.x`) instead, IP filtering is not working as intended on your host (see below).

## Known limitations

- **No client authentication.** Access control is by source IP only. Anyone who can reach the port from an allowed IP can send mail.
- **Source IP depends on your Docker networking.** With bridge networking, some hosts present all clients as the Docker gateway IP, which makes `mynetworks` ineffective. This setup was verified on the QNAP described above; **always run the test** in the previous section on your own host.
- **Client-to-relay traffic is not encrypted on the LAN** in this setup.
- **`cap_add` list found by trial and error** on Container Station. Other hosts or Docker versions may need adjustments (e.g. `NET_BIND_SERVICE`). Check the logs if the container fails to start.
- **Mail queue is not persistent** unless you add a volume for `/var/spool/postfix`. Messages in the queue are lost if the container is recreated.
- **No DKIM/SPF handling.** Deliverability depends on your provider and domain setup.

## Credits

All the heavy lifting is done by [boky/docker-postfix](https://github.com/bokysan/docker-postfix). This repository only provides a Compose configuration and brief documentation. Refer to that project for the [full list of options](https://github.com/bokysan/docker-postfix#configuration-options).

## Notes

> ⚠️ **Do not expose this relay to the Internet.** Clients are authorized by local-IP address only, there is no per-user authentication.

> ✌🏼 _THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE._
