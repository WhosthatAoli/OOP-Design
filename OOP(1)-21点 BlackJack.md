# 设计21点游戏

## 背景与游戏规则

21点-Blackjack是一个很流行的牌类游戏。目标是让自己手上的牌越接近21点越好，但是不能超过21点。

玩家Player可以选择给自己发两张牌或者更多。庄家Dealer同样抽两张牌，一张是盖牌。玩家比庄家的牌大，玩家胜利。但是如果超过21点，自动判负。双方手牌同样大小则是平局。Ace可以自己选择为1分或11分。

![ood-1](https://github.com/WhosthatAoli/OOP--/blob/main/images/blackjack/ood-1.png)

## 游戏条件

面试不是考察你对于游戏的理解程度，所以会给出特定的游戏条件。这里我们以下面的游戏条件为标准

### 基础条件

+ 两个玩家，其中一个作为庄家
+ 一副牌，每一轮重新洗牌

### 牌

+ 一副牌52张牌，四种花色
+ 每种花色有13张

### 游戏顺序

+ 游戏开始，庄家和玩家都抽两张牌，庄家有一张牌为盖牌（玩家看不到）
+ 玩家手牌和在21点以下，可以选择继续抽牌，如果抽牌后超过21点，自动判负，本轮游戏结束
+ 庄家的手牌和若比玩家小，继续抽牌，如果抽牌后超过21点，自动判负，本轮游戏结束
+ 最终，手牌和大的一方胜利

### 下注

+ 玩家可以下注任意数值注码，只要不超过其拥有的数值
+ 庄家会自动匹配玩家下注的注码

## OOP设计

### 上层设计

+ 每一张**牌card**有花色与数值
  + 花色在blackjack游戏中不重要，但是如果游戏有UI，需要考虑花色
  + 同样的，ten，jack，queen，king这四张卡有同样的数值，所以不需要进行区分
+ **一副牌（deck）**拥有52张牌，作为数组存在
  + Deck需要负责洗牌（shuffling）与发牌（popping，drawing cards）
+ **玩家（Player）**
  + 可以是庄家（Dealer）或者普通玩家（User），作为抽象类使用
  + 普通玩家需要选择行动，抽牌或者停止抽牌
  + 普通玩家有积分余额（Balance）且可以进行下注（Bet）

## Code具体实现

花色有四种不同，可以选择`Enum`去定义

```python
from enum import Enum

class Suit(Enum):
    CLUBS, DIAMONDS, HEARTS, SPADES = 'clubs', 'diamonds', 'hearts', 'spades'

```

一张牌有花色与数值，下面创建牌的class：

```python
class Card:
    def __init__(self, suit, value):
        self._suit = suit
        self._value = value
    
    def getSuit(self):
        return self._suit

    def getValue(self):
        return self._value

    def print(self):
        print(self.getSuit(), self.getValue())

```

一副手牌包括牌，点数和，有抽牌功能，下面创建手牌的class：

```python
class Hand:
    def __init__(self):
        self._score = 0
        self._cards = []

    def addCard(self, card):
        self._cards.append(card)
        if card.getValue() == 1:
            self._score += 11 if self._score + 11 <= 21 else 1
        else:
            self._score += card.getValue()
        print('Score: ', self._score)

    def getScore(self):
        return self._score

    def getCards(self):
        return self._cards

    def print(self):
        for card in self.getCards():
            print(card.getSuit(), card.getValue())

```

一副牌有一系列的牌，有洗牌与发牌的功能，下面创建一副牌（deck）的class：

```python
import random

class Deck:
    def __init__(self):
        self._cards = []
        for suit in Suit:  #四种花色
            for value in range(1, 14): #1到13
                self._cards.append(Card(suit, min(value, 10))) #blackjack中ten以上value都为10

    def print(self):
        for card in self._cards:
            card.print()

    def draw(self):
        return self._cards.pop()

    def shuffle(self):
        for i in range(len(self._cards)):
            j = random.randint(0, 51)
            self._cards[i], self._cards[j] = self._cards[j], self._cards[i]

```



下面创建玩家（Player）的抽象类，后面再创建庄家Dealer和普通玩家UserPlayer的extend class去override `makeMove()` 函数

```python
from abc import ABC, abstractmethod

class Player(ABC):
    def __init__(self, hand):
        self._hand = hand

    def getHand(self):
        return self._hand

    def clearHand(self):
        self._hand = Hand()

    def addCard(self, card):
        self._hand.addCard(card)

    @abstractmethod
    def makeMove(self):
        pass

```



UserPlayer Class，拥有余额且可以进行下注

```python
class UserPlayer(Player):
    def __init__(self, balance, hand):
        super().__init__(hand)
        self._balance = balance

    def getBalance(self):
        return self._balance

    def placeBet(self, amount):
        if amount > self._balance:
            raise ValueError('Insufficient funds')
        self._balance -= amount
        return amount

    def receiveWinnings(self, amount):
        self._balance += amount

    def makeMove(self):
        if self.getHand().getScore() > 21:
            return False
        move = input('Draw card? [y/n] ')
        return move == 'y'

```

Dealer Class:

```python
class Dealer(Player):
    def __init__(self, hand):
        super().__init__(hand)
        self._targetScore = 17

    def updateTargetScore(self, score):
        self._targetScore = score

    def makeMove(self):
        return self.getHand().getScore() < self._targetScore

```



GameRound，相当于Game Controller

```python
class GameRound:
    def __init__(self, player, dealer, deck):
        self._player = player
        self._dealer = dealer
        self._deck = deck

    def getBetUser(self):
        amount = int(input('Enter a bet amount: '))
        return amount

    def dealInitialCards(self):
        for i in range(2):
            self._player.addCard(self._deck.draw())
            self._dealer.addCard(self._deck.draw())
        print('Player hand: ')
        self._player.getHand().print()
        dealerCard = self._dealer.getHand().getCards()[0]
        print("Dealer's first card: ")
        dealerCard.print()

    def cleanupRound(self):
        self._player.clearHand()
        self._dealer.clearHand()
        print('Player balance: ', self._player.getBalance())

    def play(self):
        self._deck.shuffle()

        if self._player.getBalance() <= 0:
            print('Player has no more money =)')
            return
        userBet = self.getBetUser()
        self._player.placeBet(userBet)

        self.dealInitialCards()

        # User makes moves
        while self._player.makeMove():
            drawnCard = self._deck.draw()
            print('Player draws', drawnCard.getSuit(), drawnCard.getValue())
            self._player.addCard(drawnCard)
            print('Player score: ', self._player.getHand().getScore())

        if self._player.getHand().getScore() > 21:
            print('Player busts!')
            self.cleanupRound()
            return
        
        # Dealer makes moves
        while self._dealer.makeMove():
            self._dealer.addCard(self._deck.draw())
        
        # Determine winner
        if self._dealer.getHand().getScore() > 21:
            print('Player wins')
            self._player.receiveWinnings(userBet * 2)
        elif self._dealer.getHand().getScore() > self._player.getHand().getScore():
            print('Player loses')
        else:
            print('Game ends in a draw')
            self._player.receiveWinnings(userBet)
        self.cleanupRound()

```



以上我们就实现了简单的21点游戏，可以进行测试

```python
player = UserPlayer(1000, Hand())
dealer = Dealer(Hand())

while player.getBalance() > 0:
    gameRound = GameRound(player, dealer, Deck()).play()

```

