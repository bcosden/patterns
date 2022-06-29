# Single Hub using Static Routes (UDR's)

This pattern utilizes UDR's to force routing in Azure virtual networks. While BGP can be used in this pattern, the UDR's are required to force ExpressRoute to NVA and Spoke to NVA.

Note the ExpressRoute Gateway and Circuit is deployed as part of the script but the provisioning of the circuit is not part of the script.