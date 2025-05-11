# OrbitalRescue-
Aerospace Decay Simulator with Collision Risk Estimator
from flask import Flask, request, render_template_string
import math
import json

app = Flask(__name__)

Re = 6371000
mu = 3.986004418e14

def get_density(alt_km):
    alt = alt_km
    if alt < 0:
        return 1.225
    if alt <= 100:
        log_rho = 0.088 - 0.0634 * alt
    elif alt <= 200:
        log_rho = -6.252 - 0.0245 * (alt - 100)
    elif alt <= 300:
        log_rho = -8.70 - 0.026 * (alt - 200)
    else:
        log_rho = -11.30 - 0.01 * (alt - 300)
    if alt > 200:
        log_rho -= 2
    return 10**log_rho

def simulate_lifetime(alt_km, mass, area, Cd, dt=60):
    a = (Re + alt_km * 1000.0)
    t = 0.0
    history = []  # track altitude vs time
    while True:
        alt_m = a - Re
        history.append((t/86400.0, alt_m/1000.0))
        if alt_m <= 0:
            break
        v = math.sqrt(mu / a)
        rho = get_density(alt_m / 1000.0)
        dE = -0.5 * rho * Cd * area * v**3 * dt
        da = (2 * a*a / mu) * dE
        a += da
        t += dt
        if t > 1e9 or a <= Re:
            break
    return t, history

def risk_estimate(alt_km, lifetime_years):
    base = 100 * (1 - math.exp(-lifetime_years/3.0))
    factor = math.exp(-alt_km/500.0)
    risk = base * factor
    risk_pct = min(99.9, max(0.1, round(risk, 1)))
    return risk_pct

