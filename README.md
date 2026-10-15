# 🚍 Jihlava MHD (DPMJ) — Unofficial Live Transit API Documentation

> **Unofficial documentation** for the real-time public transit (MHD) tracking API operated by **Statutární město Jihlava** and **Dopravní podnik města Jihlavy (DPMJ)**.

![Status](https://img.shields.io/badge/API_Status-Active-brightgreen)
![Protocol](https://img.shields.io/badge/Backend-ArcGIS_REST_10.91-blue)
![Format](https://img.shields.io/badge/Format-JSON_%7C_GeoJSON_%7C_PBF-orange)
![Auth](https://img.shields.io/badge/Auth-None_(Public)-success)

---

## 📖 Overview & Hardware Architecture

The official Jihlava live transit map ([sgis.jihlava-city.cz/web/mhd/](https://sgis.jihlava-city.cz/web/mhd/)) is powered by an **Esri ArcGIS Enterprise** backend (`v10.91`). 

### How the Telemetry Reaches the API
1. **Onboard Vehicle Computer (*Palubní počítač*):** Tracks GPS position, current stop, timetable deviation (delay), and route/course IDs.
2. **Radio Telemetry Modem (RASPRO / MotoTRBO):** Buses and trolleybuses transmit their telemetry over DPMJ's digital UHF/VHF radio network using **Motorola MotoTRBO** radios interfaced via **RASPRO TRBOdata** hardware modules.
3. **Dispatch & Ingestion (`dopravaapi.jihlava-city.cz`):** Central dispatch receives the radio packets and forwards the vehicle states to the city's transit API ingestion server.
4. **ArcGIS REST MapServer:** The data is exposed publicly via the `Ji_MHD_aktualni` MapServer, which the web frontend polls every **12 seconds** (`refreshInterval: 0.2` minutes).

---

## 🔗 Base Endpoints

* **Live Map Frontend:** `https://sgis.jihlava-city.cz/web/mhd/`
* **ArcGIS Portal WebMap Item:** `https://gis.jihlava-city.cz/portal/sharing/rest/content/items/5d1ff2038a0343a7baece3a91064072a/data?f=pjson`
* **Primary MapServer Base URL:**
  ```text
  https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer
  ```

---

## 🗂️ Available Layers & Tables

The `Ji_MHD_aktualni` MapServer is divided into several numbered sub-layers. Some layers are hidden on the official web map (`defaultVisibility: false`) but remain publicly queryable via the REST API.

| Layer ID | Name | Type | Default Visibility | Description |
| :--- | :--- | :--- | :---: | :--- |
| **`47`** | `poloha vozidel MHD (aktuální)` | Point | ✅ Yes | **Active moving vehicles** currently on a route. |
| **`50`** | `poloha vozidel MHD (DPMJ)` | Point | ❌ No | **Hidden DPMJ layer** for moving vehicles (mirrors Layer 47). |
| **`2`** | `poloha vozidel MHD (čekající)` | Point | ✅ Yes | **Waiting vehicles** parked at terminal stops waiting for departure. |
| **`51`** | `čekající vozidla MHD (DPMJ)` | Point | ❌ No | **Hidden DPMJ layer** for waiting vehicles (mirrors Layer 2). |
| **`3`** | `zastávky` | Point | ✅ Yes | **All MHD transit stops** with stop IDs and coordinates. |
| **`1`** | `uzavírka, událost (bod)` | Point | ✅ Yes | **Road closures & service alerts** (point locations + alert text). |
| **`48`** | `uzavírka, událost (linie)` | Polyline | ✅ Yes | **Road closures & detours** (street polylines). |
| **`4`** | `trasy MHD` | Polyline | ✅ Yes | **Transit route polylines** (overview network). |
| **`5`** | `jednotlivé trasy MHD` | Polyline | ❌ No | **Individual transit line polylines**. |
| **`46`** | `poloha vánočního vozu` | Point | ❌ No | Special thematic tracker: Moving Christmas vehicle. |
| **`45`** | `poloha čekajícího vánočního vozu`| Point | ❌ No | Special thematic tracker: Waiting Christmas vehicle. |
| **`49`** | `poloha vozů s výkresy ZŠ` | Point | ❌ No | Special thematic tracker: Buses with elementary school drawings. |
| **`52`** | `DO Jihlavy` | Point | ✅ Yes | Special thematic tracker: "DO Jihlavy" promotional vehicle. |
| **`1017`**| `seznam linek MHD` | Table | — | Master table of MHD lines. |

---

## 🚀 Ready-to-Use Query URLs

### 1. Get All Moving Vehicles (Clean JSON)
Since `latitude` and `longitude` are already included as attributes in Layer `47`, you can set `returnGeometry=false` to strip out the redundant Esri geometry wrapper:
```http
GET https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/47/query?where=1%3D1&outFields=*&returnGeometry=false&f=json
```

### 2. Get All Waiting Vehicles at Terminals (Clean JSON)
```http
GET https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/2/query?where=1%3D1&outFields=*&returnGeometry=false&f=json
```

### 3. Get All Moving Vehicles as Standard GeoJSON
Ideal for feeding directly into **Leaflet**, **Mapbox GL**, **OpenLayers**, or **QGIS**. Adding `outSR=4326` instructs ArcGIS to reproject the geometry from the Czech S-JTSK grid (`WKID 5514`) into standard WGS84 GPS coordinates:
```http
GET https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/47/query?where=1%3D1&outFields=*&outSR=4326&f=geojson
```

### 4. Get Active Road Closures & Detour Alerts
```http
GET https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/1/query?where=1%3D1&outFields=*&outSR=4326&f=json
```

---

## 🔍 Advanced Filtering (`where` Parameter)

Because this is an ArcGIS REST server, the `where` parameter accepts standard **SQL `WHERE` clauses** (URL-encoded). Instead of downloading all buses and filtering in your code, you can ask the server to filter them for you:

| Filter Goal | Raw SQL `where` Clause | URL-Encoded Parameter |
| :--- | :--- | :--- |
| **All records (default)** | `1=1` | `where=1%3D1` |
| **Only Line B** | `linka='B'` | `where=linka%3D%27B%27` |
| **Only Trolleybuses** | `typ='trolejbus'` | `where=typ%3D%27trolejbus%27` |
| **Only Buses** | `typ='autobus'` | `where=typ%3D%27autobus%27` |
| **Vehicles delayed > 3 mins** | `delayinmins>3` | `where=delayinmins%3E3` |
| **Specific Fleet Number (#12)**| `carnum=12` | `where=carnum%3D12` |

---

## 📊 Data Schema Reference

### Vehicle Attributes (Layers `47`, `50`, `2`, `51`)

 note: The API exposes extra internal telemetry (`carnum`, `speed`, `azimut`, `routeid`) that is not shown to users on the public web map popups.

| Field Name | Data Type | Example | Description |
| :--- | :--- | :--- | :--- |
| `objectid` | `Integer (OID)` | `2` | Internal ArcGIS feature object ID. |
| `linka` | `String(10)` | `"E"`, `"31"` | Public line designation (letters for trolleybuses, numbers for buses). |
| `typ` | `String(10)` | `"trolejbus"` | Vehicle type (`"trolejbus"` or `"autobus"`). |
| `carnum` | `SmallInteger` | `12` | **DPMJ Fleet Number (*evidenční číslo vozu*)** — unique to the physical vehicle. |
| `delayinmins` | `SmallInteger` | `1` | Current schedule delay in minutes (`0` = on time, negative = ahead of schedule). |
| `posledni` | `String(255)` | `"Motorpal hala"` | Name of the last transit stop the vehicle passed. |
| `konecna` | `String(255)` | `"Chlumova"` | Name of the vehicle's final destination stop for the current run. |
| `latitude` | `Double` | `49.41507` | Live WGS84 GPS Latitude. |
| `longitude` | `Double` | `15.5623894` | Live WGS84 GPS Longitude. |
| `speed` | `SmallInteger` | `18` | Current vehicle speed in **km/h**. |
| `azimut` | `SmallInteger` | `157` | Compass heading in degrees (`0` = North, `90` = East, `180` = South, `270` = West). |
| `lineid` | `SmallInteger` | `205` | Internal CIS line identifier (e.g., `205` = Line E, `131` = Line 31). |
| `routeid` | `SmallInteger` | `4` | Duty / course number (*kurz*). |
| `laststopid` | `SmallInteger` | `42` | Internal ID of the last stop passed (`posledni`). |
| `lastpostid` | `SmallInteger` | `2` | Platform / post number (*označník*) at the last stop. |
| `finalstopid` | `SmallInteger` | `8` | Internal ID of the final destination stop (`konecna`). |
| `state` | `SmallInteger` | `0` | Vehicle operational status code. |

#### Example JSON Payload (Single Vehicle Feature)
```json
{
  "attributes": {
    "objectid": 2,
    "carnum": 12,
    "delayinmins": 0,
    "finalstopid": 8,
    "laststopid": 42,
    "lastpostid": 2,
    "latitude": 49.41507,
    "lineid": 205,
    "longitude": 15.5623894,
    "routeid": 4,
    "state": 0,
    "linka": "E",
    "konecna": "Chlumova",
    "posledni": "Motorpal hala",
    "typ": "trolejbus",
    "azimut": 157,
    "speed": 13
  }
}
```

---

### Road Closures & Alerts Attributes (Layer `1`)

| Field Name | Data Type | Example | Description |
| :--- | :--- | :--- | :--- |
| `objectid` | `Integer (OID)` | `48` | Internal object ID. |
| `typ` | `String(255)` | `"uzavírka"` | Alert category. |
| `popis` | `String(500)` | `"UZAVÍRKA 17. LISTOPADU"` | Short headline of the closure or event. |
| `poznamka` | `String(1000)` | `"Vážení cestující..."` | Full passenger advisory describing detours and skipped stops. |
| `datum_od` | `Date (Epoch ms)`| `1776204000000` | Start timestamp in **Unix epoch milliseconds** (UTC). |
| `datum_do` | `Date (Epoch ms)`| `1793401200000` | End timestamp in **Unix epoch milliseconds** (UTC). |
| `stav_prvku` | `String(255)` | `"Stávající"` | Current status of the alert (e.g., `"Stávající"` = Active). |

---

## 💻 Code Examples

### 1. Bash (`curl` + `jq`)
Print a live formatted table of all moving vehicles in the terminal:
```bash
curl -s "https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/47/query?where=1%3D1&outFields=*&returnGeometry=false&f=json" | jq -r '.features[].attributes | "\(.linka)	#\(.carnum) (\(.typ))	Delay: \(.delayinmins)m	Speed: \(.speed)km/h	\(.posledni) -> \(.konecna)"'
```

### 2. Python (`requests`)
Fetch both moving and waiting vehicles with clean browser headers:
```python
import requests

BASE_URL = "https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer"
HEADERS = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) JihlavaMHDClient/1.0",
    "Referer": "https://sgis.jihlava-city.cz/web/mhd/"
}

def get_vehicles(layer_id=47, line=None):
    where_clause = f"linka='{line}'" if line else "1=1"
    params = {
        "where": where_clause,
        "outFields": "*",
        "returnGeometry": "false",
        "f": "json"
    }
    response = requests.get(f"{BASE_URL}/{layer_id}/query", params=params, headers=HEADERS, timeout=10)
    response.raise_for_status()
    data = response.json()
    return [feature["attributes"] for feature in data.get("features", [])]

if __name__ == "__main__":
    moving_vehicles = get_vehicles(layer_id=47)
    for v in moving_vehicles:
        print(f"[{v['linka']}] Vehicle #{v['carnum']} ({v['typ']}) | "
              f"{v['posledni']} -> {v['konecna']} | "
              f"Delay: {v['delayinmins']} min | Speed: {v['speed']} km/h | "
              f"GPS: {v['latitude']}, {v['longitude']}")
```

### 3. JavaScript / Node.js (`fetch`)
```javascript
const url = "https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/47/query?where=1%3D1&outFields=*&returnGeometry=false&f=json";

async function fetchJihlavaBuses() {
  const res = await fetch(url, {
    headers: { "Referer": "https://sgis.jihlava-city.cz/web/mhd/" }
  });
  const data = await res.json();
  const buses = data.features.map(f => f.attributes);
  console.table(buses, ["linka", "carnum", "typ", "delayinmins", "speed", "posledni", "konecna"]);
}

fetchJihlavaBuses();
```

### 4. Home Assistant (`configuration.yaml`)
Track the count of active vehicles on a specific line (e.g., Line `E`) and expose vehicle details as attributes:
```yaml
rest:
  - resource: "https://gis.jihlava-city.cz/server1/rest/services/verejnost/Ji_MHD_aktualni/MapServer/47/query?where=linka%3D'E'&outFields=*&returnGeometry=false&f=json"
    scan_interval: 15
    headers:
      Referer: "https://sgis.jihlava-city.cz/web/mhd/"
    sensor:
      - name: "MHD Jihlava Linka E"
        value_template: "{{ value_json.features | length }}"
        unit_of_measurement: "vozy"
        json_attributes_path: "$.features[0].attributes"
        json_attributes:
          - carnum
          - delayinmins
          - posledni
          - konecna
          - speed
          - latitude
          - longitude
```

---

## ⚠️ Best Practices & Rate Limits

1. **Respect the Polling Interval:** The official web map refreshes every **12 seconds** (`refreshInterval: 0.2` min). Do not poll the server faster than once every **12–15 seconds**, or the city's firewall may rate-limit or block your IP.
2. **Include a `Referer` Header:** While direct requests currently work without authentication, the ArcGIS portal config explicitly lists `authorizedCrossOriginDomains: ["https://sgis.jihlava-city.cz", "tmapserver.cz", "https://dopravaapi.jihlava-city.cz/", "github.io"]`. If making browser-side XHR requests or automated backend calls, include `Referer: https://sgis.jihlava-city.cz/web/mhd/` (or host your frontend on GitHub Pages `github.io`, which is whitelisted for CORS!).
3. **Avoid `resultRecordCount` in Production:** When testing, adding `resultRecordCount=2` limits the response and triggers `"exceededTransferLimit": true`. Omit `resultRecordCount` in production to receive all active vehicles (the server allows up to `maxRecordCount: 2000` per request).

---

## ⚖️ Disclaimer

This documentation was created via clean-room inspection of public network requests for educational and interoperability purposes. All transit data is the property of **Statutární město Jihlava** (`gis@jihlava-city.cz`) and **Dopravní podnik města Jihlavy, a.s. (DPMJ)**. Because this is an undocumented internal endpoint, the schema or URLs may change without notice.
