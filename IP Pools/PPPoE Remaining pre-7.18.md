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
	:local minaddress
	:local maxaddress
	:local findmask
	:local mask
	:local findindex
	:local tmpint
	:local maxindex
	:local line
    :set totaladdresses 0
	:set totalused 0

	:foreach pool in=[find where name~$poolprefix] do={
		:set poolname [get $pool name]
		:set pooladdresses 0
        :set poolused [:len [used find where pool=[:tostr $poolname]]]

		:foreach range in=[:toarray [get $pool range]] do={
			:set findmask [:find [:tostr $range] "/"]
			:if ([:len $findmask] > 0) do={
				:set mask [:pick [:tostr $range] ($findmask + 1) [:len [:tostr $range]]]
				:set tmpint (2<<(31-$mask))
				:set pooladdresses ($pooladdresses + $tmpint)
			} else={
				:set findindex [:find [:tostr $range] "-"]
				:if ([:len $findindex] > 0) do={
					:set minaddress [:pick [:tostr $range] 0 $findindex]
					:set maxaddress [:pick [:tostr $range] ($findindex + 1) [:len [:tostr $range]]]
				} else={
					:set minaddress [:tostr $range]
					:set maxaddress [:tostr $range]
				}
				:for x from=0 to=([:len [:tostr $minaddress]] - 1) do={
					:if ([:pick [:tostr $minaddress] $x ($x + 1)] = ".") do={
						:set minaddress ([:pick [:tostr $minaddress] 0 $x] . "," . [:pick [:tostr $minaddress] ($x + 1) [:len [:tostr $minaddress]]])
					}
				}
				:for x from=0 to=([:len [:tostr $maxaddress]] - 1) do={
					:if ([:pick [:tostr $maxaddress] $x ($x + 1)] = ".") do={
						:set maxaddress ([:pick [:tostr $maxaddress] 0 $x] . "," . [:pick [:tostr $maxaddress] ($x + 1) [:len [:tostr $maxaddress]]])
					}
				}
				:if ([:len [:toarray $minaddress]] = [:len [:toarray $maxaddress]]) do={
					:set maxindex ([:len [:toarray $minaddress]] - 1)
					:for x from=$maxindex to=0 step=-1 do={
						:set tmpint 1
						:if (($maxindex - $x) > 0) do={
							:for y from=1 to=($maxindex - $x) do={ :set tmpint (256 * $tmpint) }
						}
						:set tmpint ($tmpint * ([:tonum [:pick [:toarray $maxaddress] $x]] - [:tonum [:pick [:toarray $minaddress] $x]]) )
						:set pooladdresses ($pooladdresses + $tmpint)
					}
				}
				:set pooladdresses ($pooladdresses + 1)
			}
		}
		:set $totalused ($totalused + $poolused)
		:set $totaladdresses ($totaladdresses + $pooladdresses)
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
