# DHCPACK-Logger-and-Infoblox-Lease-Updater

#### Table of Contents

1. [Overview](#overview)
2. [Credits](#credits)
3. [Workflow - List of scripts and their function](#workflow)
    * [dhcplog.py - main script to process dhcp log file and update database](#dhcplog.py)
    * [dhcpsearch.py - database search script](#dhcpsearch.py)
    * [rest.py - script to push expiration updates into Infoblox](#rest.py)
    * [dhcppurge.py - script to clean up the database](#dhcppurge.py)
4. [System Setup](#set-up)



## Overview

Infoblox comes with a captive portal for doing authenticated DCHP, but lacks
one very important feature.   It does NOT have the ability to remove devices
from the database after they have been inactive for a set amount of time.
This script attempts to solve that problem by processing the Infoblox DHCP log
file, extracting each unique DHCP OFFER, and updating the corresponding MAC
filter entry with an updated expiration date based on a user defined
configuration.

This project contains a number of Python scripts that are used to perform the
following functions:

* Process Infoblox DHCP log files
* Log each unique MAC address handed out into a MySQL database
* Update Infoblox captive portal expiration date for each MAC address that
  received a DHCP OFFER/ACK using the RestFUL API.
* Offer a CLI interface to quickly search DHCP history

This readme will provide a brief overview of the general workflow of the
software, what each script does, and how to set up the automated system. For
more detailed information on each method within each script, check the source
code for the given script, as each file contains a large amount of step-by-step
explanatory comments.


## Credits and Copyright

Original concept and proof of concept code by Jason E. Murray of Washington
University in St. Louis.

Vast majority (and all the real work) by Andrew Hannenbrink while working under
the STARS (Student Technology and Resource Support -
http://sts.wustl.edu/programs/stars/) program.

All code is the copyright of Washington University in St. Louis and has been
released under the GNU Public License.

## Workflow

Three scripts and a MySQL database comprise this project: dhcplog.py,
dhcpsearch.py, rest.py, and a MySQL database.

###  dhcplog.py and the 'dhcpdb' Database 

Typically saved at /usr/local/bin/dhcplog.py. This script runs once a day,
parsing yesterday's Infoblox DHCP log file, loooking for lines describing a
DHCPACK event. Then, it updates this information to the database. When a
DHCPACK event is found, it extracts the date, client's IP address, mac address,
and hostname.  Yesterday's Infoblox log file by default can be found at the
path /var/log/infoblox.log.1  (This file is rotated every night by a process
called logrotate.   It is important to make sure that whatever system you use,
rotates the log file before this script is called on it).  The file can be very
large, and for Washington University's network, is usually longer than five
million lines on any given day. Below are five lines of a past log file,
infoblox.log.1:


	#Aug  6 09:07:31 128.252.0.17 dhcpd[310]: DHCPINFORM from 128.252.92.140 via 128.252.157.254
	#Aug  6 09:07:31 128.252.0.17 dhcpd[310]: DHCPACK to 128.252.92.140 (00:1f:29:01:d5:72) via eth1
	#Aug  6 09:07:31 128.252.0.17 dhcpd[310]: Hostname CC-036060 replaced by CC-036060
	#Aug  6 09:07:31 128.252.0.17 dhcpd[310]: DHCPINFORM from 128.252.159.33 via 128.252.159.254
	#Aug  6 09:07:31 128.252.0.17 dhcpd[310]: DHCPACK to 128.252.159.33 (00:21:70:14:5d:79) via eth1

This script conveniently groups all of the information to be loaded into the
database with this regular expression:

	'^(\w+\s+\d+\s+\d{2}:\d{2}:\d{2}) (\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) .*(DHCPACK) on (\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) to (([0-9a-f]{2}[:-]){5}[0-9a-f]{2}) ?(?:\((.*)\))? via'

Upon starting, the script also opens the database 'dhcpdb', looking for three
tables: 'clients', 'history', and 'tempMacs'. If the script cannot find these
tables, it will create them. These three tables are formatted as follows:

	clients (
		id INTEGER PRIMARY KEY
		macaddress CHAR(18) UNIQUE
	);

	history (
		id INTEGER PRIMARY KEY
		clientid INTEGER (This is foreign key for id in the clients table)
		ip CHAR(30)		
		date CHAR(30)		
		clienthostname CHAR(40)
	);

	tempMacs (
		id INTEGER PRIMARY KEY
		macaddress CHAR(18) UNIQUE
	);

The clients table keeps track of each unique MAC address which was offered an
IP address.

The history table keeps track of each unique DHCPACK event in recent history.
Each row in the history table corresponds to IP address offered to a client.

The tempMacs table keeps track of all clients who have had a DHCPACK event in
the last week. tempMacs is used once a week to update the DHCP leases of recent
users. This table is created in order to reduce the load on the Infoblox server
when the weekly update to DHCP filter is performed.   This allows us to make a
single update per MAC address per week to the backend Infoblox database.  The
table is purged once a week by the rest.py script.



### dhcpsearch.py

This script can be used for searching the MySQL database 'dhcpdb' from the
command line, and should be saved at the path /usr/local/bin/dhcpsearch.py. For
usage, run '$ python /usr/local/bin/dhcpsearch.py -h'. This command returns the
following usage message:

Search with three arguments.

* The -p option specifies ip address. The ip address should be surrounded by
  quote ticks.

* The -t option specifies the date and time, in the format: <'YYYY-MM-DD
  HH:MM:SS.00'> (military time surrounded by quote ticks).

* The -i option specifies the time increment (in minutes) of how long of an
  increment you want to search over. The time increment should not be
  surrounded by quote ticks.

### rest.py

rest.py uses Infoblox's Python NIOS REST API to update users' lease expiration
date within the Infoblox system by requesting and modifying JSON data objects
for each user. rest.py runs once a week and should be saved at
/usr/local/bin/rest.py. This script takes every entry from the tempMacs table
in the 'dhcpdb' database, and attempts to update their DHCP lease expiration
date to 180 days (can be customized in the configuration file) from the current
date.

IMPORTANT: In order for rest.py to run at decent pace, it needs to make a lot
of concurrent HTTP 'GET' requests. So make sure that the Infoblox Grid Manager
client has its http connection limit set to a high enough value with the
CLI command: 

    set connection_limit https <limit number>

Where <limit number> is between 0 and 2147483647, where 0 denotes no limit.
This is also mentioned in the SET UP section of this read me.

When this script finishes running, it clears the tempMacs table, preparing it
to re-populate with next week's clients who participate in DHCPACKS.

### dhcppurge.py
	
dhcppurge.py runs once a month, deleting rows from the history table that are a
certain number of months old. So if it's May, and dbExpirationMonths is set to
2, all entries from March will be deleted, regardless of when the script runs
in May.

Next, dhcppurge.py removes any rows from the clients table that no longer have
any corresponding entries in the history table.


## SET UP

To set up this system, start by saving the files to the folling paths:
	
	/usr/local/bin/dhcplog.py
	/usr/local/bin/dhcpsearch.py
	/usr/local/bin/dhcppurge.py
	/usr/local/bin/rest.py
	/usr/local/etc/dhcpcfg.py

Next, set up a Cron job. I created a new user with the ability to execute all
six files, and set up their crontab file to look like this:
	
	# m h  dom mon dow   command
	00 03  *   *   *  python /usr/local/bin/dhcplog.py
	00 06  *   *   6  python /usr/local/bin/rest.py
	00 17  19  *   *  python /usr/local/bin/dhcppurge.py

This runs dhcplog.py once a day at three in the morning, rest.py once a week on
Saturday at six in the morning, and dhcppurge.py once a month on the nineteenth
at five in the afternoon. 

Next, edit the file /usr/local/etc/dhcpcfg.py to configure the program
settings. This is what each variable means in dhcpcfg.py:

* logFilePath -	Path to infoblox DHCP log file, usually 'infoblox.log.1'
* sqlHost - MySQL host, usually 'localhost'
* sqlUser - Username of user to sign into MySQL with. Make sure that this user has permission to edit the database specified by the 'db' parameter below.
* sqlPasswd - The MySQL user's corresponding MySQL password
* db - Name of the MySQL database to be used. The software system can create the database structure and schema on its own, but you do need to make a database by this name manually. Though, there is no table creation necessary.
* senderEmail -	The email address used to send diagnostic reports of dhcplog.py, dhcppurge.py, and rest.py
* senderPassword - senderEmail's password
* recipientEmails - A python list of email addresses to send the diagnostic reports to
* maxLease - The number of days used to set a user's DHCP expiration date into the future from their last DHCPACK event.
* ibxUrl - Infoblox Grid Master URL. i.e. 'https://gm.ip.wustl.edu/wapi/v1.2/'
* infobloxUser - The username for the Infoblox account used to connect to the Grid Master
* infobloxPasswd - infobloxUser's password
* dbExpirationMonths - Approximately the number of months to keep a history record before deleting it from the database. Check the section 'dhcppurge.py' in this README to see exactly how this works.
