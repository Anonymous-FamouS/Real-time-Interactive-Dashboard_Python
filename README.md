# Real-time-Interactive-Dashboard_Python

https://github.com/user-attachments/assets/2ef813a4-6667-48d7-be34-e89d7d69f594

https://github.com/user-attachments/assets/901f4b72-1c4c-4234-a138-3b7cb7b7b1fc


```python
import panel as pn
import pandas as pd
import datetime
import time
import threading
import requests

from google.transit import gtfs_realtime_pb2

pn.extension("tabulator")

GTFS_RT_VEHICLE_URL = "https://cdn.mbta.com/realtime/VehiclePositions.pb"

# Function to fetch live GTFS-RT data
def fetch_gtfs_data():
    feed = gtfs_realtime_pb2.FeedMessage()
    response = requests.get(GTFS_RT_VEHICLE_URL)
    feed.ParseFromString(response.content)

    data = []
    now = datetime.datetime.now()

    for entity in feed.entity:
        if entity.HasField("vehicle"):
            v = entity.vehicle
            vehicle_id = v.vehicle.id
            route_id = v.trip.route_id if v.trip else None
            lat = v.position.latitude
            lon = v.position.longitude
            speed = v.position.speed if v.position.HasField("speed") else None

            data.append({
                "vehicle_id": vehicle_id,
                "route": route_id,
                "lat": lat,
                "lon": lon,
                "speed": speed,
                "timestamp": now
            })

    return pd.DataFrame(data)

# Initial data
data_df = fetch_gtfs_data()
data_pane = pn.widgets.Tabulator(data_df, height=400, width=1000)

# Background thread to update data
def update_loop():
    while True:
        time.sleep(10)
        try:
            new_df = fetch_gtfs_data()
            data_pane.value = new_df
        except Exception as e:
            print("Error fetching GTFS-RT data:", e)

thread = threading.Thread(target=update_loop, daemon=True)
thread.start()

# Dashboard layout
dashboard = pn.Column(
    "# 🚍 MBTA Live Vehicle Positions (GTFS-RT)",
    "Updates every 10 seconds",
    data_pane
)

dashboard.servable()


```


### Second Snippet (Table and Pydeck Map Dashboard)


```python

import panel as pn
import pandas as pd
import datetime
import time
import threading
import requests
import pydeck as pdk

from google.transit import gtfs_realtime_pb2

pn.extension("tabulator", "deckgl")

GTFS_RT_VEHICLE_URL = "https://cdn.mbta.com/realtime/VehiclePositions.pb"

# Function to fetch GTFS-RT data
def fetch_gtfs_data():
    feed = gtfs_realtime_pb2.FeedMessage()
    response = requests.get(GTFS_RT_VEHICLE_URL)
    feed.ParseFromString(response.content)

    data = []
    now = datetime.datetime.now()

    for entity in feed.entity:
        if entity.HasField("vehicle"):
            v = entity.vehicle
            vehicle_id = v.vehicle.id
            route_id = v.trip.route_id if v.trip else None
            lat = v.position.latitude
            lon = v.position.longitude
            speed = v.position.speed if v.position.HasField("speed") else None

            data.append({
                "vehicle_id": vehicle_id,
                "route": route_id,
                "lat": lat,
                "lon": lon,
                "speed": speed,
                "timestamp": now
            })

    return pd.DataFrame(data)

# Initial data
data_df = fetch_gtfs_data()

# Table component
data_table = pn.widgets.Tabulator(data_df, height=400, width=1000)

# Initial pydeck map setup
initial_view_state = pdk.ViewState(
    latitude=data_df['lat'].mean(),
    longitude=data_df['lon'].mean(),
    zoom=11,
    pitch=0,
)

scatter_layer = pdk.Layer(
    "ScatterplotLayer",
    data=data_df,
    get_position='[lon, lat]',
    get_fill_color='[0, 0, 255, 160]',
    get_radius=50,
    pickable=True,
)

deck_map = pn.pane.DeckGL(
    pdk.Deck(
        layers=[scatter_layer],
        initial_view_state=initial_view_state,
        map_provider="carto",            # ✅ no Mapbox needed
        map_style="light"                # carto's light theme
    ),
    height=500,
    sizing_mode="stretch_width"
)

# Background thread to update data and map
def update_loop():
    while True:
        time.sleep(10)
        try:
            new_df = fetch_gtfs_data()
            data_table.value = new_df

            new_layer = pdk.Layer(
                "ScatterplotLayer",
                data=new_df,
                get_position='[lon, lat]',
                get_fill_color='[0, 0, 255, 160]',
                get_radius=50,
                pickable=True,
            )

            new_view = pdk.ViewState(
                latitude=new_df["lat"].mean(),
                longitude=new_df["lon"].mean(),
                zoom=11,
                pitch=0,
            )

            deck_map.object = pdk.Deck(
                layers=[new_layer],
                initial_view_state=new_view,
                map_provider="carto",
                map_style="light"
            )
        except Exception as e:
            print("Update error:", e)

threading.Thread(target=update_loop, daemon=True).start()

# Final dashboard layout
dashboard = pn.Column(
    "# 🚍 MBTA Live Vehicle Positions (GTFS-RT)",
    "Auto-updates every 10 seconds. No Mapbox token needed.",
    deck_map,
    data_table
)

dashboard.servable()

```

