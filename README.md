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

    ```contract;
 
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

    ```
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


