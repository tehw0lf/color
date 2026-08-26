# Color

The only use case of this project is to set the body's background color to the value provided in the search value:

| Format | Example |
| --- | --- |
| HEX | https://example.com/?8e8e8e |
| RGB | https://example.com/?rgb(142,142,142) |
| CMYK | https://example.com/?cmyk(0,0,0,44.3) |
| Named color | https://example.com/?grey |

The first three all resolve to `#8e8e8e`. Any CSS color value works — named
colors resolve to their CSS definition, so `grey` is `#808080`, not `#8e8e8e`.
CMYK is not a CSS format and is parsed separately; its components accept
decimals, which is what `44.3` above needs to land exactly on `#8e8e8e`.

The resolved value is shown in the overlay as HEX, RGB and CMYK.
