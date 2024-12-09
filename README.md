# League of Legends Role Analysis


## Introduction

League of Legends, a MOBA developed by Riot Games, is one of the most popular games in the world. It's especially fun to watch; as a result, there are many professional eSports tournaments every year that millions of people follow. The dataset that I am working on today contains data collected from tournaments that occurred in 2022. It has 150180 rows, and every 12 rows corresponds to a single match. League matches are 5v5, so the first 10 of the 12 rows correspond to the 10 individual players, while the latter 2 rows correspond to team statistics. This means that there are over 12000 matches recorded in this dataset. 

Because each match is a 5v5, it's important that every player contributes their best to their team. Each player plays a certain role and must play in a certain part of the map: *top*, *jg* (jungle), *mid* (middle), *bot* (bottom), and *sup* (support). Each role is, naturally, very different from one another. 

*Supports* can be a high-HP tank who soaks damage for the team or an enchanter who buffs damage output and/or protects the team. They begin the game in the bot lane and build items devoted to obtaining vision for the team. 
*Bot laners* are usually ranged marksmen who output the most damage on the team due to their auto attacks being ranged and having no cost. Their sustained damage is high, but they are easy to kill due to low defenses (armor and magic resist). Thus, the support accompanies them.
*Mid laners* are usually mages with large area-of-effect spells, but marksmen and assassins also see play here. Due to being in the middle of the map, they have very high impact and often coordinate with the jungler. They often act as another carry alongside the bot laner.
*Junglers* are a roaming role responsible for killing monsters in the jungle and helping to secure neutral objectives like Baron and Dragon. They also can exit the jungle into a lane to help out their teammates. 
*Top laners* are either tanks with a large amount of crowd-control (e.g. stuns) or fighters that are tanky but have high sustained damage in exchange for being melee. They put a lot of pressure on whatever lane they're on and, unlike bot laners, often act by themselves.


The overarching question I want to answer is: **How do the roles differ in terms of post-match statistics? Are they distinct enough that I can accurately predict which role a player played given their post-match data?** Even though there are 161 features, I'm only interested in 18 of them:

**datacompleteness**
: Whether this row's data is 'complete' or 'partial'. Relevant for data cleaning.
 
**position**
: The role a player played.
 
**champion**
: The champion a player played.

**gamelength**
: How long the match played by the player lasted in seconds.

**result**
: Whether the player won the match. 1 means victory, while 0 means defeat.

**kills**
: How many kills the player had.

**deaths**
: How many deaths the player had.

**assists**
: How many assists the player got by contributing to kills.

**teamkills**
: The total number of kills the player's team had.

**teamdeaths**
: The total number of deaths the player's team had.

**dpm**
: How much damage the player dealt per minute to champions.

**damageshare**
: The percentage of the team's total damage output that the player dealt.

**damagetakenperminute**
: The amount of damage taken by the player, averaged over number of minutes. 

**damagemitigatedperminute**
: The amount of damage reduced by the player through armor, magic resist, and shielding, averaged over number of minutes. 

**earnedgold**
: The amount of gold earned by the player; does not include passively earned gold.

**earnedgoldshare**
: The percentage of the team's total earned gold that the player earned.
 
**total cs**
: The total amount of minions and monsters killed by the player. 




## Data Cleaning and Exploratory Data

### Data Cleaning
After selecting only the above relevant columns, I first removed the rows for which **position** = 'team' because these rows' statistics do not reflect individual players and their roles and thus aren't relevant to my question. This reduces the total number of rows from 150180 to 125150. Checking for any missing values, I see that, for the selected columns, only the rows where **datacompleteness** is 'partial' has missing values. Analyzing further, I see that, apart from **damagemitigatedperminute**, which has almost 19000 missing values spread throughout the rows, the remaining missing values, if removed, would hardly affect the total number of rows. Thus, I removed the 20 rows that had missing values in **damagetochampions**, **dpm**, **damageshare**, **damagetakenperminute**, or **earnedgold**. 

#### Creating New Columns

**kills**, **deaths**, and **assists** are obviously important in determining which role a player played. Often, bot and mid laners will have the most kills, while supports will have the most assists. However, since all three are likely to accumulate more as **gamelength** increases, they aren't great metrics for differentiating roles on their own. For example, in a 30 minute game, 5 kills may be very little, but in a 15 minute game, it could be the majority of the team's kills. Thus, it makes more sense to normalize these statistics by the team's totals. 

As a result, I created three new columns: **killshare**, **deathshare**, and **assistshare**. **killshare** is calculated as **kills** / **teamkills**, **deathshare** as **deaths** / **teamdeaths**, and **assistshare** as **assists** / **teamkills** because assists are obtained through contributing to teammates' kills.

