# **Informe de Análisis de Datos – Prueba Técnica**  

## **Introducción**  
En esta prueba técnica se realiza un análisis de datos para evaluar discrepancias entre registros internos y transacciones de depósitos de jugadores. Se han proporcionado varios conjuntos de datos, incluyendo información sobre jugadores, fuentes de tráfico y depósitos realizados.  

El objetivo principal es identificar inconsistencias entre los registros, analizar posibles causas de las discrepancias y generar visualizaciones que ayuden a interpretar los resultados. Para ello, se utilizan herramientas como Pandas para manipulación de datos, SQL para consultas y Matplotlib para visualización.  

---

## **Metodología y Pasos Realizados**  

1. **Carga de datos:** Se importaron los archivos CSV en un Jupyter Notebook usando Pandas.  
2. **Exploración inicial:** Se analizaron las primeras filas y se verificó la estructura de los datos.  
3. **Limpieza de datos:** Se eliminaron valores nulos, duplicados y se estandarizaron tipos de datos.  
4. **Creación de base de datos SQLite:** Se almacenaron los datos en SQLite para realizar consultas estructuradas.  
5. **Cálculo de métricas clave:** Se generaron tablas para primeros depósitos (FTD) y jugadores CPA.  
6. **Comparación con registros internos:** Se analizaron discrepancias entre los datos esperados y los reales.  
7. **Visualización de datos:** Se generaron gráficos para interpretar mejor los patrones y diferencias.  

---

## **Código y Análisis**  

### **1. Carga de Datos**  
Los archivos CSV se importan en Pandas y se convierten las fechas al formato correcto.  

```python
import sqlite3  
import pandas as pd  
import numpy as np  
import matplotlib.pyplot as plt  
import seaborn as sns  

# Configuración de estilo para gráficos
plt.rcParams['figure.figsize'] = (10, 5)

# Cargar datos en DataFrames
df_players = pd.read_csv("./data/players.csv")
df_traffic_sources = pd.read_csv("./data/traffic_sources.csv")
df_deposits = pd.read_csv("./data/deposits.csv")
df_internal_records = pd.read_csv("./data/internal_records.csv")

# Convertir columnas de fecha a tipo datetime
df_players["registration_date"] = pd.to_datetime(df_players["registration_date"], errors='coerce')
df_traffic_sources["registration_date"] = pd.to_datetime(df_traffic_sources["registration_date"], errors='coerce')
df_deposits["deposit_date"] = pd.to_datetime(df_deposits["deposit_date"], errors='coerce')
df_internal_records["month"] = pd.to_datetime(df_internal_records["month"], errors='coerce')
```

---

### **2. Creación de Base de Datos SQLite**  
Se crea una base de datos SQLite para almacenar y analizar los datos de manera estructurada.  

```python
# Conectar a SQLite
conn = sqlite3.connect("gaming_data.db")
cursor = conn.cursor()

# Crear tablas
cursor.executescript('''
    CREATE TABLE IF NOT EXISTS players (
        player_id INTEGER PRIMARY KEY,
        name TEXT,
        country TEXT,
        registration_date DATE
    );
    CREATE TABLE IF NOT EXISTS traffic_sources (
        player_id INTEGER,
        trafficker TEXT,
        cost_of_acquisition REAL,
        registration_date DATE,
        source TEXT
    );
    CREATE TABLE IF NOT EXISTS deposits (
        deposit_id INTEGER PRIMARY KEY,
        player_id INTEGER,
        deposit_date DATE,
        deposit_amount REAL
    );
    CREATE TABLE IF NOT EXISTS internal_records (
        month DATE,
        expected_ftd INTEGER,
        expected_cpa INTEGER,
        status TEXT
    );
''')
conn.commit()
```

Los datos se insertan en las tablas:  

```python
df_players.to_sql("players", conn, if_exists="replace", index=False)
df_traffic_sources.to_sql("traffic_sources", conn, if_exists="replace", index=False)
df_deposits.to_sql("deposits", conn, if_exists="replace", index=False)
df_internal_records.to_sql("internal_records", conn, if_exists="replace", index=False)
```

