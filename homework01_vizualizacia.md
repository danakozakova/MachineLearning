# IB031 – Domáci úkol 1: Interaktívna vizualizácia

## Použité knižnice

| Knižnica | Účel |
|---|---|
| `pandas` | Načítanie a manipulácia s dátami |
| `numpy` | Zaokrúhľovanie hodnôt v heatmape |
| `plotly` | Interaktívne grafy (`graph_objects`, `FigureWidget`) |
| `ipywidgets` | Dropdown widget pre filtrovanie |
| `IPython.display` | Funkcia `display()` na zobrazenie widgetov |

### Inštalácia (ak treba)

```bash
pip install pandas numpy plotly ipywidgets openpyxl
```

> [!NOTE]
> `openpyxl` je potrebný na čítanie `.xlsx` súborov cez pandas.

---

## Štruktúra notebooku

### Bunka 1 – Načítanie dát

```python
import pandas as pd

df = pd.read_excel("australia_weather.xlsx", skiprows=10, header=None)

# Ponecháme len stĺpce 2 až 25 (reálne dáta)
df = df.iloc[:, 2:26]

# Priradíme názvy stĺpcov
df.columns = [
    'Date', 'Location',
    'MinTemp', 'MaxTemp',
    'Rainfall', 'Evaporation', 'Sunshine',
    'WindGustDir', 'WindGustSpeed',
    'WindDir9am', 'WindDir3pm',
    'WindSpeed9am', 'WindSpeed3pm',
    'Humidity9am', 'Humidity3pm',
    'Pressure9am', 'Pressure3pm',
    'Cloud9am', 'Cloud3pm',
    'Temp9am', 'Temp3pm',
    'RainToday', 'RISK_MM', 'RainTomorrow'
]

# Zahodíme prázdne riadky
df = df.dropna(how='all')

print(f"Rozmer: {df.shape}")
df.head()
```

---

### Bunka 2 – Príprava dát pre koreláciu

```python
import numpy as np

numeric_cols = df.select_dtypes(include='number').columns.tolist()
corr = df[numeric_cols].corr()
```

---

### Bunka 3 – Heatmapa (FigureWidget)

```python
import plotly.graph_objects as go

heatmap_fig = go.FigureWidget(data=go.Heatmap(
    z=corr.values,
    x=corr.columns,
    y=corr.columns,
    colorscale='RdBu_r',
    zmin=-1, zmax=1,
    text=np.round(corr.values, 2),
    texttemplate='%{text}',
    textfont={"size": 9},
    hovertemplate='%{x} vs %{y}<br>Korelácia: %{z:.3f}<extra></extra>'
))

heatmap_fig.update_layout(
    title='Klikni na bunku → zobrazí sa scatter plot',
    width=750, height=650,
    xaxis=dict(tickangle=45),
);
```

---

### Bunka 4 – Scatter plot (FigureWidget)

```python
init_x, init_y = numeric_cols[0], numeric_cols[1]

scatter_fig = go.FigureWidget()

for label, color in [('No rain', 'steelblue'), ('Rain', 'tomato')]:
    mask = df['RainTomorrow'] == ('No' if label == 'No rain' else 'Yes')
    scatter_fig.add_trace(go.Scattergl(
        x=df.loc[mask, init_x],
        y=df.loc[mask, init_y],
        mode='markers',
        name=label,
        marker=dict(size=4, opacity=0.4, color=color),
        hovertemplate='%{x:.1f}, %{y:.1f}<extra></extra>'
    ))

scatter_fig.update_layout(
    title=f'{init_x} vs {init_y} (r = {corr.loc[init_x, init_y]:.3f})',
    xaxis_title=init_x,
    yaxis_title=init_y,
    width=750, height=500,
    legend=dict(title='RainTomorrow'),
);
```

---

### Bunka 5 – Dropdown, callbacky a zobrazenie

