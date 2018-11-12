# Cuckoo and MISP Integration
Cuckoo comes with ready-to-use modules to interact with the MISP REST API via the PyMISP Python module.
There is one processing module (to search for existing IoC’s in MISP) and one reporting module (to create a new event in MISP).
The configuration is very simple, just define your MISP URL and API key in the proper configuration files and you’re good to go:

# Cuckoo configuration (change these two modules with the proper url and api key) Your MISP apikey can be found under the "Automation" tab
```
# cd /data/cuckoo/.cuckoo/conf/

processing.conf-enabled = yes
processing.conf-
processing.conf:[misp]
processing.conf-enabled = yes
processing.conf:url = https://misp.xxxxxxxxxx
processing.conf-apikey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
maxioc = 100
--
reporting.conf-logname = syslog.log
reporting.conf-
reporting.conf:[misp]
reporting.conf-enabled = yes
reporting.conf:url = https://misp.xxxxxxxxxx
reporting.conf-apikey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

# Cuckoo configuration is finished once the two configuration files have been set !

# MISP configuration (misp-modules must be enabled and running)
You will need to login to your MISP and go to the Administration -> Server Settings & Maintenance -> Plug-in Settings -> Import
ADD PICTURE HERE
