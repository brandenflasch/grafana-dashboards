# Emporia Vue Energy Monitoring with Home Assistant & Grafana

A complete guide to setting up real-time whole-home energy monitoring using Emporia Vue hardware, Home Assistant, InfluxDB, and Grafana.

## System Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Emporia Vue   │────▶│  Emporia Cloud  │────▶│    Vuegraf      │
│   (Hardware)    │     │      API        │     │  (Python app)   │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    Grafana      │◀────│    InfluxDB     │◀────│  Home Assistant │
│   Dashboard     │     │   (time-series) │     │     (host)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## Hardware Requirements

### Emporia Vue Gen 2
- Whole-home energy monitor
- Supports up to 16 circuit sensors (50A and 200A CT clamps)
- Requires 2.4GHz WiFi connection
- Cloud-connected (data accessed via API)

### Circuit Configuration Example

**Main Panel (200A service):**
| Circuit | Sensor | Description |
|---------|--------|-------------|
| Main-TotalUsage | Built-in | Total household consumption |
| Main-1 | CT Clamp | Family Room |
| Main-2 | CT Clamp | Washer |
| Main-5 | CT Clamp | Kitchen/Garage Lights |
| Main-9 | CT Clamp | Garage/Outside |
| Main-10 | CT Clamp | Heat Pump |
| Main-11 | CT Clamp | Water Heater |
| Main-12 | CT Clamp | Dryer |
| Main-13 | CT Clamp | Range |
| Main-14 | CT Clamp | Aux Heat/Blower |
| Main-97 | CT Clamp | Generator Inlet |
| Main-Balance | Calculated | Unmonitored circuits |

**Subpanel (if applicable):**
| Circuit | Sensor | Description |
|---------|--------|-------------|
| Subpanel | Built-in | Subpanel total |
| Subpanel-3 | CT Clamp | Master/Nursery |
| Subpanel-4 | CT Clamp | Garage Middle |
| Subpanel-6 | CT Clamp | Tesla Wall Connector |
| Subpanel-7 | CT Clamp | NEMA 14-50 Outlet |
| Subpanel-8 | CT Clamp | Dishwasher |
| Subpanel-Balance | Calculated | Unmonitored circuits |

---

## Software Setup

### Prerequisites
- Home Assistant OS installed (or supervised installation)
- Emporia Vue account with hardware configured
- Basic familiarity with Home Assistant add-ons

---

## Step 1: Install InfluxDB Add-on

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**
2. Search for "InfluxDB" and install the official add-on
3. Configure the add-on:

```yaml
# InfluxDB Add-on Configuration
databases:
  - emporia_vue_detailed
```

4. Start the add-on and open the Web UI
5. Create a database user:
   - Go to the InfluxDB admin panel
   - Create database: `emporia_vue_detailed`
   - Create user with write permissions

**Note the connection details:**
- Host: `a0d7b954-influxdb` (internal Docker hostname)
- Port: `8086`
- Database: `emporia_vue_detailed`

---

## Step 2: Install Vuegraf

Vuegraf is a Python application that pulls data from the Emporia Cloud API and writes it to InfluxDB.

### Option A: Run as Home Assistant Add-on (Recommended)

1. Add the Vuegraf repository to Home Assistant:
   - Go to **Settings → Add-ons → Add-on Store**
   - Click the three dots menu → **Repositories**
   - Add: `https://github.com/jertel/vuegraf`

2. Install the Vuegraf add-on

3. Configure via the add-on configuration panel

### Option B: Run as Standalone Container/Script

1. SSH into your Home Assistant host
2. Create configuration directory:
```bash
mkdir -p /config/vuegraf
```

3. Create configuration file `/config/vuegraf/vuegraf.json`:
```json
{
  "influxDb": {
    "version": 1,
    "host": "a0d7b954-influxdb",
    "port": 8086,
    "database": "emporia_vue_detailed",
    "user": "your_influxdb_user",
    "pass": "your_influxdb_password"
  },
  "updateIntervalSecs": 5,
  "detailedDataEnabled": true,
  "detailedIntervalSecs": 15,
  "lagSecs": 2,
  "accounts": [
    {
      "name": "your_account_name",
      "email": "your_emporia_email@example.com",
      "password": "your_emporia_password"
    }
  ]
}
```

