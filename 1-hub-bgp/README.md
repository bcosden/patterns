# Single Hub using BGP

There are no UDR's required for the primary data path (ExpressRoute, VPN, Hub, and Spokes). The only UDR is for the internet breakout on the NVA secondary NIC.

Note the ExpressRoute Gateway and Circuit is deployed as part of the script but the provisioning of the circuit is not part of the script.