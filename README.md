# HelioDual

A dual-axis solar tracker that derives a **two-axis pointing error from a single photoresistor**.

Conventional trackers read the light gradient in *space* — four sensors around a shade fin, driving the difference between them to zero. HelioDual has one sensor, so it reads the gradient in *time*: the panel dithers a few degrees on a known path, and the phase of the resulting flicker says which way the source is.

Built for HackGT 13 (hardware track) on an Arduino Uno with two SG90 servos and a KY-018 light module.

---

## How it works

### Direction from a single brightness reading

One photoresistor tells you how bright it is, not where the light is. Moving while measuring is what converts brightness into direction.

The platform sways along a sine path. Multiply the light signal by that same sine and sum over a full cycle:

- **Off to one side** — the signal rises as the platform swings toward the source. Products are mostly positive, the sum is positive.
- **Off to the other side** — the signal is inverted. The sum is negative.
- **On the peak** — light rises on *both* halves of the sway. The signal comes back at twice the dither frequency and the sum cancels to zero.

Zero correlation means centred, which is the one thing a single brightness reading can never tell you on its own. This is synchronous detection — the operating principle of a lock-in amplifier.

### Two axes from one sensor

Roll dithers at 2 cycles per measurement window, pitch at 3. Sines with different whole-number cycle counts over a common window are orthogonal, so each axis' reference integrates the other's motion to exactly zero. One sensor, two independent channels, no time-multiplexing.

The counts are also **coprime on purpose**. Gear backlash distorts each sine and generates harmonics — roll's land at 4, 6, 8; pitch's at 6, 9. Neither ever falls on the other's fundamental. A pair like 2 and 4 would dump roll's second harmonic straight into pitch's channel every time the gears rattled.

A useful side effect of narrowband detection: correlating against a 0.67 Hz reference rejects almost everything else, so 120 Hz ripple from mains-powered room lighting integrates away.

### Quadrature detection

The photoresistor's response time and mechanical lag shift the signal out of phase with the command. Correlating against sine alone recovers only `cos(φ)` of the true amplitude, and the faster axis loses more — which would make pitch quietly less sensitive than roll for no visible reason.

Correlating against cosine as well captures what leaked into quadrature. The magnitude of I and Q together is independent of phase; the sign of I still gives direction while lag stays under a quarter cycle.

### Acquisition

Outside the collimator's acceptance cone the signal is flat and there is no gradient to correlate, so tracking cannot begin from an arbitrary orientation. A serpentine raster sweeps the full travel and takes the brightest point as a starting estimate.

That coarse pass is fast and therefore lag-biased: the sensor reports a rise late, so the apparent peak sits behind the true one. Each axis is then refined by **sweeping in both directions and averaging** — opposing passes carry equal and opposite bias, so their mean is unbiased for any sensor response time. A single pass can only reduce that bias by sweeping slower; it can never remove it.

---

## Hardware

| | |
|---|---|
| Controller | Arduino Uno (ATmega328P) |
| Actuation | 2 × SG90 servo — roll on **D9**, pitch on **D10** |
| Sensor | 1 × KY-018 photoresistor module — `S` → **A0**, middle → **5V**, `−` → **GND** |
| Status | Onboard LED (D13): solid = locked, flickering = searching |
| Dependencies | `Servo.h` only |

### The collimator is not optional

A bare photoresistor sees the whole room. Rotating it a few degrees barely changes what reaches it, and the gradient is buried in noise.

Fit a **2–3 cm opaque tube** over the sensor — a black drinking straw works — and mount it on the moving platform along its normal. Acceptance half-angle is `atan(bore / length)`; a 6 mm bore at 25 mm gives about ±13°.

If the tube is light-coloured, blacken the bore. A reflective inner wall bounces off-axis light down to the sensor and undoes the entire point.

### Sensor configuration

```c
#define USE_INTERNAL_PULLUP 0   // 0 for a module with its own divider resistor
#define INVERT_READING      1   // 1 when raw ADC falls as light rises
```

KY-018 board revisions differ in divider orientation, and much of the documentation online contradicts itself. Verify with the `m` light meter — cover the sensor and check that brightness *falls* — rather than trusting a datasheet.

---

## Serial commands

115200 baud.

| Key | Action |
|---|---|
| `s` | Start / stop tracking |
| `f` | Acquire — sweep for the source, then refine |
| `c` | Centre both axes at 90° |
| `m` | Light meter — verify sensor polarity and collimation |
| `b` | Benchmark: fixed panel vs tracked panel |
| `k` | Measure gear backlash |
| `p` | Measure pointing repeatability |
| `l` | Toggle CSV logging during benchmark |
| `+` `-` | Dither amplitude |
| `[` `]` | Loop gain |
| `<` `>` | Light floor |
| `?` | Help |

