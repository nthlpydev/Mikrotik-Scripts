```
######## Customize These Variables #########
:local PublicAddresses {1.1.1.1;2.2.2.2;3.3.3.3}
:local startport 1025
:local numberOfPorts 2000
####################################

:local success false
:local currentport
:local foundport
:local foundaddress

:if ($leaseBound="1") do={
  #check for available block
  :if ($leaseActIP in 100.64.0.0/10) do={
    :foreach addr in=$PublicAddresses do={
      :if (!$success) do={
        :set currentport $startport
        :while (($currentport <= (65535 - $numberOfPorts)) && !$success) do={
          :if ([:len [/ip firewall nat find where to-addresses=$addr to-ports="$currentport-$($currentport + $numberOfPorts - 1)"]] = 0) do={
            :set success true
            :set foundport $currentport
            :set foundaddress $addr
          } else={
            :set currentport ($currentport + $numberOfPorts)
          }
        }
      }
    }
    # Specific client chain jumps
    /ip firewall nat add chain=clients action=jump jump-target="cl-$leaseActMAC" src-address=$leaseActIP
  
    # Translation rule
    /ip firewall nat add chain="$leaseActMAC" action=src-nat protocol=tcp src-address=$leaseActIP to-addresses=$foundaddress to-ports="$foundport-$($foundport + $numberOfPorts - 1)"
    /ip firewall nat add chain="$leaseActMAC" action=src-nat protocol=udp src-address=$leaseActIP to-addresses=$foundaddress to-ports="$foundport-$($foundport + $numberOfPorts - 1)"
    /ip firewall nat add chain="$leaseActMAC" action=src-nat protocol=icmp src-address=$leaseActIP to-addresses=$foundaddress
  }
}
:if ($leaseBound=0) do={
  :if ($leaseActIP in 100.64.0.0/10) do={
    /ip firewall nat remove [find where src-address=$leaseActIP]
  }
}
```
