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