Another column I want to create is **damageabsorption**, which is **damagemitigatedperminute** / **damagetakenperminute**. The problem with **damagemitigatedperminute** and **damagetakenperminute** is that they will be naturally higher for matches with more fighting. **damageabsorption** is intended to provide a comparison between the amount of damage being mitigated by armor and magic resist and the amount of damage actually taken. This is important because, for example, bot laners almost never build armor or magic resist and also have low base stats in general, while top laners usually focus on it while having high base stats. For any instance of damage taken, the bot laner should have a low **damageabsorption**, while top laners should have a higher one. 

To create **damageabsorption**, however, **damagemitigatedperminute** needs to have its approximately 19000 missing values imputed, as I don't want to drop so many entries. 

#### Imputation

Instead of imputing **damagemitigatedperminute** first and then calculating **damageabsorption** directly, I decided to take the following steps:

1. Calculate **damageabsorption** as **damagemitigatedperminute** / **damagetakenperminute**.
2. Impute **damageabsorption**.
3. Directly calculate **damagemitigatedperminute** = **damageabsorption** * **damagetakenperminute**. 

For 2., I decided to use *conditional mean imputation* because **damageabsorption** is dependent on both the role and the champion. This meant that, for each missing value, I filled it in using the mean of the **damageabsorption** of all instances of that champion in that role. If no data exists for that champion-role combination, I instead filled it with the mean of the entire role. The idea is that, since pro play is quite standardized, most players are using the same builds for each champion, so each instance of a champion should have similar **damageabsorption**. If the champion has no other instances in the dataset, we use the role's average **damageabsorption** because it's likely that, for a particular role, champions are quite similar. When champions are used in different roles, however, they usually have different builds and even playstyles. For example, *Rell*, a tank usually used in the support role, became popular as a jungler for a while at some point. Due to supports' low gold income from not killing minions or monsters, Rell had to build different items than her jungler incarnation, which would result in a different **damageabsorption**.

The following graph shows the distribution of **damageabsorption** before and after imputation. We can see that the distibutions are very similar, indicating that the imputation process has preserved the overall trends and did a good job.

<iframe
  src="assets/impute.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Below is the head of the cleaned dataframe after removing some columns that will no longer be necessary after the Univariate and Bivariate analyses. 

| position   |   result |   damageshare |   earnedgoldshare |   damageabsorption |   killshare |   deathshare |   assistshare |
|:-----------|---------:|--------------:|------------------:|-------------------:|------------:|-------------:|--------------:|
| top        |        0 |     0.278784  |          0.253859 |           0.725283 |    0.222222 |     0.157895 |      0.222222 |
| jng        |        0 |     0.208009  |          0.19022  |           0.688527 |    0.222222 |     0.263158 |      0.666667 |
| mid        |        0 |     0.252086  |          0.210665 |           0.391605 |    0.222222 |     0.105263 |      0.333333 |
| bot        |        0 |     0.196358  |          0.242201 |           0.471872 |    0.222222 |     0.210526 |      0.222222 |
| sup        |        0 |     0.0647631 |          0.103054 |           1.03178  |    0.111111 |     0.263158 |      0.666667 |


### Univariate Analysis

I performed univariate analysis on **kills**, separated by **position**. 

<iframe
  src="assets/kills-univariate.html"
  width="800"
  height="400"
  frameborder="0"
></iframe> 
We can see that, as expected, supports get very few kills, while bot laners have the largest proportion of high-kill games. This makes sense; bot laners are usually ranged marksmen with auto attacks that not only deal high damage but also have no cooldown or mana cost, so unlike a mid laner mage who can blow up an enemy or two, bot laners are more likely to kill an entire team by themselves if left unchecked, and they're also better at securing kills due to their sustained damage. Mid laners are no slouch either when it comes to kills, however, having more high-kill games than other roles beside bot laner. On the other hand, top laners have a relatively large percentage of 0-kill games, excluding supports. This makes sense because top lane itself is more isolated from the other lanes, which can broadly be chalked up to the vulnerable bot laner being a huge target, more people being in bot lane in general, and Dragon being in the bot-side jungle. Speaking of the jungle, junglers seem to be in the middle of mid laners and top laners in terms of kill potential. 

Overall, this plot shows the difference in the roles' ability to obtain kills. 


### Bivariate Analysis

I performed bivariate analysis on **damagetakenperminute** and **dpm**, separated by **position**. 

<iframe
  src="assets/gold.html"
  width="800"
  height="400"
  frameborder="0"
></iframe> 
This shows that bot laners and mid laners, especially the former, have high DPS and take little damage, while supports' low gold income prevents them from excelling at either tanking or dealing damage. On the other hand, top laners and junglers both take a lot of damage while overall not dealing as much damage as mid laners or bot laners, demonstrating their higher HP and defenses. 

### Interesting Aggregates

The following pivot table shows the average difference in **earnedgold** between each **position** depending on **result**. 

result|    0 |       1 |   Average Gold Difference |
position|---:|--------:|--------------------------:|
bot| 8259.83 | 10758   |                   2498.19 |
 jg| 5658.76 |  7634.5 |                   1975.73 |
