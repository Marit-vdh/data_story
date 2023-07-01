---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

```{code-cell} ipython3
:tags: [hide-input]

import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import matplotlib.pyplot as plt
import csv
import statistics as st
import numpy as np
from myst_nb import glue

color_blind_red, color_blind_green, color_blind_blue = '#d95f02', '#1b9e77', '#7570b3'
```

```{code-cell} ipython3
:tags: [hide-input]

# Importeer het dataframe met de punten die gegeven zijn per songfestival
euro_df = pd.read_csv('eurovision_song_contest_1975_2019.csv')
euro_df.columns=[column.strip() for column in euro_df.columns]
# euro_df.head()
```

```{code-cell} ipython3
:tags: [hide-input]

# Kaart van de ontwikkeling van het gemiddelde van de punten in de finale per land
def total_per_year(dataframe):
    """
    Berekent de totale hoeveelheid punten die een land per jaar in de finale van het eurovisie
    van de vakjury heeft gekregen.
    """

    # Maak een lege dictionary aan voor het opslaan van de data
    total_year = {}

    # Itereer over de dataframe met gegeven punten
    for index, row in dataframe.iterrows():

        # Controleer of er een finale gezongen werd en of de punten gegeven zijn door
        # de vakjury
        if row[1] == 'f' and row[3] == 'J':

            # Controleer of het land al in het dictionary staat
            if row[5] in total_year:

                # Controleer of het jaar al bij het land staat
                if row[0] in total_year[row[5]]:

                    # Als het jaar al aanwezig is bij het land, worden de punten opgeteld
                    total_year[row[5]][row[0]] += row[6]

                # Als het jaar nog niet aanwezig is wordt het jaar toegevoegd aan het dictionary
                # van het land
                else:
                    total_year[row[5]].update({row[0]: row[6]})

            # Als het land nog niet in het dictionary staat, wordt een nieuwe key aangemaakt met
            # daarin weer een dictionary van het jaar en het aantal punten dat behaald is
            else:
                total_year.update({row[5]: {row[0]:row[6]}})

    # Maak een dataframe om het dictionary in te kunnen zetten
    total_per_year_df = pd.DataFrame({'Country':[], 'Year':[], 'Points':[]})

    # Voeg alle elementen van het dictionary per land toe aan het dataframe
    for country, euro_df in total_year.items():
        for year, score in euro_df.items():
            total_per_year_df.loc[len(total_per_year_df)] = [country, year, score]

    # Sorteer het dataframe zodanig dat er slices per land gesorteerd op jaar ontstaan
    # Dit garandeert goed verloop van de functie voor de gemiddeldes
    sorted_df = total_per_year_df.sort_values(['Country' , 'Year'], ascending=[True, True], ignore_index=True)

    return sorted_df

def development_of_mean(dataframe):
    """
    Berekent de verandering van het gemiddelde totale aantal punten dat een land per jaar heeft gekregen.
    Elk jaar wordt dus het gemiddelde geupdate naar het nieuwe aantal punten dat is behaald, tenzij
    er niet is deelgenomen aan de finale. In dat geval wordt hetzelfde gemiddelde nog eens toegevoegd
    om gaten in de uiteindelijke grafiek te voorkomen. De functie vult ook de waardes aan tot het jaar
    2019, zodat de grafiek bij het eindpunt van de animatie alle uiteindelijke gemiddeldes bevat.
    """

    # Creëer een dataframe met de totale aantal punten per land per jaar
    total_per_year_df = total_per_year(dataframe)

    # Maak een dictionary met het eerste land, jaar en punten
    mean_per_year = {total_per_year_df.iloc[0,0] : {total_per_year_df.iloc[0,1]: total_per_year_df.iloc[0,2]}}

    # Maak counters om de gemiddelden mee te kunnen berekenen
    total_years = 0
    total_points = total_per_year_df.iloc[0,2]

    # Loop over het dataframe met totale aantal punten, sla de eerste rij over
    for index, row in total_per_year_df[1:].iterrows():
        # Controleer of het land hetzelfde is als in de rij ervoor
        if row[0] == total_per_year_df.iloc[index - 1, 0]:

            # Als het land hetzelfde is, controleer of het jaar maar één verschilt
            # Zo niet, vul dan de missende jaren aan met het bestaande gemmiddelde
            if row[1] != (total_per_year_df.iloc[index - 1, 1] + 1):
                for year in range(total_per_year_df.iloc[index - 1, 1] + 1, row[1]):
                    mean_per_year[row[0]].update({year : (total_points / total_years)})

            # Vul het dict aan met het jaar en het nieuwe gemiddelde
            total_points += row[2]
            total_years += 1
            mean_per_year[row[0]].update({row[1] : (total_points / total_years)})

        else:

            # Als het land niet hetzelfde is als de rij ervoor, controleer of het laatst aangevulde jaar 2019 is
            # Zo niet, vul dan het dict aan met hetzelfde gemiddelde tot 2019
            if total_per_year_df.iloc[index - 1, 1] != 2019:
                for year in range(total_per_year_df.iloc[index - 1, 1] + 1, 2020):
                    mean_per_year[total_per_year_df.iloc[index - 1, 0]].update({year : (total_points / total_years)})

            # Update het dict met het nieuwe land, jaar en aantal punten
            mean_per_year.update({row[0]: {row[1] : row[2]}})

            # Zet het totale aantal punten naar het ereste aantal punten en zet het aantal jaren op 1
            total_points = row[2]
            total_years = 1

    # Maak een dataframe om het dictionary in te kunnen zetten
    mean_per_year_df = pd.DataFrame({'Country':[], 'Year':[], 'Points':[]})

    # Itereer over het dictionary om de waardes in het dataframe te zetten
    for country, euro in mean_per_year.items():
        for year, score in euro.items():
            mean_per_year_df.loc[len(mean_per_year_df)] = [country, year, score]

    return mean_per_year_df

# Creëer een dataframe met het gemiddelde aantal punten per jaar
mean_per_year = development_of_mean(euro_df)

year_order = [year for year in range(1957, 2020)]

# Creëer een figuur met een wereldkaart waar per jaar de gemiddelde punten op worden laten zien
fig_per_year = px.choropleth(mean_per_year,
                    category_orders={'Year': year_order},
                    locationmode='country names',
                    locations='Country',
                    color='Points',
                    animation_frame='Year',
                    title='Gemiddelde van totale aantal punten in de finale per land per jaar',
                    height=700,
                    color_continuous_scale=['#1E88E5', '#FFC107', '#FF005D'],
                    labels={'Country': 'Land', 'Year': 'Jaar', 'Points': 'Gemiddeld aantal punten'})

# fig_per_year.add_annotation(dict(font=dict(color='black',size=15),
#                                         x=0,
#                                         y=-0.12,
#                                         showarrow=False,
#                                         text="Figuur 1: Een geoplot die het gemiddeld gehaalde aantal punten van landen in de finale weergeeft per jaar",
#                                         textangle=0,
#                                         xanchor='left',
#                                         xref="paper",
#                                         yref="paper"))

# Laat het figuur zien en zet vast in een markdown variabele
glue("map_per_year", fig_per_year, display=True)

```

`{glue:figure} map_per_year `