@app.route('/', methods=['GET','POST'])
def index():
    mass = 500.0
    altitude = 400.0
    area = 10.0
    cd = 2.0

    lifetime_days = lifetime_years = None
    risk0 = risk1 = None
    lifetime2_days = lifetime2_years = None
    plot_data = []

    if request.method == 'POST':
        try:
            mass = float(request.form.get('mass', mass))
            altitude = float(request.form.get('altitude', altitude))
            area = float(request.form.get('area', area))
            cd = float(request.form.get('cd', cd))
        except:
            pass
        t0, history0 = simulate_lifetime(altitude, mass, area, cd, dt=60)
        lifetime_days = t0 / 86400.0
        lifetime_years = lifetime_days / 365.0
        risk0 = risk_estimate(altitude, lifetime_years)

        new_alt = altitude + 50.0
        t1, history1 = simulate_lifetime(new_alt, mass, area, cd, dt=60)
        lifetime2_days = t1 / 86400.0
        lifetime2_years = lifetime2_days / 365.0
        risk1 = risk_estimate(new_alt, lifetime2_years)

        plot_data = [{
            'x': [pt[0] for pt in history0],
            'y': [pt[1] for pt in history0],
            'mode': 'lines',
            'name': f'Initial @ {altitude} km'
        }, {
            'x': [pt[0] for pt in history1],
            'y': [pt[1] for pt in history1],
            'mode': 'lines',
            'name': f'+50km @ {new_alt} km'
        }]

    html = '''
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>OrbitalRescue | Aerospace Decay Simulator</title>
  <script src="https://cdn.plot.ly/plotly-latest.min.js"></script>
  <script src="https://cesium.com/downloads/cesiumjs/releases/1.108/Build/Cesium/Cesium.js"></script>
  <link href="https://cesium.com/downloads/cesiumjs/releases/1.108/Build/Cesium/Widgets/widgets.css" rel="stylesheet" />
  <style>
    body { background-color: #0b0c10; color: #ffffff; font-family: 'Segoe UI', sans-serif; padding: 2rem; margin: 0; }
    h1 { color: #66fcf1; font-size: 2.5rem; text-align: center; margin-bottom: 0.5rem; }
    p.tagline { color: #c5c6c7; text-align: center; margin-top: 0; margin-bottom: 2rem; }
    .container { display: flex; flex-wrap: wrap; max-width: 1200px; margin: auto; }
    form { background: #1f2833; padding: 1rem; border-radius: 10px; margin: 0.5rem; flex: 1 1 300px; }
    .results, .visual { background: #1f2833; padding: 1rem; border-radius: 10px; margin: 0.5rem; }
    .visual { flex: 2 1 600px; }
    label { display: block; margin-top: 1rem; font-weight: bold; }
    input { width: 100%; padding: 0.5rem; margin-top: 0.3rem; border: none; border-radius: 5px; }
    button { margin-top: 1rem; padding: 0.7rem 1.2rem; border: none; background-color: #45a29e; color: #fff; border-radius: 5px; cursor: pointer; }
    button:hover { background-color: #66fcf1; color: #000; }
    #cesiumContainer { width: 100%; height: 400px; margin-bottom: 1rem; border: 2px solid #45a29e; border-radius: 10px; }
    #plot { width: 100%; height: 400px; }
    footer { width: 100%; margin-top: 3rem; text-align: center; font-size: 0.9rem; color: #c5c6c7; }
  </style>
</head>
<body>
  <h1>🚀 OrbitalRescue</h1>
  <p class="tagline">Aerospace Decay Simulator with Collision Risk Estimator</p>
  <div class="container">
    <form method="POST">
      <label>Satellite Mass (kg):<input type="number" step="0.1" name="mass" value="{{mass}}" required></label>
      <label>Altitude (km):<input type="number" step="1" name="altitude" value="{{altitude}}" required></label>
      <label>Cross-sectional Area (m²):<input type="number" step="0.1" name="area" value="{{area}}" required></label>
      <label>Drag Coefficient (Cd):<input type="number" step="0.1" name="cd" value="{{cd}}" required></label>
      <button type="submit">Simulate</button>
    </form>
    {% if lifetime_days %}
    <div class="results">
      <h3>Results</h3>
      <p><strong>Initial Lifetime:</strong> {{lifetime_days|round(1)}} days (~{{lifetime_years|round(2)}} years)</p>
      <p><strong>Initial Collision Risk:</strong> {{risk0}}%</p>
      <p><strong>Post-Maneuver Lifetime (↑50km):</strong> {{lifetime2_days|round(1)}} days (~{{lifetime2_years|round(2)}} years)</p>
      <p><strong>Post-Maneuver Risk:</strong> {{risk1}}%</p>
    </div>
    <div class="visual">
      <div id="cesiumContainer"></div>
      <div id="plot"></div>
    </div>
    <script>
      Cesium.Ion.defaultAccessToken = 'YOUR_CESIUM_ION_TOKEN';
      const viewer = new Cesium.Viewer('cesiumContainer', {
        imageryProvider: new Cesium.IonImageryProvider({ assetId: 3 }),
        baseLayerPicker: false,
        shouldAnimate: true
      });
      const radius = 6371 + {{altitude}};
      const satellite = viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(0, 0, radius*1000),
        point: { pixelSize: 10, color: Cesium.Color.CYAN, outlineColor: Cesium.Color.WHITE, outlineWidth: 2 },
        label: { text: 'Your Satellite', fillColor: Cesium.Color.CYAN }
      });
      viewer.zoomTo(satellite);

      var data = {{ plot_data|tojson }};
      var layout = { title: 'Altitude vs. Time (days)', xaxis: { title: 'Time (days)' }, yaxis: { title: 'Altitude (km)' } };
      Plotly.newPlot('plot', data, layout);
    </script>
    {% endif %}
  </div>
  <footer>
    Build v1.0 • May 2025 • Developed by Ariana Cordero Tramontana • Contact: <a href="mailto:ariana.corderot@gmail.com" style="color:#66fcf1;">ariana.corderot@gmail.com</a>
  </footer>
</body>
</html>
'''
    return render_template_string(html,
                                 mass=mass, altitude=altitude,
                                 area=area, cd=cd,
                                 lifetime_days=lifetime_days,
                                 lifetime_years=lifetime_years,
                                 risk0=risk0,
                                 lifetime2_days=lifetime2_days,
                                 lifetime2_years=lifetime2_years,
                                 risk1=risk1,
                                 plot_data=plot_data)

if __name__ == '__main__':
    app.run(debug=True, port=5024)