### Configuration Parameters

| Parameter | Description | Recommended Value |
|-----------|-------------|-------------------|
| `updateIntervalSecs` | How often to poll Emporia API | 5 |
| `detailedDataEnabled` | Enable high-resolution data | true |
| `detailedIntervalSecs` | Resolution of detailed data | 15 |
| `lagSecs` | Delay to account for API lag | 2 |

### Verify Data Collection

Check that data is flowing into InfluxDB:
```bash
# SSH into Home Assistant
docker exec -it addon_a0d7b954_influxdb influx -database emporia_vue_detailed -execute "SHOW MEASUREMENTS"
```

Expected output:
```
name: measurements
name
----
energy_usage
```

Query sample data:
```bash
docker exec -it addon_a0d7b954_influxdb influx -database emporia_vue_detailed -execute "SELECT * FROM energy_usage LIMIT 5"
```

---

## Step 3: Install Grafana Add-on

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**
2. Search for "Grafana" and install the official add-on
3. Start the add-on and open the Web UI
4. Default login: Check add-on logs for initial credentials

---

## Step 4: Configure Grafana Data Source

1. In Grafana, go to **Connections → Data Sources → Add data source**
2. Select **InfluxDB**
3. Configure:

| Setting | Value |
|---------|-------|
| Name | InfluxDB |
| Query Language | InfluxQL |
| URL | `http://a0d7b954-influxdb:8086` |
| Database | `emporia_vue_detailed` |
| User | your_influxdb_user |
| Password | your_influxdb_password |
| HTTP Method | GET |

4. Click **Save & Test** - should show "datasource is working"

---

## Step 5: Import Dashboard

1. In Grafana, go to **Dashboards → Import**
2. Upload the `energy-monitor.json` file from this repository
3. Select your InfluxDB data source when prompted
4. Click **Import**

### Customize for Your Circuits

The dashboard queries use device names from Vuegraf. You'll need to update queries to match your circuit names.

Find your device names:
```bash
docker exec -it addon_a0d7b954_influxdb influx -database emporia_vue_detailed -execute "SHOW TAG VALUES FROM energy_usage WITH KEY = device_name"
```

Update panel queries to match your device names. Example query:
```sql
SELECT last("usage") FROM "energy_usage"
WHERE ("device_name" = 'Your Device Name Here') AND $timeFilter
```

---

## Dashboard Features

### Time Series Charts
- **Total Power (All Consumers)**: Stacked area chart showing all circuits
- **Heavy Loads**: Heat pump, water heater, aux heat
- **EV Charging**: Tesla charger and NEMA 14-50 outlet

### Power Gauges (Real-time Watts)
- Large gauge for total grid consumption
- Individual gauges for each monitored circuit
- Color thresholds indicate usage levels (blue → green → yellow → red)

### Energy Gauges (kWh for Time Period)
- Shows energy consumed during the selected time range
- Uses InfluxDB `INTEGRAL()` function to calculate:
```sql
SELECT INTEGRAL("usage") / 3600000 FROM "energy_usage"
WHERE ("device_name" = 'Circuit Name') AND $timeFilter
```
- Division by 3,600,000 converts watt-seconds to kilowatt-hours

### Tooltip Configuration
- Hover shows all values stacked
- Configured with: `"tooltip":{"mode":"all","sort":"desc"}`
- Dashboard-level setting: `"graphTooltip":1`

---

## InfluxQL Query Reference

### Current Power (Watts)
```sql
SELECT last("usage") FROM "energy_usage"
WHERE ("device_name" = 'Device Name') AND $timeFilter
```

### Average Power Over Time
```sql
SELECT mean("usage") FROM "energy_usage"
WHERE ("device_name" = 'Device Name') AND $timeFilter
GROUP BY time($__interval) fill(null)
```

