# 设计一个电影推荐系统

![Movie recommendation system](https://github.com/WhosthatAoli/OOP-Design/blob/main/images/movie-recommendation/Movie%20recommendation%20system.png)

## 设计背景

#### 一些常问问题：

+ 用户如何给电影打分？
+ 以什么方法给用户推荐电影？
+ 有多少用户和电影？
+ 边际问题：
  + 如果有两个电影符合推荐的条件，选择哪一个？
  + 如果是新用户，没有评价过任何电影，如何进行推荐？

### 基础条件

+ 用户可以给电影打分，范围为1分-5分
+ 推荐算法基于用户相似度，评分相似度等（有很多种推荐算法，这里并不强调算法）
+ 如果两个电影同时符合推荐条件，选择排序靠前的一个
+ 如果用户没有给任何电影评过分，选择得分最高的一个

## OOP设计

### 上层设计

+ 建立**Movie** class封装电影信息
+ 建立**User** class封装用户信息
+ 使用一个**MovieRating** enum 来代表评分
+ 建立一个**RatingRegister** class 用来存储用户对电影的评分
  +  拥有用户对电影评分的字典
  + 拥有电影获得用户评分的字典
  + 具体字典设计见Code
+ 使用一个**MovieRecommender** class来推荐电影
  + 需要传入当前的RatingRegister作为参数
  + 所有推荐计算在此类中完成

## Code

```python
class Movie:
    def __init__(self, id, title):
        self._id = id  #唯一id
        self._title = title

    def getId(self):
        return self._id

    def getTitle(self):
        return self._title

```



```python
class User:
    def __init__(self, id, name):
        self._id = id
        self._name = name

    def getId(self):
        return self._id

```

**MovieRating** 的enum代表评分

```python
from enum import Enum

class MovieRating(Enum):
    NOT_RATED = 0
    ONE = 1
    TWO = 2
    THREE = 3
    FOUR = 4
    FIVE = 5

```

**RatingRegister** class 存储电影评分的信息，注意这里的字典设计，便于后续推荐的处理

```python
class RatingRegister:
    def __init__(self):
        self._userMovies = {}   # Map<UserId, List<Movie>>
        self._movieRatings = {} # Map<MovieId, Map<UserId, Rating>>

        self._movies = []       # List<Movie>
        self._users = []        # List<User>

    def addRating(self, user, movie, rating):
        if movie.getId() not in self._movieRatings:
            self._movieRatings[movie.getId()] = {}
            self._movies.append(movie)
        if user.getId() not in self._userMovies:
            self._userMovies[user.getId()] = []
            self._users.append(user)
        self._userMovies[user.getId()].append(movie)
        self._movieRatings[movie.getId()][user.getId()] = rating

    def getAverageRating(self, movie):
        if movie.getId() not in self._movieRatings:
            return MovieRating.NOT_RATED.value
        ratings = self._movieRatings[movie.getId()].values()
        ratingValues = [rating.value for rating in ratings]
        return sum(ratingValues) / len(ratings)

    def getUsers(self):
        return self._users

    def getMovies(self):
        return self._movies

    def getUserMovies(self, user):
        return self._userMovies.get(user.getId(), [])

    def getMovieRatings(self, movie):
        return self._movieRatings.get(movie.getId(), {})

```



```python
class MovieRecommendation:
    def __init__(self, ratings):
        self._movieRatings = ratings

    def recommendMovie(self, user):
        if len(self._movieRatings.getUserMovies(user)) == 0:
            return self._recommendMovieNewUser()
        else:
            return self._recommendMovieExistingUser(user)

    def _recommendMovieNewUser(self):
        best_movie = None
        best_rating = 0
        for movie in self._movieRatings.getMovies():
            rating = self._movieRatings.getAverageRating(movie)
            if rating > best_rating:
                best_movie = movie
                best_rating = rating
        return best_movie.getTitle() if best_movie else None

    def _recommendMovieExistingUser(self, user): 
        best_movie = None
        similarity_score = float('inf') # Lower is better

        for reviewer in self._movieRatings.getUsers():
            if reviewer.getId() == user.getId():
                continue
            score = self._getSimilarityScore(user, reviewer)  #算法
            if score < similarity_score:
                similarity_score = score
                recommended_movie = self._recommendUnwatchedMovie(user, reviewer)
                best_movie = recommended_movie if recommended_movie else best_movie
        return best_movie.getTitle() if best_movie else None

    def _getSimilarityScore(self, user1, user2):  #推荐算法相关
        user1_id = user1.getId()
        user2_id = user2.getId()
        user2_movies = self._movieRatings.getUserMovies(user2)
        score = float('inf') # Lower is better

        for movie in user2_movies:
            cur_movie_ratings = self._movieRatings.getMovieRatings(movie)
            if user1_id in cur_movie_ratings:
                score = 0 if score == float('inf') else score
                score += abs(cur_movie_ratings[user1_id].value - cur_movie_ratings[user2_id].value)
        return score

    def _recommendUnwatchedMovie(self, user, reviewer):
        user_id = user.getId()
        reviewer_id = reviewer.getId()
        best_movie = None
        best_rating = 0

        reviewer_movies = self._movieRatings.getUserMovies(reviewer)
        for movie in reviewer_movies:
            cur_movie_ratings = self._movieRatings.getMovieRatings(movie)
            if user_id not in cur_movie_ratings and cur_movie_ratings[reviewer_id].value > best_rating:
                best_movie = movie
                best_rating = cur_movie_ratings[reviewer_id].value
        return best_movie

```

下面就可以来进行测试，注意测试的流程

```python
user1 = User(1, 'User 1')
user2 = User(2, 'User 2')
user3 = User(3, 'User 3')

movie1 = Movie(1, 'Batman Begins')
movie2 = Movie(2, 'Liar Liar')
movie3 = Movie(3, 'The Godfather')

ratings = RatingRegister()
ratings.addRating(user1, movie1, MovieRating.FIVE)
ratings.addRating(user1, movie2, MovieRating.TWO)
ratings.addRating(user2, movie2, MovieRating.TWO)
ratings.addRating(user2, movie3, MovieRating.FOUR)

recommender = MovieRecommendation(ratings)

print(recommender.recommendMovie(user1)) # The Godfather
print(recommender.recommendMovie(user2)) # Batman Begins
print(recommender.recommendMovie(user3)) # Batman Begins

```