mid| 7401.45 |  9500.5 |                   2099.06 |
sup| 2770.29 |  4143.4 |                   1373.11 |
top| 7133.94 |  9171.3 |                   2037.36 |

We can see that bot laners have the highest difference between **earnedgold** depending on **result**, which indicates that getting gold to the bot laner is more important for victory than with other roles. They also have a lot more gold on average than other roles. This table also shows an important aspect of junglers in pro play, where each player is cooperating far more than selfish solo players would: due to resources being limited on the map, junglers often sacrifice their own gold from monsters to bot laners, so they often play less-gold-reliant champions. This is another factor as to why bot laners have the most gold on average while junglers have the second least.



This pivot table shows the average **damagemitigatedperminute** per **position**

result|       0 |       1 |
position|--------:|--------:|
bot| 292.644 | 279.771 |
 jg| 750.821 | 823.778 |
mid| 375.76  | 363.017 |
sup| 420.733 | 394.485 |
top| 752.699 | 749.987 |

Bot laners and mid laners both mitigate less damage when they win, which shows that, as the carries, taking damage is not conducive to victory, but the difference is not large overall. An interesting observation is that junglers and supports display opposite statistics, with the former mitigating much more damage in victories, while supports mitigate quite a bit more in defeats. This may indicate that tankier supports are not as effective as enchanters for winning, while tankier junglers are. Top laners have about the same level of damage mitigation regardless of victory, though, which may indicate that they always play tankier, frontline champions.

All in all, this is a very interesting table. 



## Framing a Prediction Problem

The prediction problem is as previously stated: **Are the roles distinct enough that I can accurately predict which role a player played given their post-match data?** This is a classification problem, and since there are 5 roles, this is multiclass classification. The response variable is **position**, which I chose because I've shown many ways in which each role differs from the others in their post-game statistics, and I think there's enough of a difference that it's possible for a model to do so. Additionally, I've always been interested in how much impact each role has on the game. 

As for metric, I'm going to use accuracy because the dataset is balanced, as each match has 2 players for each of the 5 roles. 

Below is the cleaned dataframe from earlier.

| position   |   result |   damageshare |   earnedgoldshare |   damageabsorption |   killshare |   deathshare |   assistshare |
|:-----------|---------:|--------------:|------------------:|-------------------:|------------:|-------------:|--------------:|
| top        |        0 |     0.278784  |          0.253859 |           0.725283 |    0.222222 |     0.157895 |      0.222222 |
| jng        |        0 |     0.208009  |          0.19022  |           0.688527 |    0.222222 |     0.263158 |      0.666667 |
| mid        |        0 |     0.252086  |          0.210665 |           0.391605 |    0.222222 |     0.105263 |      0.333333 |
| bot        |        0 |     0.196358  |          0.242201 |           0.471872 |    0.222222 |     0.210526 |      0.222222 |
| sup        |        0 |     0.0647631 |          0.103054 |           1.03178  |    0.111111 |     0.263158 |      0.666667 |



At the time of prediction, we know all of the information in the dataframe and want to predict **position**, so I will use all of the other features: **position**, **result**, **damageshare**, **earnedgoldshare**, **damageabsorption**, **killshare**, **deathshare**, and **assistshare**. 



## Baseline Model

My model is a `RandomForestClassifier` that uses the features **killshare**, **deathshare**, **assistshare**, **damageshare**, and **result**. The first 4 are all quantitative, while **result** is nominal. To standardize the scales of the quantitative features, `StandardScaler` is used on all of them, while `OneHotEncoder` is used on **result** because it is nominal. 

After being fit on the training data, the model gets an accuracy of `51%` on the testing data. Since a model that randomly predicts would have an average accuracy of `20%`, this current model is somewhat "good", but it still only predicts a player's role correctly half of the time. 


## Final Model

For the final model, I added **earnedgoldshare** and **damageabsorption**. **earnedgoldshare** helps bot laners stand out in particular, since they usually have the highest percentage of gold on the team, while junglers are quite below every role except support, the lowest, in this regard. Bot laners don't always have the most **damageshare** or **killshare**, so **earnedgoldshare** is great because bot laners scale very well and thus have a lot of gold funneled to them. **damageabsorption** should help because each of the roles focuses on different levels of offense and defense on average, allowing the model to better differentiate especially between mid laners and top laners, which are similar in **earnedgoldshare**. Thus, the model's performance should improve. These are both quantitative, so I used `StandardScaler` on them like the other quantitative columns.

The modeling algorithm I chose was, again, `RandomForestClassifier`, but this time I used `GridSearchCV` to find the best hyperparameters. In particular, combinations of the hyperparameters `n_estimators`, `min_samples_split`, and `max_features` were searched through. 

After fitting the model on the training data, I found that the best hyperparameters were `max_features` = 'sqrt', min_samples_split = `6`, and n_estimators = 200. The model got an accuracy of `72%`, which is a big improvement over the `51%` of the baseline model. 
