# okhdr 👌
> a perceptually uniform hdr color storage format

## Specification

### Layout
```
31 29          19           9            0
+-+------------+------------+------------+
| |   b (10)   |   a (10)   |   L (10)   |
+-+------------+------------+------------+
```

### Implementation
```
// 10:10:10 Oklab packing (30 bits total)
// Layout: [ b:10 | a:10 | L:10 ]
//   can be stored e.g. in `rgb10a2uint` / `A2B10G10R10_UINT_PACK32` texture
// Ranges: L ∈ [0,2], a ∈ [-0.5,+0.5], b ∈ [-0.5,+0.5]
// Info: Encloses all of Rec. 2020, and HDR up to 1000 nits

uint32_t pack( float L, float a, float b ) {
    uint32_t Lbits = (uint32_t)roundf( ( L / 2.0f ) * 1023.0f );            //  10 bits
    uint32_t abits = (uint32_t)roundf( ( ( a + 0.5f ) / 1.0f ) * 1023.0f ); //  10 bits
    uint32_t bbits = (uint32_t)roundf( ( ( b + 0.5f ) / 1.0f ) * 1023.0f ); //  10 bits
    return ( bbits << 20 ) | ( abits << 10 ) | Lbits;
}

void unpack( uint32_t p, float* L, float* a, float* b ) {
    uint32_t Lbits =  p & 0x3FF;
    uint32_t abits = ( p >> 10 ) & 0x3FF;
    uint32_t bbits = ( p >> 20 ) & 0x3FF;
    *L = ( Lbits / 1023.0f ) * 2.0f;
    *a = ( abits / 1023.0f ) - 0.5f;
    *b = ( bbits / 1023.0f ) - 0.5f;
}
```

## Description
quantize unconstrained `oklab` source of truth to optimal ranges and step sizes.
considered `oklch` as well, but has multiple issues (lossy, error-prone wrap-around handling, little benefit in information density).

## JND
the just noticable difference.
- ΔE < 1.0: imperceptible (color‑critical work, e.g. design, print, film)
- ΔE < 2.0: differences are invisible in motion or at distance (general display / UI / LED animations)
- ΔE < 3.0: can be acceptable, if audience is far away and the context is dynamic (stage lighting, aerial light shows)

## Quantization
brute forced at `step = 0.01` for L, a, b respectively.
this corresponds to imperceptible differences in oklab lightness.
- sRGB / SDR: L \in [0,1] ~ 80 nits (D65 reference white)
- HDR10 / Dolby Vision: L \in [0,2] ~10k nits (1k monitor, 10k outdoor led board)
- Shader: L \in [0,4] ~1mil nits (sunlight)

### srgb
`L=1, a=0.25, b=0.31`
| packing      | bits | avg    | max     |
| ------------ | ---- | ------ | ------- |
| 9:8:8        | 25   | 0.1404 | 0.9153  |
| **8:8:8**    | 24   | 0.1769 | 1.6258  |
| 6:5:5        | 16   | 1.2431 | 9.1087  |
| 7:4:5        | 16   | 1.5234 | 8.3867  |
| 5:5:6        | 16   | 1.2376 | 12.5599 |
| 5:6:5        | 16   | 1.4513 | 15.3120 |

### p3
`L=1, a=0.4, b=0.4`
| packing      | bits | avg    | max     |
| ------------ | ---- | ------ | ------- |
| 9:9:9        | 27   | 0.0997 | 0.9746  |
| 9:8:9        | 26   | 0.1236 | 1.0893  |
| 9:9:8        | 26   | 0.1447 | 1.3501  |
| 10:8:8       | 26   | 0.1518 | 1.2311  |
| 9:7:8        | 24   | 0.2171 | 1.6471  |
| **8:8:8**    | 24   | 0.2006 | 2.0453  |
| 10:7:7       | 24   | 0.2965 | 2.4359  |

### hdr-2
`L=2, a=0.5, b=0.5`
| packing      | bits | avg    | max    |
| ------------ | ---- | ------ | ------ |
| 12:10:10     | 32   | 0.0458 | 0.7359 |
| 12:9:9       | 30   | 0.0884 | 1.5480 |
| **10:10:10** | 30   | 0.0661 | 0.9285 |
| 11:9:9       | 29   | 0.0918 | 1.5534 |
| 10:9:9       | 28   | 0.1027 | 1.7146 |
| 9:7:8        | 24   | 0.2662 | 3.6240 |
| 8:8:8        | 24   | 0.2656 | 3.7638 |
| 10:7:7       | 24   | 0.3547 | 5.5077 |
| 9:8:7        | 24   | 0.3201 | 5.6605 |

### hdr-4
`L=4, a=0.5, b=0.5`
| packing      | bits | avg    | max     |
| ------------ | ---- | ------ | ------- |
| **12:10:10** | 32   | 0.0449 | 1.0267  |
| 8:8:8        | 24   | 0.2989 | 6.2256  |
| 10:7:7       | 24   | 0.3373 | 6.4088  |
