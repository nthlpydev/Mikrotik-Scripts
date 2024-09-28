# General script
This needs to be run once before configuring anything else. There are two values to update here, `StartingAddress` and `ClientCount`. `StartingAddress` is the first IP in the CGNAT IP Pool you will be handing out to clients. This should be in the 100.64.0.0/10 block that is assigned for CGNAT. `ClientCount` is the number of addresses in the address pool. If you use multiple pools, you will need to run this once for each pool. The script example is setup for using the IP Pool of 100.64.0.0/24. As this is PPPoE, we can use the entire /24 for addressing, so we start at 100.64.0.0, and we have 256 IPs in the pool. This will work for pools greater than a /24 if you use the correct client count.
```
{
   ######## Customize These Variables #########
   :local StartingAddress 100.64.0.0
   :local ClientCount 256
   ####################################
   
   # All client chain jump
   /ip firewall nat add chain=srcnat action=jump jump-target=clients src-address="$StartingAddress-$($StartingAddress + $ClientCount - 1)" 
} 
```
# On-Up
This will go in the on-up section of the PPPoE Profile in the Scripts tab. There are three values to tweak here. The first is the IP Address array `PublicAddresses` which is the addresses that are set aside for mapping the clients to. Usually you will want these as setup on a loopback interface, which is just a bridge that has no ports assigned to it. If you have more than a few of these, you will likely want to look into a proper appliance for CGNATing. The second is the integer value in `startport` which is recommended to be 1025 and cannot go higher than 65535. The final is the integer value `numberOfPorts` which is the number of ports assigned to each client. You can get this value by taking (65535 - `startport`) and then divide by the number of clients you want per Public Address. The default values of 1025 for `startport` and 2000 for `numberOfPorts` will give you a ratio of 32:1 clients to public IPs.
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


#check for available block
:if ($"remote-address" in 100.64.0.0/10) do={
  :foreach PublicAddress in=$PublicAddresses do={
    :if (!$success) do={
      :set currentport $startport
      :while (($currentport <= (65535 - $numberOfPorts)) && !$success) do={
        :if ([:len [/ip firewall nat find where to-addresses=$PublicAddress to-ports="$currentport-$($currentport + $numberOfPorts - 1)"]] = 0) do={
          :set success true
          :set foundport $currentport
          :set foundaddress $PublicAddress
        } else={
          :set currentport ($currentport + $numberOfPorts)
        }
      }
    }
  }

  # Specific client chain jumps
  /ip firewall nat add chain=clients action=jump jump-target="$user" src-address=$"remote-address" comment=$user
  
  # Translation rules
  /ip firewall nat add chain="$user" action=src-nat protocol=tcp src-address=$"remote-address" to-addresses=$foundaddress to-ports="$foundport-$($foundport + $numberOfPorts - 1)" comment=$user
  /ip firewall nat add chain="$user" action=src-nat protocol=udp src-address=$"remote-address" to-addresses=$foundaddress to-ports="$foundport-$($foundport + $numberOfPorts - 1)" comment=$user
  /ip firewall nat add chain="$user" action=src-nat protocol=icmp src-address=$"remote-address" to-addresses=$foundaddress comment=$user
}
```
# On-Down
This will go in the on-down section of the PPPoE Profile in the Scripts tab. There is no customization needed on this script.
```
:if ($"remote-address" in 100.64.0.0/10) do={
  /ip firewall nat remove [find where src-address=$"remote-address"]
}
```
