# Typy zadań

## Dane wektorowe

1. Wczytanie na różne sposoby danych wektorowych:
```python
# many
contour_lines = [os.path.join(folder_path, file) for file in os.listdir(folder_path) if file.endswith('.shp') and file.startswith('c')]
for el in contour_lines:
    gdf = gpd.read_file(el)

# single
coimbra = 'f1ex1/limites_coimbra.shp'
df_coimbra = gpd.read_file(coimbra)
```
2. Nadanie wspólnego systemu odniesienia dla warstw:
```python
# single
df_coimbra = df_coimbra.to_crs(CRS('ESRI:102164'))

# many
target_crs = CRS('ESRI:102164')
gdfs = []
for el in contour_lines:
    gdf = gpd.read_file(el)
    print(el)
    print(gdf.crs)
    gdf.crs = target_crs
    print(gdf.crs)
    gdfs.append(gdf)
```
3. Złączenie wielu warstw (union):
```python
union_set = gpd.GeoDataFrame()

for gdf in gdfs:
    union_set = gpd.GeoDataFrame(pd.concat([union_set, gdf], ignore_index=True), crs=gdf.crs)

union_set.plot()
```
4. Przycięcie warstwy wektorowej do innej warstwy wektorowej:
```python
clipped_lines = set1.clip(set2)  # set1 - input features, set2 - clip features (foremka)
```
5. Wybór rekordów z tabeli atrybutów:
```python
print(pd.unique(df2007.COS07n1_L)) # unique records in column

df2007_agr = df2007[df2007.COS07n1_L == 'Agricultura']
df2018_terr_art = df2018[df2018.COS18n1_L == 'Territórios artificializados']
```
7. Wzięcie części wspólnej warstw (intersect):
  ```python
common_polygons = gpd.overlay(df2007_agr, df2018_terr_art, how='intersection', keep_geom_type=True)
```
8. Obliczenie powierzchni
```python
# ha
total_area_hectares = sum(common_polygons['geometry'].area / 10000)

# meters2
total_area_hectares = sum(common_polygons['geometry'].area)

# km2
total_area_km2 = total_area_hectares * 0.01
```
9. Rozpuszczenie jednostek (dissolve) - np. na podstawie granic powiatów utworzyć wojewódźtwo
```python
freg = gpd.read_file('f1ex3/freg_regiao_coimbra.shp')

# Dissolve
union_coimbra = freg.unary_union

# Convert to geodataframe
union_coimbra_gdf = gpd.GeoDataFrame(geometry=[union_coimbra])
```
10. Wyświetlenie ogólnych informacji o warstwie, kolumnach w tabeli atrybutów:
```python
data.info()
```
11. Zliczenie liczby punktów w poszczególnych poligonach (np. liczba aptek w dzielnicach) - część wspólna, spatial join:
```python
joined_data = gpd.sjoin(farmacias, freg_cmb, how='left', predicate='within')

farmacias_in_freg = joined_data.groupby('Freguesia').size().reset_index(name='farmacies')
freg_coimbra_with_farmacias = freg_cmb.set_index('Freguesia').join(farmacias_in_freg.set_index('Freguesia'))

freg_coimbra_with_farmacias.head()
```
12. Obliczenie dla każdej jednostki liczby punktów na 10 000 mieszkańców:
```python
freg_coimbra_with_farmacias['result'] = (freg_coimbra_with_farmacias['farmacies'] / freg_coimbra_with_farmacias['popres11']) * 10000

freg_coimbra_with_farmacias['result']
```
13. Zliczenie powierzchni poligonu w obrębie innych poligonów (np. powierzchnia obszarów zielonych w każdej z dzielnic):
```python
urban_green_space_clipped = gpd.sjoin(freg_cmb, urban_green_space, how='right', predicate='intersects')

urban_green_space_clipped['area_m2'] = urban_green_space_clipped['geometry'].area
freg_cmb['area_green'] = urban_green_space_clipped.groupby('index_left')['area_m2'].sum()
```
14. Policzenie powierzchni przypadającej na osobę:
```python
freg_cmb['green_per_person'] = freg_cmb['area_green']/freg_cmb['popres11']
freg_cmb.green_per_person
```
15. Join tabel atrybutów o różnych nazwach kolumny z indeksami:
```python
merged_data = bgri.merge(freg, left_on='idfreg', right_on='DICOFRE', how='inner') # bgri --> idfreg, freg --> DICOFRE

# optional - create geodataframe:
merged_data_gdf = gpd.GeoDataFrame(merged_data, geometry='geometry_x')
```
16. Policzenie gęstości zaludnienia na ha:
```python
merged_data['population_density'] = merged_data['popres21'] / merged_data['Area_ha']

density_result = merged_data[['Freguesia', 'population_density']].groupby('Freguesia').sum()
density_result
```
17. Utworzenie bufora:
```python
recycling_bins_buffer = recycling_bins.copy()
recycling_bins_buffer['geometry'] = recycling_bins['geometry'].buffer(500)
```
18. Zliczenie liczby osób wewnątrz bufora:
```python
# in order to prevent count same people many times
recycling_bins_buffer = recycling_bins_buffer.unary_union
intersect  = gpd.overlay(merged_data_gdf, gpd.GeoDataFrame(geometry=[recycling_bins_buffer], crs=target_crs), how='intersection')

population_within_buffer = intersect.groupby('Freguesia')['popres21'].sum().reset_index()
population_within_buffer
```
19. Procentowy udział ludności w buforze i poza buforem:
```python
# whole population
total_population = merged_data_gdf.groupby('Freguesia')['popres21'].sum().reset_index()

result = pd.merge(population_within_buffer, total_population, on='Freguesia').set_index('Freguesia')
result.columns = ['pop_in_buffors', 'total_pop']
result['percentage_within_500m'] = result['pop_in_buffors'] / result['total_pop'] * 100
```
20. Podmienienie wartości w jednej warstwie na wartości z innej warstwy (np. dodanie do CLC miejsc gdzie były pożary):
```python
intersection_result = gpd.overlay(cos2018, burns, how='intersection') # wycinamy obszary z pożarami z CLC
intersection_result['COS18n1_L'] = 'Burned Areas'  # nadajemy nazwę wartościom

combined_gdf = pd.concat([intersection_result, cos2018], ignore_index=True) # łączymy ponownie dane w kompletną warstwę
```
21. Wyświetlenie nagłówka tabeli atrybutów:
```python
freg.head()
```
22. 
