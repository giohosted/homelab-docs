So your rack layout from top to bottom is shaping up to be something like:

- 1U — Patch panel (UniFi UP-PATCH-24)
- 1U — USW-Pro-Max-24
- 1-2U — MS-A2 (on Etsy rack mount)
- 1-2U — Optiplex (on Etsy rack mount)
- 4U — NAS chassis
- 1U — Pi (velcro/hook on upright, no U needed)
- 2U — UPS (SMART1500LCDXL with rack ears)

That's roughly 11-12U used out of 18U. Comfortable amount of breathing room for future expansion.

## Rack Layout Suggestion (Now That You Scored 18U)

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