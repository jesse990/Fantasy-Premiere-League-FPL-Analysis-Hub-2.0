# Fantasy-Premiere-League-FPL-Analysis-Hub-2.0
This document outlines the key decisions, architectural changes, and technical improvements made in rebuilding the FPL Analytics Dashboard from the ground up. The rebuild was motivated by maintainability issues in the original version. It started as a learning project which meant there was a lot of early mistakes and bad habits aswell as of organisation. This along with some small data quality issues meant that scalability would be difficult as the model grows. Aside from these issues, another motivation was the amount of requests for the pbix file and positive feedback i recieved after sharing on reddit, this made me aware of the importance organisation, documentation and a clear understandable pipeline.

#Problems With the Original Dashboard

1. Messy Power Query
The original dashboard handled all transformation, and loading within Power Query. Over time this became difficult to follow — steps were inconsistently named, logic was hard to trace, debugging any issue required navigating a long chain of tightly coupled steps, which also lead to long refresh times and even freezing despite my computer being equippped to handle a model the size of mine. A lot of this stemmed from the lack of global keys in team and player tables across seasons

2. Player IDs Breaking Between Seasons
FPL rebuilds its internal player IDs at the start of every season and the initial data i pulled had no global team or player ids so the original dashboard had no mechanism to handle this. My workaround was a lot of complex joins which worked but as i mentiopned in my last post it was often hard to follow and led to long refresh times.

#Architectural Changes

Python Data Pipeline
The rebuild now pulls all data directly from the API, bigger datasets are pulled first with Python scripts, and otehrs that require little transformation are pulled directly from the API in PowerBI . Power Query is now only responsible for reading CSV files and light transformation . 

Folder Based Incremental Loading
Each gameweek's data is saved as an individual CSV file (gw1.csv, gw2.csv etc.) in a dedicated folder. Power BI reads the entire folder and appends the files automatically. My previous approach was to download each gw file , and merge it to a singular file containing all previous gameweeks in python and that would be the file powerquery reads. This lead to some issues where i accidently ran the merge script twice which was a pain to dealwith. Now this means i can update the dashboard between games and dont have to wait till the end of the gameweek since running my script will just update the file for the current gw

Cross Season Player Identity (Opta Code)
FPL assigns new internal IDs to players each season, making cross-season joins unreliable by ID alone. The rebuild uses the code field available in bootstrap-static, which corresponds to the Opta player ID and remains stable across seasons. 

