# Введение в смарт-контракты
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