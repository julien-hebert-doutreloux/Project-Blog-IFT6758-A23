---
layout: post
title: Milestone 1
---

## Table des matières

1. [Acquisition de données](#acquisition_donnees)
2. [Outils de débogage intéractif](#outil_debogage)
3. [Nettoyage de données](#nettoyage_donnees)
4. [Visualisations simples](#visualisations_simples)
4. [Visualisations avancées](#visualisations_avancees)


## 1. Acquisition de données

La LNH rend disponible de nombreuses données par le biais d'une API publique dont le end-point est le suivant : https://statsapi.web.nhl.com/api/v1/game/[GAME_ID]/feed/live/. Chaque requête envoyé à ce end-point renvoie un fichier json contenant des renseignements sur la partie en question.

Les parties ont un identifiant (GAME_ID) qui suit une structure comme suit: 2017020008

<ul>
  <li>Les 4 premiers chiffres désignent la saison (2017 dans le cas de notre exemple)</li>
  <li>Les deux chiffres suivants font référence au type de partie (01 = présaison, 02 = saison régulière, 03 = séries éliminatoires, 04 = match des étoiles)</li>
  <li>Les 4 derniers chiffres désignent le numéro de la partie (0008 dans le cas de notre exemple)</li>
</ul>

Avec ces renseignements en main, il nous est donc possible d'effectuer une série de requêtes au serveur de la LNH afin de télécharger les données d'une saison en particulier. Par exemple, si nous désirons télécharger les données pour la saison régulière de 2017, il suffit d'aller chercher la partie 2017020001 (2017 pour l'année, 02 pour spécifier que nous voulons les parties de la saison régulière et 0001 pour spécifier le numéro de la première partie de la saison). Ensuite, nous pouvons envoyer une requête pour la partie 2017020002, puis 2017020003, 2017020004, etc.

Un notebook est fournit dans le [GitHub](https://github.com/julien-hebert-doutreloux/Project-DS-IFT6758-A23/blob/master/notebooks/Milestone1.ipynb) pour accomplir l'acquisition de données. Les lignes correspondantes pour le téléchargement des données sont :
```
seasons = [2017, 2018, 2019, 2020, 2021]
game_types = ('02','03')
raw_data_path= os.path.join('.','..','data','raw','json')

# une fonction pour ranger le fichier a chemin
# raw_data_path est inclu dans la classe DA
DA.download_season(seasons, game_types, raw_data_path)
DA.download_play_type()
```

## 2. Outil de débogage interactif

Nous avons créé un outil de débogage interactif. Son principe est assez simple. L'utilisateur peut sélectionner les parties de la saison régulière ou les parties des séries élimiatoires. Ensuite, il peut choisir la saison de son choix (2017, 2018, 2019, 2020 ou 2021). Ces deux options permettent d'ajuster le contenu d'un menu déroulant contenant les parties (menu déroulant GAME_ID). Par exemple, en choisissant "Playoffs" et "2018", le menu déroulant GAME_ID ne contiendra que les parties des séries éliminatoires de 2018. Ensuite, il nous suffit de choisir une partie qui nous intéresse. Un <i>int slider</i> (EVENT ID) nous permet de parcourir tous les événements de la partie sélectionnée. Un petit tableau HTML nous indique les équipes et le score pour chaque partie. Par ailleurs, un tableau interactif dessiné avec la librairie matplotlib nous permet de visualiser la location sur la patinoire et la description de chaque événement.

```python
%matplotlib notebook
from ipywidgets import *
import matplotlib.pyplot as plt
from matplotlib import image
import numpy as np
import csv
import ast
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)

dir_path = os.path.join('.','..','data\processed\csv')
files = os.listdir(dir_path)

game_dic = {
  "Regular Season": "02",
  "Playoffs": "03",
}

# Helper functions

def display_game(var):
    season = B.value
    game_type = A.value
    game_list = []
    
    for file in files:
        temp_name = file.replace('.csv', '')
        
        if temp_name[:4] == str(season) and temp_name[4:-4] == game_dic[str(game_type)]:
            game_list.append(temp_name)
      
    C.options = game_list

    return game_list

def update_int_slider(game_list):
    
    raw_data_path= os.path.join('.','..','data','raw','json')
    file = C.value+".json"
    file_path = os.path.join(raw_data_path, file)

    if os.path.isfile(file_path):
        game = Game(path_or_url = file_path)
        df = game.clean
        D.max = df.index.values.tolist()[-1]
        
        info_text.value="""
        <table>
          <tr>
            <th style="padding:10px"></th>
            <th style="padding:10px">Home</th>
            <th style="padding:10px">Away</th>
          </tr>
          <tr>
            <td style="padding-right:10px">Teams</td>
            <td>{}</td>
            <td>{}</td>
          </tr>
          <tr>
            <td style="padding-right:10px">Final Score</td>
            <td>{}</td>
            <td>{}</td>
          </tr>
        </table>
        """.format(game.teams['home']['triCode'],game.teams['away']['triCode'],
                   game.final_score['home'], game.final_score['away'])        
    else:
        print(file_path + " est un fichier non existant")
    return

def update_graph(game_list):

    raw_data_path= os.path.join('.','..','data','raw','json')
    file = C.value+".json"
    file_path = os.path.join(raw_data_path, file)

    if os.path.isfile(file_path):
        game = Game(path_or_url = file_path)
        df = game.clean
        value = df.iloc[D.value]
        
        # update puck location
        location.set_ydata(value['y'])
        location.set_xdata(value['x'])
        
        event_description = str(value['event_description'])
              
        plt.title(event_description)
        
    else:
        print(file_path + " est un fichier non existant")
    return


# Widgets

info_text = widgets.HTML()

A = Dropdown(options=["Regular Season", "Playoffs"], value='Regular Season', description='Game Type')
A.observe(display_game)

B = Dropdown(options=["2017", "2018", "2019", "2020", "2021"], value="2017", description='Season')
B.observe(display_game)

C = Dropdown(options=[], description='Game ID')
C.observe(update_int_slider, names='value')

# max initialisé à 1, mais mis à jour la ligne suivante
D = IntSlider(value=0, min=0, max=1, step=1, description='Event ID')
# Pour avoir les valeurs de la partie affichée (exemple: 67 events)
initial_game_list = display_game(0)
D.observe(update_graph, names='value')

A.default_value = 'Regular Season'
B.default_value = "2017"
C.default_value = "2017020001"
D.default_value = "0" 

defaulting_widgets = [A,B,C,D]

default_value_button = Button(description='Default Values')

def set_default(button):
    for widget in defaulting_widgets:
        widget.value = widget.default_value

default_value_button.on_click(set_default)

display(A,B,C,D, default_value_button)

# infos sur les parties (table html)

display(info_text)

info_text.value="""
<table>
  <tr>
    <th style="padding:10px"></th>
    <th style="padding:10px">Home</th>
    <th style="padding:10px">Away</th>
  </tr>
  <tr>
    <td style="padding-right:10px">Teams</td>
    <td>WPG</td>
    <td>TOR</td>
  </tr>
  <tr>
    <td style="padding-right:10px">Final Score</td>
    <td>2</td>
    <td>7</td>
  </tr>
</table>
"""

# image (graphique matplotlib)

plt.rcParams["figure.figsize"] = [7.00, 3.50]
plt.rcParams["figure.autolayout"] = True
im = plt.imread("C:\\Users\\joels\\PycharmProjects\\Project-DS-IFT6758-A23\\figures\\nhl_rink.png")

fig, axes = plt.subplots()

im = axes.imshow(im, extent=[-100, 100, -42.5, 42,5])

location, = axes.plot(-36, -28, marker='o', color="blue")

plt.title('Josh Morrissey Wrist Shot saved by Frederik Andersen')

plt.show()
```
![alt text](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/outil_deboguage_screenshot.jpg)

## 3. Nettoyer les données

Les données qui nous sont envoyées par l'API de la LNH sont vastes. Pour un projet de sciences des données comme le nôtre, il faut premièrement décider de quelles données nous jugeons utiles. Par ailleurs, il nous faut aussi choisir une structure pour nos données. Dans notre cas, nous avons extrait les données pertinentes de chaque JSON et nous avons sauvegardés ces derniers sous la forme d'un fichier CSV. Par ailleurs, nous avons créer une classe GAME qui nous permet de convertir n'importe quel JSON brute en un dataframe pandas contenant les données utiles.


Comme pour l'[aquisition de données](#acquisition_données) une section du notebook porte sur un tutoriel pour nettoyer les données fait par le code suivant:

```
DC.raw_data_path         = raw_data_path
DC.play_type_json_path   = os.path.join(DC.raw_data_path, 'play_type.json')
DC.considered_event_list = ['Shot', 'Goal']

file_list = os.listdir(raw_data_path)
processed_data_path = os.path.join('.', '..', 'data', 'processed', 'csv')
for file in file_list:
  if file.endswith('.json') and file != 'play_type.json':
    game = Game(path_or_url = os.path.join(raw_data_path, file))
    output_path = os.path.join(processed_data_path, f"{game.id}.csv")
    game.clean.to_csv(output_path, sep=',')
```

Voici une ligne de notre dataframe à titre d'exemple:
![alt text](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/df_exemple.jpg)

Nous pouvons remarquer que le champ de « force » (égal, avantage numérique ou en désavantage numérique) n'est indiqué que pour les buts et non pas pour les tirs. De plus, il ne précise pas le nombre de joueurs sur la glace (5 contre 4, ou 5 contre 3, etc.). Une façon simple d'inclure cette information serait de regarder les pénalités données pour chaque partie. Par exemple, si les Canadiens de Montréal recoivent une pénalité de 2 minute à la 7e minute de jeu de la 1ère période, puis une autre pénalité de 2 minutes à la 8e minute de jeu de la 1ère période, alors nous savons qu'entre la 8e minute et la 9e minute de jeu de la 1ère période les Canadiens sont en désavantage numérique à 5 contre 3.

Par ailleurs, voici trois caractéristiques supplémentaires que nous proposons de créer à partir des données disponibles:
<ol>
  <li>
Nous avons téléchargé les saisons 2017 à 2021. Pour chaque partie, nous avons les statistiques des joueurs participants. Une donnée intéressant serait de compiler les statistiques individuelles pour chaque joueur sur l'ensemble des saisons 2017 à 2021 afin de créer un score global pour chaque joueur. Ce score serait particulièrement utile pour faire des prédictions avec nos données ou tout simplement pour rendre nos données plus intéressantes. Par exemple, si un excellent joueur prend une pénalité, l'événement est beaucoup plus grave que si le même événement arrive à un joueur de dernier ordre. Par ailleurs, nous pourrions connaitre la force des gardiens de buts de chaque équipe. Nous pourrions aussi évaluer la force d'une équipe à partir de la force de ses joueurs.
  </li>
  <li>
Nous pourrions reprendre la logique évoquée au point précédent, mais cette fois-ci en l'appliquant aux équipes. Nous pourrions évaluer la tendance de plusieurs variables sur n nombres de parties précédentes. Cette tendance pourrait inclure le nombre de buts marqués par partie, le ratio victoires/défaites, etc. Nous pourrions donc entre autres savoir si une équipe est lancée dans une série fulgurantes de victoires ou si elle traverse un désert marqué de défaites. Nous pourrions aussi savoir si une équipe en particulier prend beaucoup de pénalités. Nous pourrions également recréer le classement des équipes selon leurs nombres de victoires et de défaites.
  </li>
  <li>
Nous pourrions reprendre la logique évoquée aux deux points précédents, mais cette fois-ci en l'appliquant à l'ensemble de la LNH. Par exemple, aucune équipe canadienne n'a gagné la coupe Stanley depuis 1993. Connaître ce genre de tendance globale éclaire notre compréhension des dynamiques au sein de la LNH. Nous pourrions aussi évaluer les ratios de victoire/défaite entre les différentes conférences et les différentes divisions. Nous pourrions évaluer les chances d'une équipe de gagner la coupe Stanley selon sa position dans le classement. 
  </li>
</ol>

## 4. Visualisations simples

![Question 1](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q1.png)

**Quel semble être le type de tir le plus dangereux?**  

Nous pouvons observer clairement que le type de tir avec un plus grand taux de succès et celui du 'Deflected', mais si nous comparons avec la frequence de tir, alors nous dirons que le 'Tip-in' est le type de tir le plus dangerous, vu qu'il est beaucoup plus frequent que celui de son succesor.  
**Le type de tir le plus courant?**  

Il est de tout évidence que le 'Wrist Shot' et le tir le plus frequent.  

**Pourquoi est-ce que vous avez choisi ce type de graphique?**  
Car il demontre bien la frequence superposé entre le but et les tir, mais aussi son taux de succès, ce qui met en évidence la dangerosité d'un tir face à une autre.

![Question 2a](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_a.png)
![Question 2b](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_b.png)
![Question 2c](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_c.png)

**Quelle est la relation entre la distance à laquelle un tir a été effectué et la chance qu'il s'agisse d'un but ?**  
Nous pouvons observer qu'il existe clairement deux pics dans les bins de distance dont il existe un taux de succès élévé ((0-50), (150-175)). Une évidence qu'au debut nous avait laissé un peu perplexe, car il est supposé qu'en général dans le hockey les but sont plus frequent à une court distance. Cela qu'en regardant aussi la frequence de tir nous nous sommes rendu compte que les tirs éffectués à un long distance ne sont pas si fréquent.
Alors nous pouvons confirmer que les tirs à une court distance sont plus dangerous et frequents, que ceux à une longue distance.  

**Y a-t-il eu beaucoup de changements au cours des trois dernières saisons?**  
Nous observons que la tendance s'est maintient au fur des saisons

![Question 3](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q3.png)

**Quels sont les types de tirs les plus dangereux ?**  
Vu que maintenant nous divisons le taux de succès (dangérosité) par type de tir et sa distance, alors nous observons qu'à certains distances quelques tirs sont plus dangerous que d'autres. Par exemple, dans le graph, nous pouvons vérifier que le type de tir "Wrap-around" est beaucoup plus dangerous à court distance qu'à long distance, tandis que le type de tir "Tip-in" a un taux de succès superieur à un plus grand distance, malgré qu'on sait qu'il n'est pas si fréquent à ce distance. Nous observons aussi qu'à certains distances, quelques types de tir ont aucun fréquent.

## 5. Visualisations avancées
La méthodologie pour produire ces visualisations avancées est la suivante. Premièrement, les données ont été prétraitées en enlevant les tirs n'ayant pas de location et les tirs qui ont été fait à partir de la zone défensive et en inversant le signe des coordonnées (x,y) (pour garder la symmétrie) des tirs ayant été fait du côté gauche de la patinoire par un joueur dont la zone défensive est du côté droit. Une fois le prétraitement des données fait, les données de location de tirs pour chaque équipe et pour la ligue ont été lissées à l'aide d'un estimateur de densité avec noyau gaussien et des prédictions de densité ont été faites pour chaque coordonnées (x,y) de la moitié de patinoire. Par la suite, ces densités ont été converties en tir moyen par heure en multipliant celles-ci par le nombre total de tir de l'équipe (ou de la ligue) divisé par le nombre total de parties jouées par l'équipe (ou par la ligue, dépendemment de si on fait le calcul pour une équipe ou pour la ligue). Finalement, ces résultats ont été multipliés par 100 pour obtenir des résultats finaux par 100 pieds carrés et, pour chaque équipe, la différence entre les résultats finaux de l'équipe et de la ligue ont été calculés.

Note: Pour chacune des visualisations avancées ci-dessous, il faudra cliquer sur le dropdown bouton au moins une fois avant de voir les vrai graphiques (bug plotly)

{% include dropdown_2016.html %}
{% include dropdown_2017.html %}
{% include dropdown_2018.html %}
{% include dropdown_2019.html %}
{% include dropdown_2020.html %}

