# Design a Bank-设计银行系统

![bank](https://github.com/WhosthatAoli/OOP-Design/blob/main/images/Bank/bank.png)

## 设计背景

银行提供多种资产管理服务，包括存取款，信用卡，贷款等。

用户需要拥有银行账户来使用银行提供的服务。用户可以存取款，甚至可以购买理财产品。

## 设计要求

### 常问问题

+ 银行提供何种金融服务？
+ 用户必须要开户吗？银行如何管理用户账户？
+ 银行有地理位置和实际柜员（Teller）吗？
+ 我们是否要担心银行的安全性？银行是否有金库？

### 银行服务

+ 用户可以开户，存款和取款
+ 我们只考虑在实体银行（拥有实际地理位置）通过柜员的交易（Transaction）

### 柜员（Teller）

+ 柜员帮助用户实际执行存取款操作
+ 每一条交易都要记录用户和柜员信息

### 总部与分支（Headquarter and Branch）

+ 每一个分行（branch）在每一天结束时会将钱送到总部（Headquarter）
+ 无需关心金钱如果转移

## OOP设计

### 上层设计

+ 需要建立基础的**交易类（Transaction class）**可以被**存款（Deposit）**，**取款（Withdrawal）**，**开户（OpenAccount）**类继承
+ **柜员类（BackTeller）**需要封装其柜员的id。我们使用**账户（BankAccount）**来替代单纯的用户，封装用户的id和存款
+ 总部**银行（Bank）** 拥有多个**分行（BankBranch）**。建立一个**银行系统（BankSystem）**来负责总的账户信息存储和交易信息存储，因为一个用户可以在不同的分行进行交易，所以信息需要全局保存在银行系统中。

## Code

一个Transaction需要绑定用户和柜员。使用`get_transaction_description`方法给子类重写。设计遵循Open-Closed Principle。通过重写方法来增加交易的类别，而不需要修改Transacion父类。比使用全局的`enum`要更灵活，因为我们还可以在class中封装更多的信息。

```python
from abc import ABC, abstractmethod

class Transaction(ABC):
    def __init__(self, customerId, tellerId):
        self._customerId = customerId
        self._tellerId = tellerId

    def get_customer_id(self):
        return self._customerId

    def get_teller_id(self):
        return self._tellerId

    @abstractmethod
    def get_transaction_description(self):
        pass

```

三种Transaction的子类

```python
class Deposit(Transaction):
    def __init__(self, customerId, tellerId, amount):
        super().__init__(customerId, tellerId)
        self._amount = amount

    def get_transaction_description(self):
        return f'Teller {self.get_teller_id()} deposited {self._amount} to account {self.get_customer_id()}'

class Withdrawal(Transaction):
    def __init__(self, customerId, tellerId, amount):
        super().__init__(customerId, tellerId)
        self._amount = amount

    def get_transaction_description(self):
        return f'Teller {self.get_teller_id()} withdrew {self._amount} from account {self.get_customer_id()}'

class OpenAccount(Transaction):
    def __init__(self, customerId, tellerId):
        super().__init__(customerId, tellerId)

    def get_transaction_description(self):
        return f'Teller {self.get_teller_id()} opened account {self.get_customer_id()}'

```

银行柜员类，包括其id信息

```python
class BankTeller:
    def __init__(self, id):
        self._id = id

    def get_id(self):
        return self._id

```

银行账户类，包括用户信息，存款等，包括存取款函数修改内部数据

```python
class BankAccount:
    def __init__(self, customerId, name, balance):
        self._customerId = customerId
        self._name = name
        self._balance = balance

    def get_balance(self):
        return self._balance

    def deposit(self, amount):
        self._balance += amount

    def withdraw(self, amount):
        self._balance -= amount

```

银行系统（BankSystem）是存储中心，需要存储账户信息，交易信息。方便起见，这里使用数组存储账户

```python
class BankSystem:
    def __init__(self, accounts, transactions):
        self._accounts = accounts
        self._transactions = transactions

    def get_account(self, customerId):
        return self._accounts[customerId]

    def get_accounts(self):
        return self._accounts

    def get_transactions(self):
        return self._transactions

    def open_account(self, customer_name, teller_id):
        # Create account
        customerId = len(self.get_accounts())
        account = BankAccount(customerId, customer_name, 0)
        self._accounts.append(account)

        # Log transaction
        transaction = OpenAccount(customerId, teller_id)
        self._transactions.append(transaction)
        return customerId

    def deposit(self, customer_id, teller_id, amount):
        account = self.get_account(customer_id)
        account.deposit(amount)

        transaction = Deposit(customer_id, teller_id, amount)
        self._transactions.append(transaction)

    def withdraw(self, customer_id, teller_id, amount):
        if amount > self.get_account(customer_id).get_balance():
            raise Exception('Insufficient funds')
        account = self.get_account(customer_id)
        account.withdraw(amount)

        transaction = Withdrawal(customer_id, teller_id, amount)
        self._transactions.append(transaction)

```

银行分支（BankBranch）负责通过柜员来执行每一笔交易。我们也为其增加集中存款到总部（collected money）和总部放款到分行（provided）的方法。

```python
import random

class BankBranch:
    def __init__(self, address, cash_on_hand, bank_system):
        self._address = address
        self._cash_on_hand = cash_on_hand
        self._bank_system = bank_system
        self._tellers = []

    def add_teller(self, teller):
        self._tellers.append(teller)

    def _get_available_teller(self):
        index = round(random.random() * (len(self._tellers) - 1))
        return self._tellers[index].get_id()

    def open_account(self, customer_name):
        if not self._tellers:
            raise ValueError('Branch does not have any tellers')
        teller_id = self._get_available_teller()
        return self._bank_system.open_account(customer_name, teller_id)

    def deposit(self, customer_id, amount):
        if not self._tellers:
            raise ValueError('Branch does not have any tellers')
        teller_id = self._get_available_teller()
        self._bank_system.deposit(customer_id, teller_id, amount)

    def withdraw(self, customer_id, amount):
        if amount > self._cash_on_hand:
            raise ValueError('Branch does not have enough cash')
        if not self._tellers:
            raise ValueError('Branch does not have any tellers')
        self._cash_on_hand -= amount
        teller_id = self._get_available_teller()
        self._bank_system.withdraw(customer_id, teller_id, amount)

    def collect_cash(self, ratio):
        cash_to_collect = round(self._cash_on_hand * ratio)
        self._cash_on_hand -= cash_to_collect
        return cash_to_collect

    def provide_cash(self, amount):
        self._cash_on_hand += amount

```

总行Bank类负责管理所有的分行，包括集中收取款，放款，打印所有交易信息。

```python
class Bank:
    def __init__(self, branches, bank_system, total_cash):
        self._branches = branches
        self._bank_system = bank_system
        self._total_cash = total_cash

    def add_branch(self, address, initial_funds):
        branch = BankBranch(address, initial_funds, self._bank_system)
        self._branches.append(branch)
        return branch

    def collect_cash(self, ratio):
        for branch in self._branches:
            cash_collected = branch.collect_cash(ratio)
            self._total_cash += cash_collected

    def print_transactions(self):
        for transaction in self._bank_system.get_transactions():
            print(transaction.get_transaction_description())

```

这样就完成了Bank System的设计，让我们来测试一下

```python
bankSystem = BankSystem([], [])
bank = Bank([], bankSystem, 10000)

branch1 = bank.add_branch('123 Main St', 1000)
branch2 = bank.add_branch('456 Elm St', 1000)

branch1.add_teller(BankTeller(1))
branch1.add_teller(BankTeller(2))
branch2.add_teller(BankTeller(3))
branch2.add_teller(BankTeller(4))

customerId1 = branch1.open_account('John Doe')
customerId2 = branch1.open_account('Bob Smith')
customerId3 = branch2.open_account('Jane Doe')

branch1.deposit(customerId1, 100)
branch1.deposit(customerId2, 200)
branch2.deposit(customerId3, 300)

branch1.withdraw(customerId1, 50)
""" Possible Output:
    Teller 1 opened account 0
    Teller 2 opened account 1
    Teller 3 opened account 2
    Teller 1 deposited 100 to account 0
    Teller 2 deposited 200 to account 1
    Teller 4 deposited 300 to account 2
    Teller 2 withdrew 50 from account 0
"""

bank.print_transactions()

```





