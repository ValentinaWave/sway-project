#  Смарт-контракт в Sway & Counter Dapp

Это руководство содержит пошаговые инструкции о том, как
- написать смарт-контракт в Sway;
- написать тест в Rust;
- развернуть в тестовой сети Fuel;
- создать интерфейс;
- интегрировать кошелек.
  ***

### Смарт-контракт в Sway

#### Установка
========================

Чтобы установить набор инструментов Fuel, вы можете использовать скрипт fuelup-init. 
Это установит forc, forc-client, forc-fmt, forc-lsp, forc-wallet, а также fuel-core в ~/.fuelup/bin.

    curl https://install.fuel.network | sh

Проверить версии необходимых инструментов вы можете с помощью приведенных ниже команд:

    fuelup self update
    fuelup update
    fuelup default latest
  
#### Создание проекта
========================

Мы создадим простой контракт счетчика с двумя функциями: одна для увеличения счетчика и одна для возврата значения счетчика.
Для начала создадим новую пустую папку под названием fuel-project:

    mkdir fuel-project

Переместимся в папку проекта и создадим проект контракта, используя forc:

    cd fuel-project
    forc new counter-contract

Откроем свой проект в редакторе кода ./counter-contract/src/main.sw, удалим все в src/main.sw и создадим новый код:

```ruby
contract;
 
storage {
    counter: u64 = 0,
}
 
abi Counter {
    #[storage(read, write)]
    fn increment();
 
    #[storage(read)]
    fn count() -> u64;
}
 
impl Counter for Contract {
    #[storage(read)]
    fn count() -> u64 {
        storage.counter.read()
    }
 
    #[storage(read, write)]
    fn increment() {
        let incremented = storage.counter.read() + 1;
        storage.counter.write(incremented);
    }
}
```


Перейдем в папку нашего контракта и выполним команду, чтобы создать контракт:

    cd counter-contract
    forc build

#### Тестирование контракта с помощью Rust
========================

Если у вас еще не установлен Rust, вы можете установить его, выполнив следующую команду:

    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

Далее установим cargo generate:

    cargo install cargo-generate --locked

Теперь cгенерируем тестовую программу по умолчанию с помощью следующей команды:

    cargo generate --init fuellabs/sway templates/sway-test-rs --name counter-contract

>Откройте файл Cargo.toml и проверьте версию топлива, используемого в зависимости от разработки. Измените версию на 0.62.0, если это еще не так.

Вот так должен выглядеть ваш файл ./counter-contract/tests/harness.rs

```ruby
use fuels::{prelude::*, types::ContractId};
 
// Load abi from json
abigen!(Contract(
    name = "MyContract",
    abi = "out/debug/counter-contract-abi.json"
));
 
async fn get_contract_instance() -> (MyContract<WalletUnlocked>, ContractId) {
    // Launch a local network and deploy the contract
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(1),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let wallet = wallets.pop().unwrap();
 
    let id = Contract::load_from(
        "./out/debug/counter-contract.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();
 
    let instance = MyContract::new(id.clone(), wallet);
 
    (instance, id.into())
}
 
#[tokio::test]
async fn can_get_contract_id() {
    let (_instance, _id) = get_contract_instance().await;
 
    // Now you have an instance of your contract you can use to test each function
}
 
#[tokio::test]
async fn test_increment() {
    let (instance, _id) = get_contract_instance().await;
 
    // Increment the counter
    instance.methods().increment().call().await.unwrap();
 
    // Get the current value of the counter
    let result = instance.methods().count().call().await.unwrap();
 
    // Check that the current value of the counter is 1.
    // Recall that the initial value of the counter was 0.
    assert_eq!(result.value, 1);
}
```


Запускаем cargo test в терминале:

    cargo test

Если все пойдет хорошо, результат должен выглядеть следующим образом:

  ```
...
  running 2 tests
  test can_get_contract_id ... ok
  test test_increment ... ok
  test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.25s
```

Пришло время развертывания смарт-контракта, сделаем это с помощью forc из командной строки. 
Чтобы развернуть контракт, вам необходимо иметь кошелек для подписи транзакции и монеты для оплаты газа.
Если у вас уже есть fuel кошелек, то импортируйте его командой:

    forc wallet import

Если неи, то создайте новый кошелек (не забудьте записать пароль и мнемоническую фразу от кошелька):

    forc wallet new

Затем создаем новую учетную запись кошелька с помощью:
   
    forc wallet account new

Если вам нужно составить список своих учетных записей, вы можете запустить команду ниже:

    forc wallet accounts

Вы можете получить тестовые средства с помощью крана https://faucet-testnet.fuel.network/

Теперь вы можете развернуть контракт в последней тестовой сети с помощью команды:

    forc deploy --testnet

Укажите пароль вашего кошелька, затем введите номер предпочтительного счета и нажмите Y, когда будет предложено принять транзакцию.
Готово! 
>Сохраните Contract ID, так как он понадобится вам позже для подключения внешнего интерфейса.

```ruby
Contract counter-contract Deployed!

Network: https://testnet.fuel.network
Contract ID: 0x8342d413de2a678245d9ee39f020795800c7e6a4ac5ff7daae275f533dc05e08
Deployed in block 0x4ea52b6652836c499e44b7e42f7c22d1ed1f03cf90a1d94cd0113b9023dfa636
```

## Поздравляем, вы завершили свой первый смарт-контракт на Fuel ⛽

Далее вас ожидает создание интерфейса для взаимодействия с вашим контрактом!


    
