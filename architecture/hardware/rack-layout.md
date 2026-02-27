So your rack layout from top to bottom is shaping up to be something like:
## where is the udm-se going?????
- 1U — Patch panel (UniFi UP-PATCH-24)
- 1U — USW-Pro-Max-24
- 1-2U — MS-A2 (on Etsy rack mount)
- 1-2U — Optiplex (on Etsy rack mount)
- 4U — NAS chassis
- 1U — Pi (velcro/hook on upright, no U needed)
- 2U — UPS (SMART1500LCDXL with rack ears)

That's roughly 11-12U used out of 18U. Comfortable amount of breathing room for future expansion.

## Rack Layout Suggestion (Now That You Scored 18U)
## where is the udm-se going?????
Top to bottom:

1U Patch Panel  
1U Switch (Pro-Max-24 😮‍🔥)  
1U Horizontal Cable Manager  
1U Blank  
Servers  
UPS at bottom

Label both sides.

Wall jack label → Patch panel port number → Switch port number

Document it in:

homelab-docs → network/physical_topology.md

| U      | Device                |
| ------ | --------------------- |
| 1      | UDM-SE (1U)           |
| 2      | Patch panel (1U)      |
| 3      | USW-Pro-Max-24 (1U)   |
| 4-5    | MS-A2 shelf           |
| 6-7    | Optiplex shelf        |
| 8-11   | NAS — RSV-L4412U (4U) |
| 12     | Pi                    |
| 13-18  | Empty / future        |
| Bottom | UPS                   |
# Final layout
![[Pasted image 20260227170254.png]]
