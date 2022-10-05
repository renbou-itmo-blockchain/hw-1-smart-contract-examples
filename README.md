# ДЗ 1. Введение в смарт-контракты
Просмотреть код контрактов заданного преподавателем проекта, найти место, реализующее определенную логику и привести diff (результат работы git diff), изменяющий поведение смарт-контракта.

## MultiSigWallet
### Задание
Сделать, чтобы с баланса multisig-контракта за одну транзакцию не могло бы уйти больше, чем 66 ETH.  
Изначальный код: [github.com/gnosis/MultiSigWallet/blob/master/contracts/MultiSigWallet.sol](github.com/gnosis/MultiSigWallet/blob/master/contracts/MultiSigWallet.sol)
### Решение
```diff
--- a/original/MultiSigWallet.sol
+++ b/modified/MultiSigWallet.sol
@@ -22,6 +22,7 @@ contract MultiSigWallet {
      *  Constants
      */
     uint constant public MAX_OWNER_COUNT = 50;
+    uint constant public MAX_TRANSACTION_VALUE = 66 ether;
 
     /*
      *  Storage
@@ -91,6 +92,11 @@ contract MultiSigWallet {
         _;
     }
 
+    modifier validValue(uint value) {
+        require(value <= MAX_TRANSACTION_VALUE);
+        _;
+    }
+
     /// @dev Fallback function allows to deposit ether.
     function()
         payable
@@ -188,6 +194,7 @@ contract MultiSigWallet {
     /// @return Returns transaction ID.
     function submitTransaction(address destination, uint value, bytes data)
         public
+        validValue(value)
         returns (uint transactionId)
     {
         transactionId = addTransaction(destination, value, data);
```

Так как смарт-контракт разделяет логику проведения транзакций на создание, подтверждение и исполнение, то для предотвращения транзакций суммой больше 66 ETH необходимо всего лишь проверять это условие при создания транзакций. Для этого создадим модификатор, валидирующий сумму транзакции и воспользуемся им в функции `submitTransaction`, которая создает транзакцию.

## ERC20
### Задание
Сделать, чтобы токен не мог бы быть transferred по субботам.
### Решение
```diff
--- a/original/task-2/ERC20.sol
+++ b/modified/task-2/ERC20.sol
@@ -211,6 +211,12 @@ contract ERC20 is Context, IERC20 {
 
         _beforeTokenTransfer(sender, recipient, amount);
 
+        // Unix Epoch is Thursday, 1 January 1970, so by default daysSinceEpoch % 7 == 0 on Thursday
+        // Adding 3 to that gets us 0 on Monday and 6 on Sunday 
+        uint daysSinceEpoch = block.timestamp / (60 * 60 * 24);
+        uint dayOfWeek = (daysSinceEpoch + 3) % 7;
+        require(dayOfWeek != 5, "ERC20: cannot transfer on Saturdays");
+
         require(_balances[sender] >= amount, "ERC20: transfer amount exceeds balance");
         _balances[sender] -= amount;
         _balances[recipient] += amount;
```

Воспользуемся глобальным значением `block`, в котором доступен `timestamp` (timestamp этого блока) - посчитаем кол-во прошедших дней с начала Unix Epoch, и вычислим сегодняшний день недели на этом основании. Так как все операции трансфера пользуются функцией `_transfer`, в которой мы проверяем требование, то они автоматически будут работать правильно.

## DividendToken
### Задание
Сделать чтобы платеж в ETH принимался только специальной функцией, принимающей помимо ETH еще комментарий к платежу (bytes32). Простая отправка ETH в контракт запрещена.
### Решение
```diff
--- a/original/task-3/DividendToken.sol
+++ b/modified/task-3/DividendToken.sol
@@ -18,7 +18,7 @@ contract DividendToken is StandardToken, Ownable {
     event PayDividend(address indexed to, uint256 amount);
     event HangingDividend(address indexed to, uint256 amount) ;
     event PayHangingDividend(uint256 amount) ;
-    event Deposit(address indexed sender, uint256 value);
+    event Deposit(address indexed sender, uint256 value, bytes32 comment);
 
     /// @dev parameters of an extra token emission
     struct EmissionInfo {
@@ -37,9 +37,9 @@ contract DividendToken is StandardToken, Ownable {
         }));
     }
 
-    function() external payable {
+    function deposit(bytes32 comment) external payable {
         if (msg.value > 0) {
-            emit Deposit(msg.sender, msg.value);
+            emit Deposit(msg.sender, msg.value, comment);
             m_totalDividends = m_totalDividends.add(msg.value);
         }
     }
```

Просто создадим функцию приёма платежа, в которую также передается комментарий. Чтобы комментарий был полезен, будем передавать его в событие `Deposit`. Добавить аргумент комментария к функции без имени (fallback) нельзя, поэтому и приходится создавать отдельную функцию.