---

## Built-in instrumentation

The firmware measures its own mechanism rather than asserting anything about it.

### `k` — backlash, separated from sensor lag

Reversing direction turns the servo output shaft through the gear lash before the platform follows. Parking on a steep flank of the light curve, reversing, and counting commanded travel until the reading responds gives the lash — but that figure also contains the photoresistor's response time, which is indistinguishable from mechanical slop in a single measurement.

Sweeping at two different rates separates them:

```
apparent = lash + v · τ
```

Lash is rate-independent. Sensor lag contributes an angle proportional to sweep rate. Two rates solve for both, and the zero-rate intercept is the backlash alone — with the cell's time constant falling out as a bonus.

The result auto-sets the tracking dither to 1.6× the worst axis, because **a dither narrower than the backlash moves the command without moving the platform**. The correlation then reads flat, which is indistinguishable from being perfectly centred — a confident lock onto nothing. Every fourth locked window also re-checks at full amplitude to catch exactly that case.

### `p` — pointing repeatability

"It finds the light" is not a specification. This displaces the platform 8° off target, lets it re-converge, and records where it lands — four times, alternating which side each axis approaches from, so hysteresis shows up in the spread instead of hiding in a consistent offset.

### `b` — energy gain over a fixed panel

A tracker exists to raise the light collected over a day, so that is what gets measured: the panel is parked fixed while a lamp is swept through an arc, then the identical arc is repeated with tracking enabled, and the mean irradiance compared.

The tracked figure **includes the dither excursions**, so it is charged for the energy the search itself costs. Arc reproduction is manual, which limits repeatability to roughly ±10%.

`l` first streams per-sample CSV between `--- CSV BEGIN ---` / `--- CSV END ---` markers for plotting.

---

## Measured

Conditions: phone flashlight at ~40 cm, room lights dimmed.

| Quantity | Value | Method |
|---|---|---|
| **Pointing repeatability** | **± 1.54 °** | 4 approaches, alternating side per axis |
| Optical sensitivity | 28.0 / 24.5 counts/° | roll / pitch, on the flank |
| Angular resolution | ~0.04 °/count | derived from the above |
| Gear backlash | 2.5 – 3.3 ° | upper bound, single sweep rate |

The optics resolve roughly two orders of magnitude finer than the mechanism does. This design is **mechanically limited, not sensor limited** — the next meaningful improvement is a metal-gear servo, not a better photodetector. Repeatability landing at ±1.54° against 2.5–3.3° of measured lash is consistent with that.

### On the energy-gain figure

The `b` benchmark reports the irradiance gain of tracking over a fixed panel, and **the number it produces should not be quoted without its caveat.**

The sensor is collimated to a ~26° cone. A PV module has a cosine response out to ±90°, so the fixed reference here falls off far faster than real hardware would, and the measured ratio overstates the true benefit by a wide margin. Published figures for dual-axis tracking against fixed installations are in the region of 30–40% annually.

The benchmark is a useful check that the control loop keeps the panel on target through a moving arc. It is not a claim about photovoltaic yield.

### On the backlash figure

Reversing direction turns the servo output shaft through the gear lash before the platform follows, and the firmware measures this directly (`k`). The complication is that the photoresistor's own response time is indistinguishable from mechanical slop in a single measurement, so a single-rate figure is only ever an upper bound.

Measuring at two sweep rates should separate them, since lash is rate-independent while sensor lag contributes an angle proportional to rate. In practice the extrapolation is only as good as the settled reference it starts from — an early version sampled before the cell had finished responding and returned physically impossible results (0.00° of backlash on one axis, and a sensor time constant that disagreed between two axes sharing one sensor). The reference is now held until the reading stops drifting, and detection is directional.

---

## Known limitations

**It can be fooled by a local maximum.** Gradient ascent climbs the nearest peak, not the highest one. A bright window or a second lamp can capture it. The coarse acquisition sweep mitigates this; a production tracker would cross-check against a computed sun position from GPS and clock.

**The panel is never quite still.** The dither is not a defect to be tuned away — it *is* the measurement. Without perturbing itself, a single sensor has no way to know which direction is brighter.

**Tracking rate is ~10 °/s**, set by servo settling time and the 3 s measurement window. For context, the sun moves at 0.004 °/s, so the design is overbuilt for its nominal application by roughly three orders of magnitude.

---

## License

MIT — see [LICENSE](LICENSE).