#### Adding save button

```python
import panel as pn
import pandas as pd
import datetime
import time
import threading
import requests
import pydeck as pdk
import io
from google.transit import gtfs_realtime_pb2

pn.extension("tabulator", "deckgl")

GTFS_RT_VEHICLE_URL = "https://cdn.mbta.com/realtime/VehiclePositions.pb"

# Shared DataFrame (initialized empty)
data_df = pd.DataFrame()

# Function to fetch GTFS-RT data
def fetch_gtfs_data():
    feed = gtfs_realtime_pb2.FeedMessage()
    response = requests.get(GTFS_RT_VEHICLE_URL)
    feed.ParseFromString(response.content)

    data = []
    now = datetime.datetime.now()

    for entity in feed.entity:
        if entity.HasField("vehicle"):
            v = entity.vehicle
            vehicle_id = v.vehicle.id
            route_id = v.trip.route_id if v.trip else None
            lat = v.position.latitude
            lon = v.position.longitude
            speed = v.position.speed if v.position.HasField("speed") else None

            data.append({
                "vehicle_id": vehicle_id,
                "route": route_id,
                "lat": lat,
                "lon": lon,
                "speed": speed,
                "timestamp": now
            })

    return pd.DataFrame(data)

# Initial data load
data_df = fetch_gtfs_data()

# Table component (Tabulator)
data_table = pn.widgets.Tabulator(data_df, height=400, width=1000)

# Initial pydeck map setup
initial_view_state = pdk.ViewState(
    latitude=data_df['lat'].mean() if not data_df.empty else 42.3601,
    longitude=data_df['lon'].mean() if not data_df.empty else -71.0589,
    zoom=11,
    pitch=0,
)

scatter_layer = pdk.Layer(
    "ScatterplotLayer",
    data=data_df,
    get_position='[lon, lat]',
    get_fill_color='[0, 0, 255, 160]',
    get_radius=50,
    pickable=True,
)

deck_map = pn.pane.DeckGL(
    pdk.Deck(
        layers=[scatter_layer],
        initial_view_state=initial_view_state,
        map_provider="carto",
        map_style="light"
    ),
    height=500,
    sizing_mode="stretch_width"
)

# Correct FileDownload callback: returns bytes, not string
def get_csv_bytes():
    buffer = io.StringIO()
    data_table.value.to_csv(buffer, index=False)
    buffer.seek(0)
    return io.BytesIO(buffer.getvalue().encode('utf-8'))


download_button = pn.widgets.FileDownload(
    label="💾 Export CSV",
    filename="vehicle_positions.csv",
    callback=get_csv_bytes,
    button_type="primary"
)


# Background thread for updating data and map every 10 seconds
def update_loop():
    global data_df
    while True:
        time.sleep(10)
        try:
            new_df = fetch_gtfs_data()
            data_df = new_df
            # Update the table data source
            data_table.value = new_df

            # Update the pydeck map layers and view
            new_layer = pdk.Layer(
                "ScatterplotLayer",
                data=new_df,
                get_position='[lon, lat]',
                get_fill_color='[0, 0, 255, 160]',
                get_radius=50,
                pickable=True,
            )

            new_view = pdk.ViewState(
                latitude=new_df["lat"].mean() if not new_df.empty else 42.3601,
                longitude=new_df["lon"].mean() if not new_df.empty else -71.0589,
                zoom=11,
                pitch=0,
            )

            deck_map.object = pdk.Deck(
                layers=[new_layer],
                initial_view_state=new_view,
                map_provider="carto",
                map_style="light"
            )
        except Exception as e:
            print("Update error:", e)

threading.Thread(target=update_loop, daemon=True).start()

# Final layout
dashboard = pn.Column(
    "# 🚍 MBTA Live Vehicle Positions (GTFS-RT)",
    "Auto-updates every 10 seconds. No Mapbox token needed.",
    deck_map,
    download_button,
    data_table,
)

dashboard.servable()

```

https://github.com/user-attachments/assets/9e115b17-9188-47b4-a187-5b1d4c5cd215


### With search and filter

