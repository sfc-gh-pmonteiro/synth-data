# Synth Regions

Regional parameter reference for synthetic data generation.
Provides seasonality multipliers, holiday calendars, and business patterns per region.

## Holiday Calendar Schema

Each region's event table uses two separate multipliers:

| Column | Type | Purpose |
|--------|------|---------|
| `volume_multiplier` | NUMBER(4,2) | Affects **row count** per day (transaction volume). >1.0 = more transactions, <1.0 = fewer. |
| `value_multiplier` | NUMBER(4,2) | Affects **metric amounts** per transaction. >1.0 = higher amounts, <1.0 = lower. |

**Interpreting the Impact column below**: Each event shows its impact as volume/value pairs. Examples:
- `+3.0x retail` → `volume_multiplier: 3.0, value_multiplier: 1.0` (more transactions, same amount each)
- `-0.5x B2B` → `volume_multiplier: 0.5, value_multiplier: 1.0` (fewer transactions)
- `+1.4x retail` → `volume_multiplier: 1.2, value_multiplier: 1.2` (both more transactions AND higher basket)
- `+2.5x retail` → `volume_multiplier: 2.0, value_multiplier: 1.25` (heavy volume + moderate ticket increase)

**Default rule**: When only one multiplier is stated in the Impact column, split it as: `volume_multiplier = sqrt(impact)`, `value_multiplier = sqrt(impact)` (geometric split). For B2B dips, apply entirely to volume (fewer transactions, same invoice size): `volume_multiplier = impact, value_multiplier = 1.0`.

