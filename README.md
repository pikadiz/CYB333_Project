# CYB333_Project

# Failed Login Monitoring

Monitor failed login attempts on Windows servers and send alerts based on Active Directory information.

## Project Structure

- `monitor_failed_logins.py`: Main script for monitoring failed login attempts.
- `failed_login_service.py`: Script to run the monitoring script as a Windows service.
- `requirements.txt`: List of Python dependencies required for the project.

## Features

- Monitor failed login attempts from Windows Event Logs.
- Interact with Active Directory to fetch user details.
- Send email alerts to administrators upon detecting multiple failed login attempts.

### Prerequisites

Ensure you have Python and `pip` installed on your system.

## Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/pikadiz/failed-login-monitoring.git
   cd failed-login-monitoring

##  Usage
Running the Monitoring Script
     To run the monitoring script manually:
```
python monitor_failed_logins.py
```` 
Running as a Windows Service
      To run the monitoring script as a Windows service, follow these steps:

1.Install the service:
```
python failed_login_service.py install
```
2.Start the service:
```
python failed_login_service.py start
```
## Example Code Snippets 
  monitor_failed_logins.py

  ```
import win32evtlog
import win32evtlogutil
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime, timedelta
import time
from ldap3 import Server, Connection, ALL, NTLM

# Administrator email details
ADMIN_EMAIL = "admin@example.com"
SMTP_SERVER = "smtp.example.com"
SMTP_PORT = 587
EMAIL_USER = "your_email@example.com"
EMAIL_PASS = "your_password"

# AD details
AD_SERVER = "ad.example.com"
AD_USER = "your_ad_user"
AD_PASS = "your_ad_password"
BASE_DN = "DC=example,DC=com"

# Log monitoring parameters
LOG_TYPE = 'Security'
EVENT_ID = 4625
CHECK_INTERVAL = 60  # in seconds
FAILURE_THRESHOLD = 3
TIME_FRAME = 5  # in minutes

failed_attempts = {}

def send_alert(user, ip_address):
    msg = MIMEMultipart()
    msg['From'] = EMAIL_USER
    msg['To'] = ADMIN_EMAIL
    msg['Subject'] = "Alert: Multiple Failed Login Attempts"

    body = f"User {user} had {FAILURE_THRESHOLD} failed login attempts from IP {ip_address}."
    msg.attach(MIMEText(body, 'plain'))

    with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
        server.starttls()
        server.login(EMAIL_USER, EMAIL_PASS)
        text = msg.as_string()
        server.sendmail(EMAIL_USER, ADMIN_EMAIL, text)

def fetch_ad_users():
    server = Server(AD_SERVER, get_info=ALL)
    conn = Connection(server, user=AD_USER, password=AD_PASS, authentication=NTLM)
    conn.bind()

    conn.search(BASE_DN, '(objectclass=user)', attributes=['sAMAccountName', 'mail'])
    users = {entry.sAMAccountName.value: entry.mail.value for entry in conn.entries}

    conn.unbind()
    return users

def monitor_failed_logins():
    ad_users = fetch_ad_users()
    server = 'localhost'
    log_type = LOG_TYPE
    handle = win32evtlog.OpenEventLog(server, log_type)
    flags = win32evtlog.EVENTLOG_FORWARDS_READ | win32evtlog.EVENTLOG_SEQUENTIAL_READ
    time_frame = timedelta(minutes=TIME_FRAME)

    while True:
        events = win32evtlog.ReadEventLog(handle, flags, 0)
        for event in events:
            if event.EventID == EVENT_ID:
                event_time = event.TimeGenerated
                user = event.StringInserts[5]  # Extract user account name
                ip_address = event.StringInserts[18]  # Extract IP address

                if user not in failed_attempts:
                    failed_attempts[user] = []

                failed_attempts[user].append((event_time, ip_address))
                failed_attempts[user] = [attempt for attempt in failed_attempts[user] if datetime.now() - attempt[0] <= time_frame]

                if len(failed_attempts[user]) >= FAILURE_THRESHOLD:
                    if user in ad_users:
                        send_alert(ad_users[user], ip_address)
                    else:
                        send_alert(user, ip_address)
                    failed_attempts[user] = []

        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    monitor_failed_logins()
`````
failed_login_service.py
```
import win32serviceutil
import win32service
import win32event
import servicemanager
import logging
import time
from monitor_failed_logins import monitor_failed_logins  # Ensure this script is in the same directory

class FailedLoginService(win32serviceutil.ServiceFramework):
    _svc_name_ = "FailedLoginService"
    _svc_display_name_ = "Failed Login Monitoring Service"
    _svc_description_ = "Monitors failed login attempts and sends alerts to the administrator."

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)
        self.running = True

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        self.running = False
        win32event.SetEvent(self.hWaitStop)

    def SvcDoRun(self):
        servicemanager.LogMsg(servicemanager.EVENTLOG_INFORMATION_TYPE,
                              servicemanager.PYS_SERVICE_STARTED,
                              (self._svc_name_, ''))
        self.main()

    def main(self):
        logging.basicConfig(
            filename='C:\\FailedLoginService.log',
            level=logging.INFO,
            format='%(asctime)s %(levelname)-8s %(message)s',
            datefmt='%Y-%m-%d %H:%M:%S'
        )
        logging.info("Service started.")
        while self.running:
            try:
                monitor_failed_logins()
            except Exception as e:
                logging.error(f"Error: {e}")
            time.sleep(5)

if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(FailedLoginService)
