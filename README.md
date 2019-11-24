# Recommendation System for Steam Game Store: An overview of recommender systems

## Team:<br/>
Doo Hyodan, Department Of Information System, Hanyang University, ammissyouyou@gmail.com<br/>
Audrey Germain, Computer Engineering, Department of Software Engineering and Computer Engineering, Polytechnique Montreal<br/>
Geordan Jove, Aerospace Engineering, Department of Mechanical Engineering, Polytechnique Montreal<br/>

## I. Introduction
Like many young people, all member of this team have an interest in video games, more particularly in computer games.
Therefore, this project has for goal to build a recommender system for computer game.<br/>

For this project we are using the data from the Steam the biggest video game digital distribution service for computer games.
We will be using some user data and game data, the datasets used will be explained in detail further in this blog.<br/>

The primary focus of this project is to build a recommendation system that will recommend games for each user based on
their preferences and gaming habits. In order to make our recommendation system the best possible, multiple algorithms
will be implemented and compared to make the recommendation the most relevant possible for each users.<br/>

## II. Datasets
For this project, 2 differents datasets are used. Both dataset are available for free on kaggle and are extracted from Steam.<br/>

The first dataset is the user dataset. It contains the user id, the game, the behavior and the amount of hours played.
So each row of the dataset represent the behavior (play or purchase) of a user towards a game.
The amount of hours played is also specify, the column contains 1 if it's a purchase.
The dataset contains a total of 200000 rows, including 5155 different games and 12393 different users.
To create a training and testing dataset, we started by combining the information about play and purchase in a single row,
in this new form, the columns are user ID, name of the game, amount of hours of play time, play (0 if never played
and 1 if played) and purchase (technically always 1), this created a total of 128804 rows. Then we extracted 20% of
all the rows (25761 rows) for the test dataset and kept the rest  (103043 rows) for the training dataset.<br/>

The second dataset contains a list of games and their descriptions. It contains the url (directed to Steam store),
the type of package (app, bundle…), the name of the game, a short description, recent reviews, all reviews, release date,
developper, publisher, popular tags (Gore, Action, Shooter, PvP…), game detail (Multi-player, Single-player, Full controller support…),
languages, achievements, genre (Action, Adventure, RPG, Strategy…), game description, description of mature content,
minimum requirement to run the game, recommended requirement, original price and price with discount.
There is a total of 51920 games in the dataset.<br/>

## III. Methodology

We decided to use 3 differents algorithms to generate recommendation by user. We use 2 collaborative algorithm,
one using the ALS and one using the EM algorithm and we use one content-based algorithm.<br/>

### Collaborative recommender with ALS

### Collaborative recommender with EM

### Content-based recommender

To generate the recommendation for each game, the following function is used. The input of the function is the title of
the game as a string and the cosine matrix (explained later) and the output is a list of recommended game title.<br/>

```python
def get_recommendations(title, cosine_sim):

	if title not in listGames:
    	return []

	# Get the index of the game that matches the name
	idx = indices[title]

	# if there's 2 games or more with same name (game RUSH)
	if type(idx) is Series:
    	return []

	# Get the pairwise similarity scores of all games with that game
	sim_scores = list(enumerate(cosine_sim[idx]))

	# Sort the games based on the similarity scores
	sim_scores = sorted(sim_scores, key=lambda x: x[1], reverse=True)

	# Get the scores of the most similar games
	# (not the first one because this games as a score of 1 (perfect score) similarity with itself)
	sim_scores = sim_scores[1:n_recommendation + 1]

	# Get the games indices
	movie_indices = [i[0] for i in sim_scores]

	# Return the top most similar games
	return dataGames['name'].iloc[movie_indices].tolist()

```
The variable listGames is a list of all the games that are in both of the dataset (user and game dataset).
We use this because there is a lot of games in the game dataset that have never been played or purchased by any user,
so there's no use in considering them in the recommender and some of the games in the user dataset are not in the game dataset.
To maximize the amount of match between the game titles in the datasets we removed all symboles and spaces and put
every letters in lower case. We were able to find 3036 games in the game dataset that match some of the 5151 games that are in the user dataset.<br/>

The variable indices is a reverse map that use the name as key to get the index of each game in the cosine similarity matrix.
We make sure that the idx is not a Series, it can happen in the case where 2 different games have the same name (in our dataset 2 games have the name "RUSH").<br/>
```python
# Construct a reverse map of indices and game names
indices = Series(dataGames.index, index=dataGames['name']).drop_duplicates()
```
Then we get the similarity score of each games from the matrix and we order it from the most similar to the less similar.
Finally we just need to extract the amount of recommendation that we want and return the list.
The variable n_recommendation contains the amount of recommendation we want to get, we decided to set it to 20.<br/>

To generate the cosine similarity matrix we use the following code. First it calculate the matrix of frequency of each
words in the popular tag of each of the games, then it calculate the cosine similarity function.<br/>

```python
# Compute the Cosine Similarity matrix using the popular tags column
count = CountVectorizer(stop_words='english')
count_matrix_popular_tags = count.fit_transform(dataGames['popular_tags'])
cosine_sim_matrix_popular_tags = cosine_similarity(count_matrix_popular_tags, count_matrix_popular_tags)
```
To get the recommendation for each user, we implemented a function that combines the recommendations and get the
recommendation with the best reviews (extracted from the game dataset). This function takes the ID of each user,
the list of recommendation (the recommendation function explained previously is applied to all the games a user
has and a list of all the recommendations is made) and the list of all the game the user already has. The function
return a dataframe containing the user ID in the first column and then 20 column with the top recommendations.<br/>

```python
def make_recommendation_for_user(user_id, game_list, game_user_have):
	if type(game_list) is not list or len(game_list) == 0:
    	# return empty one
    	return DataFrame(data=[[user_id] + [""] * n_recommendation], columns=col_names)

	# get reviews of game recommendation, remove the games the user already has and order them by reviews
	recommendation_reviews = dataReviews.loc[dataReviews['name'].isin(game_list)]
	recommendation_reviews = recommendation_reviews.loc[~recommendation_reviews['name'].isin(game_user_have)]
	recommendation_reviews = recommendation_reviews.sort_values(by="percentage_positive_review", ascending=False)

	if len(recommendation_reviews.index) < n_recommendation:
    	return DataFrame(data=[[user_id] + recommendation_reviews["name"].tolist() +
                           	[""] * (n_recommendation - len(recommendation_reviews.index))],
                     	columns=col_names)
	else:
    	return DataFrame(data=[[user_id] + recommendation_reviews["name"].tolist()[0:n_recommendation]],
                     	columns=col_names)
```
First, is the list of recommendation is empty (can happen if none of the game the user has are in the game dataset)
or not valid, a dataframe without recommendation is returned. If there is no problem with the recommendation list,
we get a dataframe of the name of the recommended games and the review (percentage of positive review) and we remove
the games that the user already has (no need to recommend a game the user already has). Then the recommendation are
ordered from the best review to the worst. If there is less recommendations then needed, empty spaces fill the rest of the column.<br/>

All the dataframe rows produced by this functions are combine and are printed in a CSV file.<br/>

## IV. Evaluation & Analysis

## V. Related Work

For the content-based recommender system we used the some code from the blog post
[Recommender Systems in Python: Beginner Tutorial](https://www.datacamp.com/community/tutorials/recommender-systems-python?fbclid=IwAR1fz9YLOgZ95KHwoLpgb-hTdV2MekujDGBngRTG3kYmBJYxwSK3UWvNJDg)
to implement the function that give the recommendation for each games.<br/>

## VI. Conclusion: Discussion