```python
import panel as pn
import pandas as pd
import datetime
import time
import threading
import requests
import pydeck as pdk
import io
from google.transit import gtfs_realtime_pb2

pn.extension("tabulator", "deckgl")

GTFS_RT_VEHICLE_URL = "https://cdn.mbta.com/realtime/VehiclePositions.pb"

# Shared DataFrame
data_df = pd.DataFrame()

# Function to fetch GTFS-RT data
def fetch_gtfs_data():
    feed = gtfs_realtime_pb2.FeedMessage()
    response = requests.get(GTFS_RT_VEHICLE_URL)
    feed.ParseFromString(response.content)

    data = []
    now = datetime.datetime.now()

    for entity in feed.entity:
        if entity.HasField("vehicle"):
            v = entity.vehicle
            vehicle_id = v.vehicle.id
            route_id = v.trip.route_id if v.trip else None
            lat = v.position.latitude
            lon = v.position.longitude
            speed = v.position.speed if v.position.HasField("speed") else None

            data.append({
                "vehicle_id": vehicle_id,
                "route": route_id,
                "lat": lat,
                "lon": lon,
                "speed": speed,
                "timestamp": now
            })

    return pd.DataFrame(data)

# Initial data load
data_df = fetch_gtfs_data()

# Widgets for filtering
search_input = pn.widgets.TextInput(name='Search Vehicle ID or Route', placeholder='Type to filter...')
route_options = ['All'] + sorted(data_df['route'].dropna().unique().tolist())
route_filter = pn.widgets.Select(name='Filter by Route', options=route_options, value='All')

# Table component
data_table = pn.widgets.Tabulator(data_df, height=400, width=1000)

# Initial pydeck map setup
initial_view_state = pdk.ViewState(
    latitude=data_df['lat'].mean() if not data_df.empty else 42.3601,
    longitude=data_df['lon'].mean() if not data_df.empty else -71.0589,
    zoom=11,
    pitch=0,
)

scatter_layer = pdk.Layer(
    "ScatterplotLayer",
    data=data_df,
    get_position='[lon, lat]',
    get_fill_color='[0, 0, 255, 160]',
    get_radius=50,
    pickable=True,
)

deck_map = pn.pane.DeckGL(
    pdk.Deck(
        layers=[scatter_layer],
        initial_view_state=initial_view_state,
        map_provider="carto",
        map_style="light"
    ),
    height=500,
    sizing_mode="stretch_width"
)

# CSV download button
def get_csv_bytes():
    filtered_df = filter_data()
    buffer = io.StringIO()
    filtered_df.to_csv(buffer, index=False)
    return buffer.getvalue().encode()

download_button = pn.widgets.FileDownload(
    label="💾 Export CSV",
    filename="vehicle_positions.csv",
    callback=get_csv_bytes,
    button_type="primary"
)

# Filtering function using current widget values
def filter_data():
    df = data_df
    search_val = search_input.value.strip().lower()
    selected_route = route_filter.value

    if selected_route != 'All':
        df = df[df['route'] == selected_route]
    if search_val:
        df = df[
            df['vehicle_id'].str.lower().str.contains(search_val) |
            df['route'].str.lower().str.contains(search_val)
        ]
    return df

# Function to update table and map from filters
@pn.depends(search_input.param.value, route_filter.param.value)
def update_view(search, route):
    filtered_df = filter_data()
    data_table.value = filtered_df

    new_layer = pdk.Layer(
        "ScatterplotLayer",
        data=filtered_df,
        get_position='[lon, lat]',
        get_fill_color='[255, 0, 0, 160]',
        get_radius=50,
        pickable=True,
    )

    new_view = pdk.ViewState(
        latitude=filtered_df["lat"].mean() if not filtered_df.empty else 42.3601,
        longitude=filtered_df["lon"].mean() if not filtered_df.empty else -71.0589,
        zoom=11,
        pitch=0,
    )

    deck_map.object = pdk.Deck(
        layers=[new_layer],
        initial_view_state=new_view,
        map_provider="carto",
        map_style="light"
    )

# Background thread to fetch fresh data every 10 sec
def update_loop():
    global data_df, route_filter
    while True:
        time.sleep(10)
        try:
            new_df = fetch_gtfs_data()
            data_df = new_df
            # Update route options dynamically
            routes = ['All'] + sorted(new_df['route'].dropna().unique().tolist())
            route_filter.options = routes

            # Update view based on current filters
            pn.io.push_notebook()  # Ensure updates push through (optional)

            # Call update_view manually with current widget values to refresh
            update_view(search_input.value, route_filter.value)

        except Exception as e:
            print("Update error:", e)

threading.Thread(target=update_loop, daemon=True).start()

# Final dashboard layout
dashboard = pn.Column(
    "# 🚍 MBTA Live Vehicle Positions (GTFS-RT)",
    "Auto-updates every 10 seconds. No Mapbox token needed.",
    pn.Row(search_input, route_filter),
    deck_map,
    download_button,
    data_table
)

dashboard.servable()

```

https://github.com/user-attachments/assets/aeeb8b2c-3ad8-44a4-b8dc-e0e540df57cc


### THe final version


https://github.com/user-attachments/assets/2ef813a4-6667-48d7-be34-e89d7d69f594