```python
import ipywidgets as widgets
from IPython.display import display

# Dropdown pre výber lokality
locations = ['All'] + sorted(df['Location'].unique().tolist())

dropdown = widgets.Dropdown(
    options=locations,
    value='All',
    description='Lokalita:',
    style={'description_width': 'initial'},
    layout=widgets.Layout(width='300px')
)

# Slovník na ukladanie aktuálne vybraných stĺpcov
current = {'col_x': numeric_cols[0], 'col_y': numeric_cols[1]}

def update_all(location):
    """Aktualizuje oba grafy podľa vybranej lokality."""
    if location == 'All':
        df_filtered = df
    else:
        df_filtered = df[df['Location'] == location]
    
    new_corr = df_filtered[numeric_cols].corr()
    
    # Aktualizujeme heatmapu
    with heatmap_fig.batch_update():
        heatmap_fig.data[0].z = new_corr.values
        heatmap_fig.data[0].text = np.round(new_corr.values, 2)
        heatmap_fig.layout.title = f'Korelácie – {location} (n = {len(df_filtered):,})'
    
    # Aktualizujeme scatter plot
    col_x, col_y = current['col_x'], current['col_y']
    r = new_corr.loc[col_x, col_y]
    
    with scatter_fig.batch_update():
        for i, rain_val in enumerate(['No', 'Yes']):
            mask = df_filtered['RainTomorrow'] == rain_val
            scatter_fig.data[i].x = df_filtered.loc[mask, col_x]
            scatter_fig.data[i].y = df_filtered.loc[mask, col_y]
        scatter_fig.layout.title = f'{col_x} vs {col_y} (r = {r:.3f}) – {location}'


def on_location_change(change):
    update_all(change['new'])

dropdown.observe(on_location_change, names='value')


def on_heatmap_click(trace, points, selector):
    """Kliknutie na heatmapu aktualizuje scatter plot."""
    if points.point_inds:
        col_x = points.xs[0] if isinstance(points.xs[0], str) else corr.columns[points.xs[0]]
        col_y = points.ys[0] if isinstance(points.ys[0], str) else corr.columns[points.ys[0]]
        current['col_x'] = col_x
        current['col_y'] = col_y
        
        location = dropdown.value
        if location == 'All':
            df_filtered = df
        else:
            df_filtered = df[df['Location'] == location]
        
        new_corr = df_filtered[numeric_cols].corr()
        r = new_corr.loc[col_x, col_y]
        
        with scatter_fig.batch_update():
            for i, rain_val in enumerate(['No', 'Yes']):
                mask = df_filtered['RainTomorrow'] == rain_val
                scatter_fig.data[i].x = df_filtered.loc[mask, col_x]
                scatter_fig.data[i].y = df_filtered.loc[mask, col_y]
            scatter_fig.layout.xaxis.title = col_x
            scatter_fig.layout.yaxis.title = col_y
            scatter_fig.layout.title = f'{col_x} vs {col_y} (r = {r:.3f}) – {location}'

heatmap_fig.data[0].on_click(on_heatmap_click)

# Zobrazenie
display(dropdown)
display(heatmap_fig)
display(scatter_fig)
```

---

## Interakcie

| Akcia | Výsledok |
|---|---|
| **Klik na bunku heatmapy** | Scatter plot sa aktualizuje na vybraný pár premenných |
| **Zmena lokality v dropdown** | Prepočíta korelácie v heatmape + prefiltruje scatter plot |
| **Zoom** (koliesko myši / ťahanie) | Priblíženie na oboch grafoch |
| **Hover** | Tooltip s hodnotami na oboch grafoch |
| **Legenda** (klik na "No rain" / "Rain") | Skrytie/zobrazenie skupiny bodov v scatter plote |

## Splnené požiadavky zadania

- ✅ Dva rôzne typy grafov (heatmapa + scatter plot)
- ✅ Vzájomné prepojenie (klik na heatmapu → aktualizácia scatter plotu)
- ✅ Zoom pre detailné preskúmanie
- ✅ Filtrovanie/výber dátových bodov (dropdown podľa lokality)
- ✅ Interaktívne prvky (hover, klik, dropdown, legenda)
