import pandas as pd
import geopandas as gpd
from shapely import wkt
from shapely.geometry import Point, Polygon
import tkinter as tk
from tkinter import filedialog, messagebox, ttk

def count_meters_in_polygon_zones(meter_csv, circle_csv):
    # Load meters
    meters_df = pd.read_csv(meter_csv)
    meters_gdf = gpd.GeoDataFrame(
        meters_df,
        geometry=gpd.points_from_xy(meters_df['LONGITUDE'], meters_df['LATITUDE']),
        crs='EPSG:4326'
    )

    # Load circles and convert WKT LINESTRINGs to Polygons
    circles_df = pd.read_csv(circle_csv)
    circles_df['geometry'] = circles_df['WKT'].apply(lambda w: Polygon(wkt.loads(w).coords))
    circles_gdf = gpd.GeoDataFrame(circles_df, geometry='geometry', crs='EPSG:4326')

    # Spatial join: find which meters fall inside which polygons
    joined = gpd.sjoin(meters_gdf, circles_gdf, how='left', predicate='within')

    # Count meters in each polygon
    count_result = joined.groupby(joined.index_right).size().reset_index(name='TOTAL_METERS')

    # Add count and centroids
    circles_df['TOTAL_METERS'] = 0
    circles_df['CENTROID_LATITUDE'] = 0.0
    circles_df['CENTROID_LONGITUDE'] = 0.0
    circles_df['SERIAL_NO'] = range(1, len(circles_df) + 1)

    for idx, row in circles_df.iterrows():
        poly = row['geometry']
        if poly.is_valid:
            centroid = poly.centroid
            circles_df.at[idx, 'CENTROID_LATITUDE'] = centroid.y
            circles_df.at[idx, 'CENTROID_LONGITUDE'] = centroid.x

    for _, row in count_result.iterrows():
        idx = int(row['index_right'])
        if idx < len(circles_df):
            circles_df.at[idx, 'TOTAL_METERS'] = int(row['TOTAL_METERS'])

    return circles_df[['SERIAL_NO', 'WKT', 'TOTAL_METERS', 'CENTROID_LATITUDE', 'CENTROID_LONGITUDE']]

# GUI
def run_tool():
    meters_path = filedialog.askopenfilename(title="Upload Meters CSV", filetypes=[("CSV", "*.csv")])
    circles_path = filedialog.askopenfilename(title="Upload Circles CSV (with WKT)", filetypes=[("CSV", "*.csv")])
    if not meters_path or not circles_path:
        messagebox.showwarning("Missing File", "Please select both CSV files.")
        return
    try:
        result = count_meters_in_polygon_zones(meters_path, circles_path)
        output_path = filedialog.asksaveasfilename(defaultextension=".csv", title="Save Output CSV As")
        if output_path:
            result.to_csv(output_path, index=False)
            messagebox.showinfo("Success", f"✅ Output saved to:\n{output_path}")
    except Exception as e:
        messagebox.showerror("Error", f"❌ {e}")

# Fancy GUI
root = tk.Tk()
root.title("✨ Arman EDX Tool")
root.geometry("520x300")
root.configure(bg="#0f0f1f")

style = ttk.Style()
style.theme_use('clam')
style.configure('TFrame', background='#0f0f1f')
style.configure('TLabel', background='#0f0f1f', foreground='#00ffe1', font=('Orbitron', 12))
style.configure('TButton', font=('Orbitron', 11), background='#00ffe1', foreground='black')
style.map('TButton', background=[('active', '#00ffff')])

frame = ttk.Frame(root, padding="30 20 30 20")
frame.pack(fill=tk.BOTH, expand=True)

title = ttk.Label(frame, text="🛰️ Arman EDX Tool", font=("Orbitron", 18, "bold"))
title.pack(pady=10)

desc = ttk.Label(frame, text="Upload Meter & Circle CSVs to count meters in each gateway.")
desc.pack(pady=10)

process_button = ttk.Button(frame, text="📤 Upload & Count", command=run_tool)
process_button.pack(pady=25)

footer = ttk.Label(frame, text="🚀 Designed by Arman Mathew", font=("Orbitron", 9))
footer.pack(side="bottom", pady=10)

root.mainloop()