**Compound semantics**: When multiple events overlap on the same date, multiply them (Pattern #10 compound): `EXP(SUM(LN(multiplier)))`. This produces realistic stacking (e.g., "Weekend +1.1x" AND "CyberDay +3.5x" → 3.85x combined).

---

### us — United States

| Property | Value |
|----------|-------|
| Code | `us` |
| Currency | USD |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 ET |

**Monthly Activity Index**: 0.82, 0.88, 0.95, 1.00, 1.02, 1.00, 0.95, 0.98, 1.05, 1.05, 1.15, 1.15

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 1 | New Year's Day | -0.9x B2B |
| 3rd Mon Jan | MLK Day | -0.3x |
| 3rd Mon Feb | Presidents Day | -0.2x |
| Last Mon May | Memorial Day | -0.3x |
| Jul 4 | Independence Day | -0.5x B2B, +0.3x consumer |
| 1st Mon Sep | Labor Day | -0.3x |
| 4th Thu Nov | Thanksgiving | -0.8x B2B |
| Nov 25-28 | Black Friday/Cyber Monday | +3.0x retail |
| Dec 15-24 | Christmas shopping | +2.5x retail |
| Dec 25 | Christmas Day | -0.95x B2B |

---

### ca — Canada

| Property | Value |
|----------|-------|
| Code | `ca` |
| Currency | CAD |
| Fiscal Year Start | January (private) / April (govt) |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 ET/PT |

**Monthly Activity Index**: 0.85, 0.90, 0.95, 1.00, 1.00, 0.98, 0.92, 0.95, 1.02, 1.05, 1.10, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mon before May 25 | Victoria Day | -0.3x |
| Jul 1 | Canada Day | -0.4x |
| 1st Mon Aug | Civic Holiday | -0.2x |
| 2nd Mon Oct | Thanksgiving (CA) | -0.4x |
| Nov 25-28 | Black Friday (growing) | +1.5x retail |
| Dec 25 | Christmas | -0.95x B2B |
| Dec 26 | Boxing Day | +1.8x retail |

---

### eu-west — Western Europe (UK/DE/FR composite)

| Property | Value |
|----------|-------|
| Code | `eu-west` |
| Currency | EUR / GBP |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-17:30 CET/GMT |

**Monthly Activity Index**: 0.90, 0.95, 1.00, 1.00, 1.00, 0.95, 0.75, 0.75, 1.00, 1.05, 1.10, 1.05

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr (varies) | Easter | -0.4x (Fri-Mon) |
| May 1 | Labour Day (EU) | -0.3x |
| Late May | Spring bank holidays | -0.3x |
| Jul-Aug | Summer vacation | -0.25x B2B sustained |
| Nov-Dec | Christmas markets | +1.3x consumer |
| Dec 25-26 | Christmas/Boxing Day | -0.95x B2B |
| Dec 31 | New Year's Eve | -0.5x |

---

### latam-panama — Panama

| Property | Value |
|----------|-------|
| Code | `latam-panama` |
| Currency | USD |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 EST |

**Monthly Activity Index**: 1.00, 0.85, 1.00, 1.00, 0.95, 0.95, 0.95, 0.95, 0.98, 1.00, 0.90, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb (varies) | Carnival | +1.5x consumer, -0.4x B2B |
| Mar/Apr | Holy Week | -0.5x B2B |
| Nov 3-10 | Fiestas Patrias | -0.25x B2B |
| Dec 1-15 | Aguinaldo (13th month) | +0.85x all spending |
| Dec 25 | Christmas | -0.9x B2B |
| Jan-Apr | Dry season | +1.1x baseline activity |

---

### latam-brazil — Brazil

| Property | Value |
|----------|-------|
| Code | `latam-brazil` |
| Currency | BRL |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 BRT |

**Monthly Activity Index**: 0.85, 0.80, 0.95, 1.00, 1.00, 0.95, 1.00, 1.00, 1.00, 1.05, 1.15, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb/Mar (varies) | Carnival (5 days) | -0.3x B2B, +1.5x consumer |
| Jun 10-30 | Festas Juninas (São João) | +1.2x consumer (NE region) |
| Sep 7 | Independence Day | -0.2x |
| Nov (last Fri) | Black Friday (adopted) | +2.0x retail |
| Dec 25 | Christmas (summer) | +1.3x consumer |

> Note: Southern Hemisphere — summer = Dec-Feb, winter = Jun-Aug. No Christmas cold weather spike.

---

### latam-mexico — Mexico

| Property | Value |
|----------|-------|
| Code | `latam-mexico` |
| Currency | MXN |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 CST |

**Monthly Activity Index**: 0.90, 0.95, 1.00, 1.00, 1.00, 0.95, 0.95, 0.95, 1.00, 1.00, 1.15, 1.15

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb 5 | Constitution Day | -0.2x |
| Mar/Apr | Semana Santa | -0.4x B2B |
| Sep 16 | Independence Day | -0.3x |
| Nov 1-2 | Día de Muertos | +1.2x consumer |
| Nov 15-18 | Buen Fin (MX Black Friday) | +2.5x retail |
| Nov 20 | Revolution Day | -0.2x |
| Dec 1-24 | Aguinaldo spending | +1.4x retail |

---

### latam-guatemala — Guatemala

| Property | Value |
|----------|-------|
| Code | `latam-guatemala` |
| Currency | GTQ |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 CST |

**Monthly Activity Index**: 0.90, 0.92, 0.88, 0.85, 1.00, 1.00, 1.00, 1.10, 1.00, 1.00, 1.05, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr | Semana Santa (Antigua processions) | -0.6x B2B, +1.8x tourism/consumer in Antigua |
| Jun 30 | Día del Ejército | -0.2x |
| Aug 1-15 | Feria de Agosto (Guatemala City) | +1.3x consumer (capital) |
| Sep 15 | Independence Day | -0.4x B2B |
| Oct 20 | Revolution Day | -0.2x |
| Nov 1 | Día de Todos los Santos | +0.8x consumer (fiambres tradition) |
| Dec 1-24 | Aguinaldo/Bono 14 spending | +1.4x retail |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Bono 14 (July) + Aguinaldo (December) = two extra salary periods. July has a smaller spending bump (+0.3x).

---

### latam-el-salvador — El Salvador

| Property | Value |
|----------|-------|
| Code | `latam-el-salvador` |
| Currency | USD |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 CST |

**Monthly Activity Index**: 0.90, 0.92, 0.88, 0.85, 1.00, 1.00, 1.00, 1.15, 1.00, 1.00, 1.05, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr | Semana Santa | -0.5x B2B, +1.2x beach/travel |
| May 1 | Día del Trabajo | -0.3x |
| May 10 | Día de la Madre | +1.2x consumer |
| Aug 1-6 | Fiestas Agostinas (San Salvador) | +1.5x consumer (capital), -0.3x B2B |
| Sep 15 | Independence Day | -0.4x B2B |
| Nov 2 | Día de los Difuntos | -0.2x |
| Dec 1-24 | Aguinaldo spending | +1.4x retail |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Dollarized economy (USD since 2001). Remittance inflows (20%+ of GDP) spike Dec-Jan creating additional consumer spending.

---

### latam-honduras — Honduras

| Property | Value |
|----------|-------|
| Code | `latam-honduras` |
| Currency | HNL |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 CST |

**Monthly Activity Index**: 0.88, 0.90, 0.88, 0.85, 1.00, 1.05, 1.00, 1.00, 1.00, 0.95, 1.05, 1.25

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb (varies) | Carnival (La Ceiba) | +1.3x consumer (north coast) |
| Mar/Apr | Semana Santa | -0.5x B2B, +1.3x beach tourism |
| Jun 20-30 | Feria Juniana (San Pedro Sula) | +1.2x consumer (SPS metro) |
| Sep 15 | Independence Day | -0.3x |
| Oct 3-12 | Semana Morazánica | -0.4x B2B (extended break) |
| Dec 1-24 | Décimo tercer mes + Aguinaldo | +1.5x retail |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Hurricane season Jun-Nov. Major disruption events (Eta/Iota type) can cause -0.6x for weeks in affected regions.

---

### latam-costa-rica — Costa Rica

| Property | Value |
|----------|-------|
| Code | `latam-costa-rica` |
| Currency | CRC |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 CST |

**Monthly Activity Index**: 0.92, 0.95, 0.90, 0.87, 1.00, 1.00, 1.00, 1.00, 1.00, 1.02, 1.08, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr | Semana Santa | -0.5x B2B, +1.4x beach/tourism |
| Jul 25 | Anexión de Guanacaste | -0.2x |
| Aug 2 | Día de la Virgen de los Ángeles | -0.2x |
| Sep 15 | Independence Day | -0.3x |
| Dec 1 | Abolición del Ejército | -0.1x |
| Dec 1-24 | Aguinaldo spending | +1.3x retail |
| Dec 25-Jan 1 | Christmas/New Year (vacation) | -0.6x B2B, +1.5x tourism |

> Note: High-cost-of-living LATAM country. Tourism dual season: dry (Dec-Apr, +1.3x) and green/rainy (May-Nov). Tech sector (Intel/services) follows US Q4 patterns.

---

### latam-colombia — Colombia

| Property | Value |
|----------|-------|
| Code | `latam-colombia` |
| Currency | COP |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-18:00 COT |

**Monthly Activity Index**: 0.88, 0.90, 0.95, 1.00, 1.00, 1.05, 1.00, 1.00, 1.00, 1.00, 1.10, 1.15

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 1-10 | Extended New Year (puente) | -0.4x B2B |
| Feb (varies) | Carnaval de Barranquilla (4 days) | +1.5x consumer (Caribe), -0.4x B2B (Caribe only) |
| Mar/Apr | Semana Santa | -0.5x B2B |
| Jun (mid-month) | Prima de servicios (mid-year bonus) | +1.3x consumer |
| Jun/Jul (scattered) | Día sin IVA (tax-free day) | +5.0x retail (single-day spikes) |
| Jul 20 | Independence Day | -0.3x |
| Aug 7 | Battle of Boyacá | -0.2x |
| Nov (scattered) | Día sin IVA (2nd event) | +5.0x retail (single-day spike) |
| Dec 1-24 | Prima navideña spending | +1.4x retail |
| Dec 7 | Día de las Velitas (start of Christmas season) | +1.0x consumer |

> Note: Colombia has ~18 holidays/year via Ley Emiliani (moved to Mondays creating puentes). Día sin IVA (0% sales tax days) create extreme single-day retail spikes — 3 days/year, dates announced by government. Prima de servicios = extra half-salary in June + December.

---

### latam-venezuela — Venezuela

| Property | Value |
|----------|-------|
| Code | `latam-venezuela` |
| Currency | VES |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 VET |

**Monthly Activity Index**: 0.90, 0.85, 0.95, 1.00, 1.00, 1.00, 1.00, 1.00, 1.00, 1.00, 1.05, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb/Mar (varies) | Carnival (Mon-Tue before Ash Wed) | -0.5x B2B, +1.0x consumer |
| Mar/Apr | Semana Santa | -0.5x B2B |
| Apr 19 | Declaration of Independence | -0.2x |
| Jun 24 | Battle of Carabobo | -0.2x |
| Jul 5 | Independence Day | -0.3x |
| Jul 24 | Simón Bolívar's Birthday | -0.2x |
| Dec 1-24 | Christmas spending | +1.2x consumer |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Hyperinflation economy. Revenue/price data requires constant adjustment. USD parallel market dominates real transactions. Consider generating data in USD equivalent for realism. Utility payments (aguinaldo) paid in Dec but purchasing power varies wildly.

---

### latam-ecuador — Ecuador

| Property | Value |
|----------|-------|
| Code | `latam-ecuador` |
| Currency | USD |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 ECT |

**Monthly Activity Index**: 0.90, 0.88, 0.92, 0.90, 1.00, 1.00, 1.00, 1.00, 1.00, 1.00, 1.10, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb/Mar (varies) | Carnival (Mon-Tue, water fights) | -0.4x B2B, +1.2x consumer |
| Mar/Apr | Semana Santa | -0.5x B2B |
| May 24 | Battle of Pichincha | -0.2x |
| Jul 25 | Fundación de Guayaquil | -0.2x (Guayaquil only) |
| Aug 10 | Primer Grito de Independencia | -0.3x |
| Oct 9 | Independence of Guayaquil | -0.2x (coast) |
| Nov 2-3 | Día de los Difuntos + Independencia de Cuenca | -0.3x |
| Dec 1-24 | Décimo tercer sueldo spending | +1.4x retail |
| Dec 6 | Fiestas de Quito (Quito metro) | +1.2x consumer (Quito), -0.2x B2B |

> Note: Dollarized economy (USD since 2000). Décimo tercer sueldo (Dec) + Décimo cuarto sueldo (varies by region, coast=Mar, sierra=Aug) = two extra salary payments driving spending spikes. Costa vs Sierra regions have different holiday patterns.

---

### latam-peru — Peru

| Property | Value |
|----------|-------|
| Code | `latam-peru` |
| Currency | PEN |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 PET |

**Monthly Activity Index**: 0.90, 0.92, 0.95, 1.00, 1.00, 0.98, 1.20, 1.00, 1.00, 1.00, 1.05, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr | Semana Santa | -0.4x B2B |
| May 1 | Día del Trabajo | -0.2x |
| Jun 24 | Inti Raymi (Cusco) | +1.3x tourism (Cusco region) |
| Jun 29 | San Pedro y San Pablo | -0.2x |
| Jul 15-31 | Gratificación de julio (spending period) | +1.5x consumer |
| Jul 28-29 | Fiestas Patrias | -0.6x B2B, +1.2x pre-holiday consumer |
| Oct 8 | Battle of Angamos | -0.2x |
| Nov (quarterly) | CyberWow (e-commerce event) | +2.0x online retail |
| Dec 1-24 | Gratificación de diciembre spending | +1.5x retail |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Gratificación = full extra salary paid in July + December (not half like some countries). Creates two massive consumer spending peaks. CyberWow runs quarterly (Mar, May, Jul, Nov) with +2.0x each event.

---

### latam-bolivia — Bolivia

| Property | Value |
|----------|-------|
| Code | `latam-bolivia` |
| Currency | BOB |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-18:00 BOT |

**Monthly Activity Index**: 0.90, 0.88, 0.92, 0.95, 1.00, 1.00, 1.00, 1.00, 1.00, 1.00, 1.10, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb (varies) | Carnaval de Oruro (UNESCO heritage) | +1.5x consumer (Oruro/LP), -0.4x B2B national |
| Mar/Apr | Semana Santa | -0.4x B2B |
| May 1 | Día del Trabajo | -0.2x |
| Jun 21 | Año Nuevo Aymara (Willkakuti) | -0.2x (highlands) |
| Aug 6 | Independence Day | -0.4x B2B |
| Nov 2 | Día de los Muertos | -0.2x |
| Dec 1-24 | Aguinaldo spending | +1.3x retail |
| Dec 25 | Christmas | -0.9x B2B |

> Note: Partially Southern Hemisphere climate (highlands vs. tropical lowlands). Highlands (La Paz, Oruro) have mild inverted seasons; lowlands (Santa Cruz) are tropical. Aguinaldo legally due by Dec 20.

---

### latam-chile — Chile

| Property | Value |
|----------|-------|
| Code | `latam-chile` |
| Currency | CLP |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 CLT |

**Monthly Activity Index**: 0.85, 0.88, 0.95, 1.00, 1.00, 1.05, 1.00, 1.00, 1.25, 1.00, 1.05, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar/Apr | Semana Santa | -0.3x B2B |
| May 1 | Día del Trabajo | -0.2x |
| May 21 | Día de las Glorias Navales | -0.2x |
| Late May/Jun | CyberDay (3 days, biggest e-commerce event) | +3.5x online retail |
| Sep 18-19 | Fiestas Patrias (Dieciocho) | -0.8x B2B, +3.0x consumer pre-fiestas (Sep 10-17) |
| Sep 18-25 | Semana de la chilenidad (extended) | -0.6x B2B sustained |
| Oct (varies) | CyberMonday CL | +2.0x online retail |
| Oct 31 | Día de las Iglesias Evangélicas | -0.1x |
| Dec 1-24 | Aguinaldo spending | +1.4x retail |
| Dec 25 | Christmas (summer) | -0.7x B2B |

> Note: Southern Hemisphere — summer Dec-Feb, winter Jun-Aug. Fiestas Patrias (Sep 18-19) is Chile's most important holiday — country essentially stops for 5-10 days. Known as "Dieciocho." CyberDay (May/Jun) outperforms Black Friday for e-commerce.

---

### latam-argentina — Argentina

| Property | Value |
|----------|-------|
| Code | `latam-argentina` |
| Currency | ARS |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 ART |

**Monthly Activity Index**: 0.85, 0.82, 0.95, 1.00, 1.05, 1.10, 1.05, 1.00, 1.00, 1.00, 1.10, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb/Mar (varies) | Carnival (Mon-Tue) | -0.3x B2B |
| Mar 24 | Día de la Memoria | -0.2x |
| Mar/Apr | Semana Santa | -0.4x B2B |
| May (3 days) | Hot Sale (e-commerce event) | +2.5x online retail |
| May 25 | Revolución de Mayo | -0.3x |
| Jun 15-30 | Aguinaldo 1ra cuota spending | +1.3x consumer |
| Jul 9 | Independence Day | -0.3x |
| Nov (3 days) | CyberMonday AR | +2.0x online retail |
| Dec 1-24 | Aguinaldo 2da cuota spending | +1.5x retail |
| Dec 25 | Christmas (summer) | -0.7x B2B |

> Note: Southern Hemisphere — summer Dec-Feb, winter Jun-Aug. Aguinaldo split in two: 1st half due June 30, 2nd half due Dec 18. Both create spending spikes. High inflation economy — synthetic revenue data should consider annual 50-100%+ price increases for realism (or generate in USD equivalent). Hot Sale (May) and CyberMonday (Nov) are the two major e-commerce events.

**Phone Area Codes (by province):**
| Province | Code | Province | Code |
|----------|------|----------|------|
| Buenos Aires / CABA | 11 | Córdoba | 351 |
| Santa Fe (Rosario) | 341 | Santa Fe (capital) | 342 |
| Mendoza | 261 | Tucumán | 381 |
| Entre Ríos | 343 | Salta | 387 |
| Misiones | 376 | Mar del Plata | 223 |

---

### latam-uruguay — Uruguay

| Property | Value |
|----------|-------|
| Code | `latam-uruguay` |
| Currency | UYU |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-18:00 UYT |

**Monthly Activity Index**: 0.82, 0.80, 0.95, 1.00, 1.00, 1.05, 1.00, 1.00, 1.00, 1.00, 1.10, 1.20

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan-Mar (40+ days) | Carnaval (longest in the world) | +1.2x consumer (entertainment), sustained |
| Mar/Apr | Semana de Turismo (Holy Week renamed) | -0.5x B2B, +1.5x tourism/travel |
| May 1 | Día del Trabajo | -0.2x |
| Jun 19 | Natalicio de Artigas | -0.2x |
| Jun 30 | Aguinaldo 1ra mitad | +1.2x consumer |
| Aug 25 | Declaratoria de Independencia | -0.2x |
| Nov (varies) | CyberMonday UY (growing) | +1.5x online retail |
| Dec 1-20 | Aguinaldo 2da mitad spending | +1.4x retail |
| Dec 25 | Christmas (summer) | -0.7x B2B |

> Note: Southern Hemisphere — summer Dec-Feb, winter Jun-Aug. Carnaval runs ~40 days (Jan-Mar) with tablados (stages) across Montevideo — the world's longest carnival. Semana de Turismo = Semana Santa renamed (secular country). Aguinaldo split Jun + Dec like Argentina.

---

### latam-paraguay — Paraguay

| Property | Value |
|----------|-------|
| Code | `latam-paraguay` |
| Currency | PYG |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 PYT |

**Monthly Activity Index**: 0.88, 0.90, 0.92, 0.95, 1.00, 1.02, 1.00, 1.00, 1.00, 1.00, 1.05, 1.25

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Feb/Mar (varies) | Carnival | -0.3x B2B, +1.0x consumer |
| Mar 1 | Día de los Héroes | -0.2x |
| Mar/Apr | Semana Santa | -0.5x B2B |
| May 1 | Día del Trabajo | -0.2x |
| May 14-15 | Independencia del Paraguay | -0.3x |
| Jun 12 | Paz del Chaco | -0.2x |
| Aug 15 | Fundación de Asunción | -0.2x |
| Dec 1-24 | Aguinaldo spending | +1.5x retail |
| Dec 25 | Christmas (summer) | -0.8x B2B |

> Note: Southern Hemisphere — summer Dec-Feb, winter Jun-Aug. Large informal economy. Cross-border shopping from Ciudad del Este (Brazilian/Argentine shoppers) creates separate retail patterns on border. Aguinaldo due by Dec 31.

---

### latam-dominican-republic — Dominican Republic

| Property | Value |
|----------|-------|
| Code | `latam-dominican-republic` |
| Currency | DOP |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 AST |

**Monthly Activity Index**: 1.05, 1.10, 1.05, 1.00, 0.95, 0.95, 0.95, 0.90, 0.88, 0.92, 1.00, 1.15

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 6 | Día de Reyes | +0.8x consumer |
| Jan 21 | Día de la Altagracia | -0.2x |
| Jan 26 | Día de Duarte | -0.2x |
| Feb (every Sunday) | Carnival (parades each Sunday) | +1.3x consumer |
| First Sun Mar | Carnival Nacional (main event) | +1.8x consumer (national) |
| Mar/Apr | Semana Santa (beach exodus) | -0.5x B2B, +2.0x tourism/beach |
| Aug 16 | Restoration Day | -0.3x |
| Dec 1-24 | Regalía pascual (bonus) spending | +1.4x retail |
| Dec-Apr | Tourism high season | +1.4x hospitality sustained |

> Note: Tourism-driven economy (~15% GDP). Dec-Apr = high season (weather + US/Canadian visitors). Hurricane season Aug-Oct creates supply chain and insurance risk events. Regalía pascual = Christmas bonus, legally required.

---

### latam-puerto-rico — Puerto Rico

| Property | Value |
|----------|-------|
| Code | `latam-puerto-rico` |
| Currency | USD |
| Fiscal Year Start | July (PR government) / January (federal/private) |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 AST |

**Monthly Activity Index**: 1.05, 0.95, 0.95, 1.00, 1.00, 0.95, 0.95, 0.90, 0.88, 0.95, 1.15, 1.25

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 6 | Día de Reyes (Three Kings) | +1.5x consumer (bigger than Dec 25 for gifts) |
| Mar/Apr | Semana Santa | -0.3x B2B |
| Jul 4 | US Independence Day | -0.3x |
| Jul 25 | Constitución de PR | -0.2x |
| Nov (4th Thu) | Thanksgiving (adopted) | -0.5x B2B |
| Nov 25-28 | Black Friday | +2.5x retail |
| Nov 30 | Cyber Monday | +1.8x online retail |
| Dec 1-24 | Bono de Navidad spending | +1.5x retail |
| Dec 25-Jan 6 | Extended Christmas season (Parrandas) | +1.2x consumer sustained |

> Note: US territory with distinct cultural patterns. Bono de Navidad (Christmas bonus) legally mandated. Three Kings Day (Jan 6) is culturally larger than Christmas morning for gift-giving. Hurricane season Jun-Nov — major economic disruptions (Maria 2017 type). Follows US federal holidays PLUS local PR holidays.

---

### latam-cuba — Cuba

| Property | Value |
|----------|-------|
| Code | `latam-cuba` |
| Currency | CUP |
| Fiscal Year Start | January |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:00-17:00 CST |

**Monthly Activity Index**: 0.95, 0.95, 1.00, 1.00, 1.00, 1.00, 1.05, 1.00, 0.95, 0.95, 1.05, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 1 | Triunfo de la Revolución | -0.3x |
| Jan 2 | Día de la Victoria | -0.2x |
| May 1 | Día del Trabajo (major state event) | -0.4x |
| Jul 25-27 | Días de la Rebeldía Nacional | -0.3x |
| Jul (Santiago) | Carnival de Santiago de Cuba | +1.3x consumer (eastern Cuba) |
| Oct 10 | Inicio de las Guerras de Independencia | -0.2x |
| Dec 1-24 | Christmas spending (restored 1998) | +1.1x consumer |
| Nov-Apr | Tourism high season | +1.5x hospitality |

> Note: Centralized economy — limited free-market patterns. Tourism sector (hotels, paladares) follows market dynamics; state sector does not. Dual-currency system (CUP + MLC) complicates price modeling. Private sector (MSMEs, since 2021) growing but small. Generate tourism-related data in USD/EUR equivalent for realism.

---

### apac-japan — Japan

| Property | Value |
|----------|-------|
| Code | `apac-japan` |
| Currency | JPY |
| Fiscal Year Start | April |
| Business Days | Mon-Fri |
| Typical Work Hours | 09:00-20:00 JST (long hours culture) |

**Monthly Activity Index**: 0.70, 0.95, 1.30, 1.10, 0.75, 1.00, 1.05, 0.80, 1.00, 1.00, 1.05, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 1-3 | Shogatsu (New Year) | -0.7x all |
| Apr 29-May 5 | Golden Week | -0.4x B2B, +1.3x travel |
| Aug 13-16 | Obon | -0.3x B2B |
| Mar 20-31 | Fiscal year-end rush | +1.3x B2B |
| Dec 1-25 | Year-end shopping | +1.5x retail |
| Dec 28-Jan 3 | Nenmatsu-Nenshi | -0.5x B2B |

---

### apac-india — India

| Property | Value |
|----------|-------|
| Code | `apac-india` |
| Currency | INR |
| Fiscal Year Start | April |
| Business Days | Mon-Sat (many sectors) |
| Typical Work Hours | 09:30-18:30 IST |

**Monthly Activity Index**: 0.95, 0.95, 1.05, 1.00, 1.00, 0.85, 0.85, 0.90, 0.95, 1.20, 1.20, 1.00

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Mar (varies) | Holi | -0.2x B2B |
| Mar 31 | FY-end rush | +1.4x B2B |
| Jun-Sep | Monsoon season | -0.2x logistics/construction |
| Aug 15 | Independence Day | -0.3x |
| Oct/Nov (varies) | Navratri + Diwali | +2.0x consumer, -0.3x B2B (holidays) |
| Nov (post-Diwali) | Wedding season begins | +1.5x consumer |
| Jan 26 | Republic Day | -0.2x |

---

### apac-australia — Australia

| Property | Value |
|----------|-------|
| Code | `apac-australia` |
| Currency | AUD |
| Fiscal Year Start | July |
| Business Days | Mon-Fri |
| Typical Work Hours | 08:30-17:00 AEST |

**Monthly Activity Index**: 0.85, 0.90, 1.00, 1.00, 1.05, 1.30, 0.95, 0.95, 1.00, 1.00, 1.05, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Jan 26 | Australia Day | -0.3x |
| Mar/Apr | Easter | -0.4x |
| Apr 25 | ANZAC Day | -0.3x |
| Jun 1-30 | EOFY rush | +1.3x B2B (tax spending) |
| Dec 25-Jan 5 | Christmas + summer break | +1.4x retail, -0.6x B2B |
| Nov (last Fri) | Black Friday (adopted) | +1.5x retail |

> Note: Southern Hemisphere — summer = Dec-Feb, winter = Jun-Aug. Christmas is a summer/outdoor holiday.

---

### mena-uae — UAE

| Property | Value |
|----------|-------|
| Code | `mena-uae` |
| Currency | AED |
| Fiscal Year Start | January |
| Business Days | Sun-Thu (weekend = Fri-Sat) |
| Typical Work Hours | 08:00-17:00 GST |

**Monthly Activity Index**: 1.05, 1.00, 0.80, 0.85, 1.00, 0.75, 0.70, 0.75, 1.00, 1.10, 1.10, 1.10

| Date/Period | Event | Impact |
|-------------|-------|--------|
| Ramadan (~30 days) | Fasting month | -0.3x daytime B2B, +0.5x evening retail |
| Eid al-Fitr (3 days) | End of Ramadan | +1.5x consumer |
| Eid al-Adha (4 days) | Sacrifice feast | +1.2x consumer, -0.5x B2B |
| Jun-Aug | Extreme summer heat | -0.4x outdoor/construction |
| Dec 2-3 | UAE National Day | +1.3x consumer, -0.3x B2B |
| Dec-Jan | Winter tourist season | +1.4x hospitality |

> **NOTE — Ramadan Date Shifting**: Ramadan follows the Islamic lunar calendar and shifts ~11 days earlier each Gregorian year. Reference dates: 2024 = Mar 11–Apr 9, 2025 = Feb 28–Mar 29, 2026 = Feb 17–Mar 18, 2027 = Feb 7–Mar 8. For generation: compute Ramadan start as `2024-03-11` minus `(year - 2024) * 11 days` (approximate; actual varies ±1-2 days).
