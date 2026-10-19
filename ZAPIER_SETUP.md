# 6-hourly Singapore PSI email — Zapier setup (free plan)

**Card first:** paste the five-line card from `index.html` onto your page *before* the first run.

## Why this design
The free plan allows only **one trigger + one action**, and "Webhooks by Zapier" is a paid app.
So the trigger itself fetches the data: **Code by Zapier → Run JavaScript (trigger)**.
Zapier polls it every 15 min, but it only fires a *new* item when the `id` changes —
and the `id` is the 6-hour slot (00:00, 06:00, 12:00, 18:00 SGT). Result: ~4 emails a day.

Data source: NEA's official PSI feed via data.gov.sg (the same data haze.gov.sg shows).

## Steps
1. zapier.com → sign up with a **personal** Google account (not SMU). Free plan, no card.
2. **Create Zap → Trigger:** *Code by Zapier* → event *Run Javascript*.
3. Paste the code below into **Code**. Leave Input Data empty. **Test trigger** — you should see `psi_national`, `west`, `east`, etc.
4. **Action:** *Gmail → Send Email* (or *Email by Zapier → Send Outbound Email*).
   - To: your own personal email
   - Subject: `PSI {{band}} – {{psi_national}} (Singapore, {{slot}})`
   - Body:
     ```
     24-hr PSI as at {{reading_time}}
     National (max of regions): {{psi_national}} — {{band}}
     North {{north}} | South {{south}} | East {{east}} | West {{west}} | Central {{central}}
     PM2.5 24-hr (max): {{pm25_max}}
     Advice: {{advice}}
     Source: NEA via data.gov.sg · haze.gov.sg
     ```
5. Test the action → check inbox → **Publish**. Name it "6-hourly PSI".

## Trigger code
```javascript
// Code by Zapier trigger: returns one item per 6-hour slot (SGT).
const SLOT_HOURS = 6;

const now = new Date(Date.now() + 8 * 3600 * 1000); // shift to SGT
const slotHour = Math.floor(now.getUTCHours() / SLOT_HOURS) * SLOT_HOURS;
const slot = `${now.toISOString().slice(0, 10)} ${String(slotHour).padStart(2, '0')}:00 SGT`;

let readings, readingTime, pm25;
try {
  const res = await fetch('https://api-open.data.gov.sg/v2/real-time/api/psi');
  const json = await res.json();
  const item = json.data.items[0];
  readings = item.readings.psi_twenty_four_hourly;
  pm25 = item.readings.pm25_twenty_four_hourly;
  readingTime = item.updatedTimestamp || item.timestamp;
} catch (e) {
  // Fallback to the older v1 endpoint
  const res = await fetch('https://api.data.gov.sg/v1/environment/psi');
  const json = await res.json();
  const item = json.items[0];
  readings = item.readings.psi_twenty_four_hourly;
  pm25 = item.readings.pm25_twenty_four_hourly;
  readingTime = item.update_timestamp || item.timestamp;
}

if (!readings) throw new Error('No PSI readings returned');

const regions = ['north', 'south', 'east', 'west', 'central'];
const values = regions.map(r => readings[r]).filter(v => typeof v === 'number');
const national = Math.max(...values);

const band = national <= 50 ? 'Good'
  : national <= 100 ? 'Moderate'
  : national <= 200 ? 'Unhealthy'
  : national <= 300 ? 'Very Unhealthy' : 'Hazardous';

const advice = {
  'Good': 'Normal activities.',
  'Moderate': 'Normal activities.',
  'Unhealthy': 'Reduce prolonged or strenuous outdoor activity.',
  'Very Unhealthy': 'Avoid prolonged or strenuous outdoor activity.',
  'Hazardous': 'Minimise outdoor activity.'
}[band];

const pm25Values = pm25 ? regions.map(r => pm25[r]).filter(v => typeof v === 'number') : [];

output = [{
  id: slot, // Zapier dedupes on id -> fires once per slot
  slot,
  reading_time: readingTime,
  psi_national: national,
  band,
  advice,
  north: readings.north, south: readings.south, east: readings.east,
  west: readings.west, central: readings.central,
  pm25_max: pm25Values.length ? Math.max(...pm25Values) : 'n/a'
}];
```

## Step 2 checks (for your page)
- **3 runs:** after ~18 hours you'll have 3+. For each, compare the email to **Zap History** (Zaps → History) and to haze.gov.sg. Screenshot each (crop your email address).
- **Tricky case (write the expectation *before* running):** e.g. "If the API returns a region as missing/null, I expect the national figure to still be the max of the rest, not `NaN`/`-Infinity`." Or: "If the API is down, I expect the run to error, not send an empty email."
- **Fix + rerun:** e.g. the first version used `Math.max(...Object.values(readings))`, which includes the `national` key on some feeds / breaks on nulls → fixed by filtering to the five numeric regions (already in the code above). Re-run the trigger test and screenshot.
- **Export history:** follow the Week 7 page "How to export your Zap history as a JSON file", save as
  **`PS5_Tiwari_Sanesh_zap_history.json`**. Send it on eLearn only — **do not commit it to GitHub**
  (`.gitignore` already excludes it).
- **Turn the Zap off after class on Wed 21 Oct.**
