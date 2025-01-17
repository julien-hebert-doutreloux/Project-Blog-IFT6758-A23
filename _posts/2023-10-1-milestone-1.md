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

Un notebook est fournit dans le [GitHub](https://github.com/julien-hebert-doutreloux/Project-DS-IFT6758-A23/blob/master/notebooks/Milestone1.ipynb) pour procéder à l'acquisition de données. Les lignes correspondantes pour le téléchargement des données sont :
```
seasons = [2016, 2017, 2018, 2019, 2020]
game_types = ('02','03')
raw_data_path= os.path.join('.','..','data','raw','json')

# une fonction pour ranger le fichier a chemin
# raw_data_path est inclu dans la classe DA
DA.download_season(seasons, game_types, raw_data_path)
DA.download_play_type()
```

## 2. Outil de débogage interactif

Nous avons créé un outil de débogage interactif. Son principe est assez simple. L'utilisateur peut sélectionner les parties de la saison régulière ou les parties des séries élimiatoires. Ensuite, il peut choisir la saison de son choix (2016, 2017, 2018, 2019 ou 2020). Ces deux options permettent d'ajuster le contenu d'un menu déroulant contenant les parties (menu déroulant GAME_ID). Par exemple, en choisissant "Playoffs" et "2018", le menu déroulant GAME_ID ne contiendra que les parties des séries éliminatoires de 2018. Ensuite, il nous suffit de choisir une partie qui nous intéresse. Un <i>int slider</i> (EVENT ID) nous permet de parcourir tous les événements de la partie sélectionnée. Un petit tableau HTML nous indique les équipes et le score pour chaque partie. Par ailleurs, un tableau interactif dessiné avec la librairie matplotlib nous permet de visualiser la location sur la patinoire et la description de chaque événement. Finalement, un bouton "Default Values" nous permet de retourner aux valeurs par défaut pour notre outil, soit le premier événement de la première partie régulière de la saison 2016.

```python
%matplotlib notebook
from ipywidgets import *
import matplotlib.pyplot as plt
from matplotlib import image

import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)

raw_data_path = os.path.join('.','..','data','raw','json')
files = os.listdir(raw_data_path)

game_dic = {
  "Regular Season": "02",
  "Playoffs": "03",
}

# Helper functions

# Retourne une liste de parties à afficher dans le drop-down Game ID
def display_game(var):
    season = B.value
    game_type = A.value
    game_list = []
    
    for file in files:
        temp_name = file.replace('.json', '')
        
        if temp_name[:4] == str(season) and temp_name[4:-4] == game_dic[str(game_type)]:
            game_list.append(temp_name)
      
    C.options = game_list
    C.value = game_list[0]

    return game_list

# Fonction pour mettre à jour le tableau des scores ainsi que le range du int slider Event ID
def update_int_slider(game_list):
    
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

# Fonction pour mettre à jour le graphe
def update_graph(game_list):

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

B = Dropdown(options=["2016", "2017", "2018", "2019", "2020"], value="2016", description='Season')
B.observe(display_game)

C = Dropdown(options=[], description='Game ID')
C.observe(update_int_slider, names='value')

# max initialisé à 1, mais mis à jour la ligne suivante
D = IntSlider(value=0, min=0, max=1, step=1, description='Event ID')
# Pour avoir les valeurs de la partie affichée (exemple: 67 events)
initial_game_list = display_game(0)
D.observe(update_graph, names='value')

# Valeurs par défaut des widgets
A.default_value = 'Regular Season'
B.default_value = "2016"
C.default_value = "2016020001"
D.default_value = "0" 

defaulting_widgets = [A,B,C,D]

default_value_button = Button(description='Default Values')

# Fonction pour réinitialiser les widgets
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

img_data_path = os.path.join('.','..', 'figures')
img_file_name = "nhl_rink.png"
im = plt.imread(os.path.join(img_data_path, img_file_name))

fig, axes = plt.subplots()

im = axes.imshow(im, extent=[-100, 100, -42.5, 42,5])

location, = axes.plot(-77, 5, marker='o', color="blue")

plt.title('Mitchell Marner Wrist Shot saved by Craig Anderson')

plt.show()
```
![alt text](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/outil_deboguage.jpg)

## 3. Nettoyer les données

Les données qui nous sont envoyées par l'API de la LNH sont vastes. Pour un projet de sciences des données comme le nôtre, il faut premièrement décider de quelles données nous jugeons utiles. Par ailleurs, il nous faut aussi choisir une structure pour nos données. Dans notre cas, nous avons extrait les données pertinentes de chaque JSON et nous avons sauvegardés ces derniers sous la forme d'un fichier CSV. Par ailleurs, nous avons créer une classe GAME qui nous permet de convertir n'importe quel JSON brute en un dataframe pandas contenant les données utiles.


Comme pour l'[acquisition de données](#acquisition_données) une section de notre notebook est un tutoriel pour nettoyer les données. Voic le code:

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
Nous avons téléchargé les saisons 2016 à 2020. Pour chaque partie, nous avons les statistiques des joueurs participants. Une donnée intéressant serait de compiler les statistiques individuelles pour chaque joueur sur l'ensemble des saisons 2016 à 2020 afin de créer un score global pour chaque joueur. Ce score serait particulièrement utile pour faire des prédictions avec nos données ou tout simplement pour rendre nos données plus intéressantes. Par exemple, si un excellent joueur prend une pénalité, l'événement est beaucoup plus grave que si le même événement arrive à un joueur de dernier ordre. Par ailleurs, nous pourrions connaitre la force des gardiens de buts de chaque équipe. Nous pourrions aussi évaluer la force d'une équipe à partir de la force de ses joueurs.
  </li>
  <li>
Nous pourrions reprendre la logique évoquée au point précédent, mais cette fois-ci en l'appliquant aux équipes. Nous pourrions évaluer la tendance de plusieurs variables sur n nombres de parties précédentes. Cette tendance pourrait inclure le nombre de buts marqués par partie, le ratio victoires/défaites, etc. Nous pourrions donc entre autres savoir si une équipe est lancée dans une série fulgurantes de victoires ou si elle traverse un désert marqué de défaites. Nous pourrions aussi savoir si une équipe en particulier prend beaucoup de pénalités. Nous pourrions également recréer le classement des équipes selon leurs nombres de victoires et de défaites.
  </li>
  <li>
Nous pourrions reprendre la logique évoquée aux deux points précédents, mais cette fois-ci en l'appliquant à l'ensemble de la LNH. Par exemple, aucune équipe canadienne n'a gagné la coupe Stanley depuis 1993. Connaître ce genre de tendance globale éclaire notre compréhension des dynamiques au sein de la LNH. Nous pourrions aussi évaluer les ratios de victoire/défaite entre les différentes conférences et les différentes divisions. Nous pourrions aussi évaluer les chances d'une équipe de gagner la coupe Stanley selon sa position dans le classement. 
  </li>
</ol>

## 4. Visualisations simples

![Question 1](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q1.png)

**Quel semble être le type de tir le plus dangereux?**  
**Important** Nous partons de la prémisse que le taux de succès d'un tir est égal à :
<em><strong>but / (but + tir)</strong></em>

Nous voyons que le type de tir avec le plus haut taux de succès est le "Deflected". Or, en tenant compte de la fréquence de tir, nous concluons que le "Tip-in" est le type de tir le plus dangereux, puisque que son taux de succès n'est inférieur que de 1% au "Deflected" (17% contre 18%), tandis que sa fréquence est largement supérieure (4449 contre 1450). 

**Le type de tir le plus courant?**  

De tout évidence, le "Wrist Shot" est le tir le plus fréquent.  

**Pourquoi est-ce que vous avez choisi ce type de graphique?**  
Car il demontre bien la fréquence superposée entre les buts et les tirs.

![Question 2a](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_a.png)
![Question 2b](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_b.png)
![Question 2c](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q2_c.png)

**Quelle est la relation entre la distance à laquelle un tir a été effectué et la chance qu'il s'agisse d'un but ?**  
Nous pouvons observer qu'il existe clairement deux pics dans les bins de distance pour lesquelles il existe un taux de succès élévé ((0-50), (150-175)). Cette donnée nous a initialement laissé un peu perplexe, car notre hypothèse était qu'en général au hockey les buts sont plus fréquent à une courte distance. Or, nous nous sommes rendus compte que les tirs effectués à une longue distance ne sont pas si fréquents.
Aussi, nous pouvons confirmer que les tirs effectués à une courte distance sont plus dangereux et fréquents que ceux effectués à une longue distance.  

**Y a-t-il eu beaucoup de changements au cours des trois dernières saisons?**  
Non. Nous observons que la tendance s'est maintenue au cours des saisons.

![Question 3](https://raw.githubusercontent.com/julien-hebert-doutreloux/Project-Blog-IFT6758-A23/main/public/q3.png)

**Quels sont les types de tirs les plus dangereux ?**  
Conmme nous divisons le taux de succès (dangerosité) par type de tir et sa distance, nous observons qu'à certaines distances des tirs sont plus dangereux que d'autres. Par exemple, dans le graphe, nous pouvons vérifier que le type de tir "Wrap-around" est beaucoup plus dangereux à courte distance qu'à longue distance, tandis que le type de tir "Tip-in" a un taux de succès supérieur à une plus grande distance, malgré que nous savons qu'il n'est pas si fréquent à cette distance. Nous observons aussi qu'à certains distances, quelques types de tir n'ont aucune fréquence. Nous sommes aussi conscients du fait que quelques données aberrantes ont pu se glisser dans les données de la LNH. Par exemple, un "Wrap-around" est un tir qui s'effectue à une faible distance du but, puisqu'il nécessite que le joueur contourne par derrière le but adverse. Aussi, on pourrait s'attendre à ce qu'un wrap-around n'arrive que depuis une distance plus courte (moins de 50 pieds), ce qui n'est pas le cas selon nos données. 

## 5. Visualisations avancées
La méthodologie pour produire ces visualisations avancées est la suivante. Premièrement, les données ont été prétraitées en enlevant les tirs n'ayant pas de location et les tirs qui ont été fait à partir de la zone défensive et en inversant le signe des coordonnées (x,y) (pour garder la symétrie) des tirs ayant été fait du côté gauche de la patinoire par un joueur dont la zone défensive est du côté droit. Une fois le prétraitement des données fait, les données de location de tirs pour chaque équipe et pour la ligue ont été lissées à l'aide d'un estimateur de densité avec noyau gaussien et des prédictions de densité ont été faites pour chaque coordonnée (x,y) de la moitié de patinoire. Par la suite, ces densités ont été converties en tir moyen par heure en multipliant celles-ci par le nombre total de tir de l'équipe (ou de la ligue) divisé par le nombre total de parties jouées par l'équipe (ou par la ligue, dépendamment de si on fait le calcul pour une équipe ou pour la ligue). Finalement, ces résultats ont été multipliés par 100 pour obtenir des résultats finaux par 100 pieds carrés. Pour chaque équipe, la différence entre les résultats finaux de l'équipe et de la ligue a été calculée.

Note: Pour chacune des visualisations avancées ci-dessous, il faudra cliquer sur le dropdown bouton au moins une fois afin d'afficher l'image de la patinoire (bug plotly)

{% include dropdown_2016.html %}
{% include dropdown_2017.html %}
{% include dropdown_2018.html %}
{% include dropdown_2019.html %}
{% include dropdown_2020.html %}

# Réflexions sur les visualisations avancées
<ol>
  <li>
    Nous pouvons premièrement voir si une équipe effectue davantage de tirs aux buts à partir d'une zone en particulier comparativement aux autres équipes de la LNH pour une saison donnée. Nous pouvons constater quelles sont les zones fortes et les zones faibles pour l'attaque d'une équipe. Nous pouvons aussi comparer une équipe à elle-même à travers les saisons 2016 à 2021.
  </li>
  
  <li>
    Pour la saison 2016-2017, nous voyons que les Avalanches du Colorado ont une attaque assez faible, surtout devant le but. Pour la saison 2020-2021 au contraire, les Avalanches du Colorado effectuent davantage de tirs au but que la moyenne de la LNH sur une grande partie de la patinoire. Ces renseignements corroborent les résultats des classements pour ces deux saisons respectives. En 2016-2017 en effet, les Avalanches ont fini dernier de leur division avec 22 victoires et 56 défaites, tandis qu'en 2020-2021, ils ont fini premier de leur division avec 39 victoires et 13 défaites.
  </li>
  
  <li>
    Le Lightning de Tampa Bay a remporté la coupe Stanley deux saisons consécutives (2020 et 2021). Pour la saison 2020, nous voyons qu'ils effectuent davantage de tirs au but que la moyenne de la LNH devant le filet adverse. Le Lightning semble être une équipe avec une bonne aile droite, mais sur le reste de la patinoire, elle ne semble pas avoir une attaque particulièrement dominante. Pour cette même saison, ils ont obtenu 36 victoires et 17 défaites, ce qui les a hissé à la 3e position au classement de leur division. Ils ont donc été une équipe forte, mais pas autant que les Avalanches, qui ont obtenu un meilleur ratio de victoires par partie jouée (75% pour COL contre 67.9% pour TBL). 
    <br>
    Or, ce n'est pas toujours l'équipe qui a été la plus forte durant la saison qui remporte nécessairement la coupe Stanley. Pour la saison 2020, Colorado s'est fait éliminer en deuxième ronde des séries éliminatoires. Toutes sortes de données peuvent entrer en ligne de compte pour expliquer un tel phénomène, par exemple un joueur étoile qui se blesse avec les séries. Un graphique ne peut donc pas tout expliquer des forces et des faiblesses d'une équipe. Par exemple, nous voyons qu'une équipe comme les sabres de Buffalo semble avoir une attaque globalement assez faible pour la saison 2020. Or, il est difficle d'expliquer un tel résultat avec nos graphiques. Nous pouvons que le constater. 
    <br>
    Pour pousser l'analyser plus loin, il faudrait étudier pour chaque équipe des résultats comme le nombre de pénalités par match, le ratio de buts par tir, etc. Nous pourrions aussi produire le graphique inverse, c'est-à-dire combien de tirs ont été effectués contre une équipe dans la zone défensive. Nous pourrions alors obtenir une information cruciale, car bien qu'une équipe puisse tirer souvent sur le but adverse, elle doit veiller à minimiser le nombre de tirs envoyés contre son propre but. 
  </li>
</ol>
