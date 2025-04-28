Capture syslog message from wifi access points into PfSense/OPNsense syslog (very hacky)

Wifi seems to log to `hostapd`, exploit that fact by letting an access point send its logs to our PfSense/OPNsense instance and log it on machine to make diagnostics easier

Determine the PfSense/OPNsense BSD version.  Download that BSD version then compile this "marvelous" piece of code.

OR alternatively download a compiled version for x86_64 if you want to give that a go:
`curl --output '/usr/local/syslog_proxy/syslog_proxy_pfsense_freebsd_13.4.binary'  'https://raw.githubusercontent.com/jvandervyver/OPNsense/refs/heads/main/wifi-syslog/syslog_proxy_pfsense_freebsd_13.4.binary'`

**syslog_proxy.c**
```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <ctype.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <syslog.h>

#define SERVER_PORT "31953"
#define SYSLOG_PROCESS "hostapd"
#define READ_BUFFER_SIZE (1024 * 24)
#define MAX_READ_BUFFER_SIZE (READ_BUFFER_SIZE - 1)

static char buffer[READ_BUFFER_SIZE];

static int handle_new_udp_message(const char* buf, size_t length);

static void receive_udp(const int udp_socket) {
  struct sockaddr_in client_addr;
  socklen_t client_addr_size = sizeof(struct sockaddr_in);

  memset(buffer, 0, READ_BUFFER_SIZE * sizeof(char));
  const ssize_t bytes_received = recvfrom(
    udp_socket,
    buffer,
    MAX_READ_BUFFER_SIZE,
    0,
    (struct sockaddr*) &client_addr,
    &client_addr_size);

  buffer[MAX_READ_BUFFER_SIZE] = 0;
  if (bytes_received > 0 && bytes_received < MAX_READ_BUFFER_SIZE) {
    buffer[bytes_received] = 0;
    if (handle_new_udp_message(buffer, (size_t) bytes_received) != 0) {
      syslog(LOG_WARNING, "Invalid syslog message dropped", 0);
    }
  }
}

static void bind_udp(const int udp_socket, struct sockaddr_in* server_addr) {
  memset(server_addr, 0, sizeof(struct sockaddr_in));

  server_addr->sin_family = AF_INET;
  server_addr->sin_port = htons((int) atoi(SERVER_PORT));
  server_addr->sin_addr.s_addr = htonl(INADDR_ANY);

  const int result = bind(udp_socket, (struct sockaddr*) server_addr, sizeof(struct sockaddr_in));
  if (result != 0) {
    printf("Could not bind UDP socket. %s\n", strerror(errno));
    exit(2);
  }
}

static int create_udp_socket() {
  const int udp_socket = socket(PF_INET, SOCK_DGRAM, 0);
  if (udp_socket < 0) {
    printf("Could not create a socket. %s\n", strerror(errno));
    exit(1);
  }

  return udp_socket;
}

static void open_syslog() {
  openlog(SYSLOG_PROCESS, LOG_PID, LOG_LOCAL3);
}

static int handle_new_udp_message(const char* buf, size_t length) {
  /**
   * Example messages:
   * <5>Mar  9 16:38:57 kernel: klogd started: BusyBox v1.25.1 (2019-02-02 13:16:50 EST)
   * <5>Mar  9 16:39:28 rc_service: cfg_server 2561:notify_rc restart_logger
   * <5>Mar  9 16:39:29 kernel: klogd: exiting
   */

  // Minimum length
  if (length < 19) return -1;

  // Verify what appears to be priority
  int priority = 0;
  if (buf[0] == '<') {
    char parse[] = { buf[1], buf[2], 0 };
    if (!isdigit(buf[1])) return -1;
    if (buf[2] == '>') {
      parse[1] = 0;
      buf += 3;
    } else {
      if (!isdigit(buf[2])) return -1;
      if (length < 20) return -1;
      if (buf[3] != '>') return -1;
      buf += 4;
    }

    priority = atoi(parse);
  } else return -1;

  // Verify month
  if (!isalpha(buf[0]) || !isupper(buf[0])) return -1;
  if (!isalpha(buf[1]) || !islower(buf[1])) return -1;
  if (!isalpha(buf[2]) || !islower(buf[2])) return -1;
  if (buf[3] != ' ') return -1;
  buf += 4;

  // Verify day
  if (!(isdigit(buf[0]) || buf[0] == ' ')) return -1;
  if (!isdigit(buf[1])) return -1;
  if (buf[2] != ' ') return -1;
  buf += 3;

  // Verify time
  // Hour
  if (!isdigit(buf[0])) return -1;
  if (!isdigit(buf[1])) return -1;
  if (buf[2] != ':') return -1;
  //Minute
  if (!isdigit(buf[3])) return -1;
  if (!isdigit(buf[4])) return -1;
  if (buf[5] != ':') return -1;
  //Second
  if (!isdigit(buf[6])) return -1;
  if (!isdigit(buf[7])) return -1;
  if (buf[8] != ' ') return -1;
  buf += 9;

  syslog(LOG_PRI(priority), "%s", buf);
  return 0;
}

int main() {
  open_syslog();

  const int udp_socket = create_udp_socket();

  struct sockaddr_in server_addr;
  bind_udp(udp_socket, &server_addr);

  // Receive loop
  while(1) {
    receive_udp(udp_socket);
  }

  close(udp_socket);
}
```

`gcc syslog_proxy.c`

scp/copy the file to `/home/scripts/` or `/usr/local/syslog_proxy`, etc.

`chmod 555 <filename>` the binary file

Create a startup "script":
```
touch /usr/local/etc/rc.syshook.d/start/30-syslog-proxy
<copy contents>
```

```
cat /usr/local/etc/rc.syshook.d/start/30-syslog-proxy
#!/bin/sh

/home/scripts/syslog_proxy_freebsd_13.4.binary &
```

Whatever the path or filename is needs to be corrected here: `/home/scripts/syslog_proxy_freebsd_13.4.binary`