---

### **3. Cálculo de Primeros Depósitos (FTD) y Jugadores CPA**  
Se crean dos tablas en SQLite para analizar el comportamiento de los jugadores.  

```python
# Tabla de primeros depósitos (FTD)
cursor.execute("DROP TABLE IF EXISTS first_time_deposits;")
cursor.execute('''
    CREATE TABLE first_time_deposits AS
    SELECT
        player_id,
        MIN(deposit_date) AS first_deposit_date
    FROM deposits
    GROUP BY player_id;
''')
conn.commit()

# Tabla de jugadores CPA
cursor.execute("DROP TABLE IF EXISTS cpa_players;")
cursor.execute('''
    CREATE TABLE cpa_players AS
    SELECT
        player_id,
        SUM(deposit_amount) AS total_deposits
    FROM deposits
    GROUP BY player_id
    HAVING total_deposits > 100;
''')
conn.commit()
```

---

### **4. Análisis de Discrepancias**  
Se comparan los datos obtenidos con los registros internos para identificar inconsistencias.  

```python
# Reporte de FTD por mes
query_ftd_month = '''
SELECT 
    strftime('%Y-%m', first_deposit_date) AS month,
    COUNT(player_id) AS ftd_count
FROM first_time_deposits
GROUP BY month
ORDER BY month;
'''
df_ftd_report = pd.read_sql(query_ftd_month, conn)

# Reporte de CPA por mes
query_cpa_month = '''
SELECT 
    strftime('%Y-%m', d.deposit_date) AS month,
    COUNT(DISTINCT d.player_id) AS cpa_count
FROM deposits d
JOIN cpa_players c ON d.player_id = c.player_id
GROUP BY month
ORDER BY month;
'''
df_cpa_report = pd.read_sql(query_cpa_month, conn)
```

Se comparan estos datos con los registros internos:  

```python
df_comparison = df_internal_records.merge(df_ftd_report, how="left", on="month")
df_comparison = df_comparison.merge(df_cpa_report, how="left", on="month")
df_comparison.fillna(0, inplace=True)

df_comparison["ftd_discrepancy"] = df_comparison["ftd_count"] - df_comparison["expected_ftd"]
df_comparison["cpa_discrepancy"] = df_comparison["cpa_count"] - df_comparison["expected_cpa"]
```

---

### **5. Visualización de Datos**  

#### **Gráfico de discrepancias en FTD y CPA**  
```python
plt.figure(figsize=(10, 4))
plt.plot(df_comparison["month"].astype(str), df_comparison["ftd_discrepancy"], marker='o', linestyle='-', label="FTD")
plt.plot(df_comparison["month"].astype(str), df_comparison["cpa_discrepancy"], marker='s', linestyle='-', label="CPA")
plt.xlabel("Mes")
plt.ylabel("Discrepancia")
plt.title("Discrepancias en FTD y CPA")
plt.legend()
plt.xticks(rotation=45)
plt.grid(True)
plt.show()
```

#### **Histórico de depósitos por mes**  
```python
df_deposits["month_year"] = df_deposits["deposit_date"].dt.to_period("M")
df_monthly_deposits = df_deposits.groupby("month_year")["deposit_amount"].sum().reset_index()
plt.figure(figsize=(10, 4))
plt.plot(df_monthly_deposits["month_year"].astype(str), df_monthly_deposits["deposit_amount"], marker='o', linestyle='-')
plt.xlabel("Mes")
plt.ylabel("Total Depositado ($)")
plt.title("Histórico de Dinero Depositado por Mes")
plt.xticks(rotation=45)
plt.grid(True)
plt.show()
```

---

## **Conclusiones y Hallazgos**  
- Se encontraron discrepancias en los registros internos con respecto a los datos reales de depósitos.  
- Se observaron fluctuaciones en los depósitos mensuales y en la captación de jugadores.  
- Se recomienda una revisión del origen de los datos para evitar inconsistencias en los informes internos.  




