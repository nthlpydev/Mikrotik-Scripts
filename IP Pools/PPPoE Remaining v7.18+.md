This script will return the number of IPs remaining in an IP pool or group of IP pools used for PPPoE. This will work best if the pools that you are trying to check against all have a common portion of the name, which is set on the poolprefix variable. Other variables are the critical, error, and warning thresholds, set here to <50 IPs remaining, 50-74 IPs remaining, and 75-99 IPs remaining respectively.
```
:local criticalthreshold 50
:local errorthreshold 75
:local warnthreshold 100
:local poolprefix "PPPoE"
/ip pool {
  :local poolname
  :local pooladdresses
  :local poolused
  :local totaladdresses
  :local totalused
  :local poolremaining
  :local line
  :set totaladdresses 0
  :set totalused 0

  :foreach pool in=[find where name~$poolprefix] do={
    :set pooladdresses [get $pool total]
    :set poolused [get $pool used]
    :set totalused ($totalused + $poolused)
    :set totaladdresses ($totaladdresses + $pooladdresses)
  }
  :set poolremaining ($totaladdresses - $totalused)
  :set line ("PPPoE Utilization:")
  :set line ([:tostr $line] . "  [" . $totalused . "/" . $totaladdresses . "] - " . $poolremaining . " Free IPs Remaining")
  :if ( [:tonum $poolremaining] < $criticalthreshold ) do={
    :log error ("The PPPoE Pool has " . $poolremaining . " addresses free.")
    :log info message=($line)
  } else={
    :if ( [:tonum $poolremaining] < $errorthreshold ) do={
      :log warning ("The PPPoE Pool has " . $poolremaining . " addresses free.")
      :log info message=($line)
    } else={
      :if ( [:tonum $poolremaining] < $warnthreshold ) do={
        :log info ("The PPPoE Pool has " . $poolremaining . " addresses free.")
        :log info message=($line)
      } else={
        :log info message=($line)
      }
    }
  }
}
```