### Energy Consumption (kWh)
```sql
SELECT INTEGRAL("usage") / 3600000 FROM "energy_usage"
WHERE ("device_name" = 'Device Name') AND $timeFilter
```

### List All Device Names
```sql
SHOW TAG VALUES FROM "energy_usage" WITH KEY = "device_name"
```

---

## Troubleshooting

### Vuegraf Not Collecting Data

**Symptom**: No new data in InfluxDB

1. Check Vuegraf logs:
```bash
docker logs addon_vuegraf 2>&1 | tail -50
```

2. Verify Emporia credentials are correct

3. Check for Python import errors - if you see:
```
ModuleNotFoundError: No module named 'vuegraf.collect'
```
This means a local file named `vuegraf.py` is shadowing the installed package. Rename or remove it:
```bash
cd /config/vuegraf
mv vuegraf.py vuegraf_backup.py
rm -rf __pycache__
```

4. Restart Vuegraf

### InfluxDB Connection Issues

**Symptom**: Grafana shows "No data"

1. Test InfluxDB connection:
```bash
curl -G 'http://localhost:8086/query' --data-urlencode "q=SHOW DATABASES"
```

2. Verify the internal hostname is correct:
   - From within Home Assistant containers: `a0d7b954-influxdb`
   - May vary based on add-on version

3. Check firewall/network settings

### Grafana Dashboard Not Loading

1. Verify data source is configured correctly
2. Check that queries match your actual device names
3. Ensure time range includes data (try "Last 1 hour")

### Data Gaps

**Symptom**: Missing data points in charts

1. Check if Vuegraf was running during the gap
2. Emporia API has rate limits - don't set `updateIntervalSecs` below 5
3. Historical data cannot be backfilled from Emporia API

---

## Advanced Configuration

### Retention Policies

For long-term storage efficiency, configure InfluxDB retention policies:

```sql
-- Keep detailed data for 30 days
CREATE RETENTION POLICY "30d" ON "emporia_vue_detailed" DURATION 30d REPLICATION 1 DEFAULT

-- Create downsampled data for long-term storage
CREATE RETENTION POLICY "1y" ON "emporia_vue_detailed" DURATION 365d REPLICATION 1

-- Continuous query to downsample
CREATE CONTINUOUS QUERY "cq_hourly" ON "emporia_vue_detailed"
BEGIN
  SELECT mean("usage") AS "usage"
  INTO "1y"."energy_usage_hourly"
  FROM "energy_usage"
  GROUP BY time(1h), "device_name"
END
```

### Alerting

Grafana can send alerts when power exceeds thresholds:

1. Edit a panel → Alert tab
2. Create alert rule (e.g., "Grid Total > 10000W for 5 minutes")
3. Configure notification channel (email, Slack, etc.)

### Home Assistant Integration

Expose Grafana data back to Home Assistant using the InfluxDB sensor:

```yaml
# configuration.yaml
sensor:
  - platform: influxdb
    host: a0d7b954-influxdb
    port: 8086
    database: emporia_vue_detailed
    username: your_user
    password: your_password
    queries:
      - name: "Total Power"
        measurement: energy_usage
        where: "device_name = 'Your Household - Main-TotalUsage'"
        field: usage
        unit_of_measurement: W
```

---

## File Structure

```
grafana-dashboards/
├── README.md                 # This documentation
├── energy-monitor.json       # Main Grafana dashboard
└── examples/
    └── vuegraf.json.example  # Example Vuegraf configuration
```

---

## Contributing

Feel free to submit issues or pull requests with improvements to the dashboard or documentation.

---

## Credits

- [Emporia Energy](https://www.emporiaenergy.com/) - Hardware manufacturer
- [Vuegraf](https://github.com/jertel/vuegraf) - Emporia API to InfluxDB bridge
- [Home Assistant](https://www.home-assistant.io/) - Home automation platform
- [Grafana](https://grafana.com/) - Visualization platform
- [InfluxDB](https://www.influxdata.com/) - Time-series database

---

## License

MIT License - Use freely for personal and commercial purposes.
