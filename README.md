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
========================


### Cоздание интерфейса для взаимодействия с вашим контрактом!

Для начала установим fuel wallet
https://chromewebstore.google.com/detail/fuel-wallet/dldjpboieedgcmpkchcjcbijingjcgok

После настройки кошелька нажмите кнопку «Кран» в кошельке, чтобы получить токены тестовой сети.

Вернемся на директорию выше и инициализируем проект React с помощью TypeScript:


    cd ..
    npx create-react-app frontend --template typescript

Полученный вывод будет таким:

    Success! Created frontend at Fuel/fuel-project/frontend

Установим необходимые зависимости:

    cd frontend
    npm install fuels @fuels/react @fuels/connectors @tanstack/react-query

Создадим файл конфигурации:

    npx fuels init --contracts ../counter-contract/ --output ./src/sway-api
    npx fuels build

Внутри папки fuel-project/frontend/src добавим код, который взаимодействует с нашим контрактом.
Изменим код fuel-project/frontend/src/index.tsx, он будет таким:

```ruby
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';
import { FuelProvider } from '@fuels/react';
import {
  defaultConnectors,
} from '@fuels/connectors';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);
root.render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <FuelProvider
        fuelConfig={{
          connectors: defaultConnectors(),
        }}
      >
        <App />
      </FuelProvider>
    </QueryClientProvider>
  </React.StrictMode>
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

Затем изменим файл fuel-project/frontend/src/App.tsx, предвариельно указав свой CONTRACT_ID в "0x...":

```ruby
import { useEffect, useState } from "react";
import {
  useBalance,
  useConnectUI,
  useIsConnected,
  useWallet
} from '@fuels/react';
import { CounterContractAbi__factory  } from "./sway-api"
import type { CounterContractAbi } from "./sway-api";

// REPLACE WITH YOUR CONTRACT ID
const CONTRACT_ID = 
  "0x...";

export default function Home() {
  const [contract, setContract] = useState<CounterContractAbi>();
  const [counter, setCounter] = useState<number>();
  const { connect, isConnecting } = useConnectUI();
  const { isConnected } = useIsConnected();
  const { wallet } = useWallet();
  const { balance } = useBalance({
    address: wallet?.address.toAddress(),
    assetId: wallet?.provider.getBaseAssetId(),
  });

  useEffect(() => {
    async function getInitialCount(){
      if(isConnected && wallet){
        const counterContract = CounterContractAbi__factory.connect(CONTRACT_ID, wallet);
        await getCount(counterContract);
        setContract(counterContract);
      }
    }
    
    getInitialCount();
  }, [isConnected, wallet]);

  const getCount = async (counterContract: CounterContractAbi) => {
    try{
      const { value } = await counterContract.functions
      .count()
      .get();
      setCounter(value.toNumber());
    } catch(error) {
      console.error(error);
    }
  }

  const onIncrementPressed = async () => {
    if (!contract) {
      return alert("Contract not loaded");
    }
    try {
      await contract.functions
      .increment()
      .call();
      await getCount(contract);
    } catch(error) {
      console.error(error);
    }
  };

  return (
    <div style={styles.root}>
      <div style={styles.container}>
        {isConnected ? (
          <>
            <h3 style={styles.label}>Counter</h3>
            <div style={styles.counter}>
              {counter ?? 0}
            </div>

            {balance && balance.toNumber() === 0 ? (
              <p>Get testnet funds from the <a target="_blank" rel="noopener noreferrer"  href={`https://faucet-testnet.fuel.network/?address=${wallet?.address.toAddress()}`}>Fuel Faucet</a> to increment the counter.</p>
          ) : 
          (
            <button
            onClick={onIncrementPressed}
            style={styles.button}
            >
              Increment Counter
            </button>
          )
          }
            
          <p>Your Fuel Wallet address is:</p>
          <p>{wallet?.address.toAddress()}</p>
          </>
        ) : (
          <button
          onClick={() => {
            connect();
          }}
          style={styles.button}
          >
            {isConnecting ? 'Connecting' : 'Connect'}
          </button>
        )}
      </div>
    </div>
  );
}

const styles = {
  root: {
    display: 'grid',
    placeItems: 'center',
    height: '100vh',
    width: '100vw',
    backgroundColor: "black",
  } as React.CSSProperties,
  container: {
    color: "#ffffffec",
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
  } as React.CSSProperties,
  label: {
    fontSize: "28px",
  },
  counter: {
    color: "#a0a0a0",
    fontSize: "48px",
  },
  button: {
    borderRadius: "8px",
    margin: "24px 0px",
    backgroundColor: "#707070",
    fontSize: "16px",
    color: "#ffffffec",
    border: "none",
    outline: "none",
    height: "60px",
    padding: "0 1rem",
    cursor: "pointer"
  },
}
```

### Запускаем проект
========================

Внутри каталога fuel-project/frontend запускаем:

    npm start

Получаем результат:

```ruby
Compiled successfully!

You can now view frontend in the browser.

  Local:            http://localhost:3000
  On Your Network:  http://192.168.4.48:3000

Note that the development build is not optimized.
To create a production build, use npm run build.
```
В браузере подключаем свой кошелек Fuel и если у вас есть тестовая сеть ETH на Fuel, вы должны увидеть значение счетчика и кнопку увеличения.

## Вы только что создали полнофункциональное децентрализованное приложение на Fuel! ⛽





    
    
    
    

    
