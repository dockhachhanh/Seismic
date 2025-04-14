# Hướng Dẫn Cài Đặt và Chạy Dự Án Walnut App
Dự án walnut-app là một ứng dụng blockchain sử dụng hợp đồng thông minh Walnut.sol trên Seismic Devnet để mô phỏng trò chơi "lắc và đập" quả óc chó. Người chơi (Alice và Bob) thực hiện các hành động shake, hit, reset, và look, với mục tiêu đạt output:
- Round 1: Alice thực hiện shake và hit, cuối cùng gọi look để thấy kernel = 7n.
- Round 2: Bob thực hiện reset, shake, hit, và look để thấy kernel = 3n.
- Access Control: Alice gọi look trong round của Bob phải báo lỗi NOT_A_CONTRIBUTOR.

Hướng dẫn này sẽ giúp bạn cài đặt từ đầu và chạy dự án thành công.
## Yêu cầu trước khi bắt đầu
- Máy tính: Linux (Ubuntu khuyến nghị) hoặc macOS.
- Kết nối internet: Để tải gói và tương tác với Seismic Devnet.
- Kiến thức cơ bản: Biết cách dùng terminal và chỉnh sửa file văn bản.
- Ví ETH: Bạn sẽ cần 3 ví với ít nhất 0.01 ETH mỗi ví trên Seismic Devnet.
---
## Bước 1: Cài đặt môi trường
### 1. Cài Node.js và Bun:
Node.js: Dùng để chạy npm (nếu cần).
```bash
sudo apt update
sudo apt install -y curl
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # Kiểm tra, nên là v18.x hoặc cao hơn
```
Bun: Trình chạy JavaScript cho CLI
```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun --version  # Kiểm tra, nên là v1.2.9 hoặc cao hơn
```
### 2. Cài Foundry (sforge):
Foundry là công cụ biên dịch và triển khai hợp đồng Solidity.
```bash
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc
foundryup
forge --version  # Kiểm tra, nên là phiên bản mới nhất
```
### 3. Cài git:
Dùng để tạo thư mục dự án.
```bash
sudo apt install -y git
git --version
```
## Bước 2: Thiết lập ví và lấy ETH
###1. Tạo 3 ví Ethereum:
Bạn cần 3 khóa riêng (private key) cho:
- Ví triển khai hợp đồng (PRIVKEY).
- Ví của Alice (ALICE_PRIVKEY).
- Ví của Bob (BOB_PRIVKEY).

Lưu các khóa này vào file an toàn, ví dụ walnut-keys.txt. Không chia sẻ khóa riêng!

Lấy ETH trên Seismic Devnet:
Truy cập faucet của Seismic Devnet: https://faucet-2.seismicdev.net/

Nhập từng địa chỉ ví (lấy từ khóa riêng bằng công cụ như ethers.js hoặc MetaMask).

Yêu cầu 0.1 ETH cho mỗi ví.

Kiểm tra số dư:

https://explorer-2.seismicdev.net/address/<địa-chỉ-ví>

Đảm bảo mỗi ví có ít nhất 0.01 ETH.

Ví dụ địa chỉ từ log:
Alice: 0x7BDC905f3D8d250a6d15a68E5D2efeE0527F3331

Bob: 0x889FF238475cAA63eFf42C595a052bcF1eBd9226

Nếu dùng ví cũ, kiểm tra số dư tại:

https://explorer-2.seismicdev.net/address/0x7BDC905f3D8d250a6d15a68E5D2efeE0527F3331
https://explorer-2.seismicdev.net/address/0x889FF238475cAA63eFf42C595a052bcF1eBd9226

## Bước 3: Tạo cấu trúc dự án
### Tạo thư mục dự án:
```bash
mkdir ~/walnut-app
cd ~/walnut-app
```
### Tạo thư mục con:
```bash
mkdir -p packages/cli/src packages/cli/lib packages/contracts/src packages/contracts/script
```
### Khởi tạo package.json
#### CLI
```bash
cd ~/walnut-app/packages/cli
bun init -y
```
Chỉnh sửa packages/cli/package.json
```json
{
  "name": "walnut-cli",
  "license": "MIT License",
  "type": "module",
  "scripts": {
    "dev": "bun run src/index.ts"
  },
  "dependencies": {
    "dotenv": "^16.4.7",
    "viem": "^2.22.3",
    "seismic-viem": "1.0.9"
  },
  "devDependencies": {
    "@types/node": "^22.7.6",
    "typescript": "^5.6.3"
  }
}
```
Contracts
```bash
cd ~/walnut-app/packages/contracts
forge init
```
### Xóa file mẫu src/Counter.sol và script/Counter.s.sol.
## Bước 4: Tạo file mã nguồn
### Hợp đồng Walnut.sol:
File: packages/contracts/src/Walnut.sol
Nội dung
```solidity
// SPDX-License-Identifier: MIT License
pragma solidity ^0.8.13;

contract Walnut {
    uint256 initialShellStrength;
    uint256 shellStrength;
    uint256 round;
    suint256 initialKernel;
    suint256 kernel;
    mapping(uint256 => mapping(address => uint256)) hitsPerRound;

    event Hit(uint256 indexed round, address indexed hitter, uint256 remaining);
    event Shake(uint256 indexed round, address indexed shaker);
    event Reset(uint256 indexed newRound, uint256 shellStrength);

    constructor(uint256 _shellStrength, suint256 _kernel) {
        initialShellStrength = _shellStrength;
        shellStrength = _shellStrength;
        initialKernel = _kernel;
        kernel = _kernel;
        round = 1;
    }

    function getShellStrength() public view returns (uint256) {
        return shellStrength;
    }

    function hit() public requireIntact {
        shellStrength--;
        hitsPerRound[round][msg.sender]++;
        emit Hit(round, msg.sender, shellStrength);
    }

    function shake(suint256 _numShakes) public requireIntact {
        kernel += _numShakes;
        emit Shake(round, msg.sender);
    }

    function reset() public requireCracked {
        shellStrength = initialShellStrength;
        kernel = initialKernel;
        round++;
        emit Reset(round, shellStrength);
    }

    function look() public view requireCracked onlyContributor returns (uint256) {
        return uint256(kernel);
    }

    function set_number(suint256 _kernel) public {
        kernel = _kernel;
    }

    modifier requireCracked() {
        require(shellStrength == 0, "SHELL_INTACT");
        _;
    }

    modifier requireIntact() {
        require(shellStrength > 0, "SHELL_ALREADY_CRACKED");
        _;
    }

    modifier onlyContributor() {
        require(hitsPerRound[round][msg.sender] > 0, "NOT_A_CONTRIBUTOR");
        _;
    }
}
```
### Script triển khai Walnut.s.sol:
File: packages/contracts/script/Walnut.s.sol

Nội dung
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {Walnut} from "../src/Walnut.sol";

contract WalnutScript is Script {
    Walnut public walnut;

    function run() public {
        uint256 deployerPrivateKey = vm.envUint("PRIVKEY");
        vm.startBroadcast(deployerPrivateKey);
        walnut = new Walnut(3, suint256(0));
        vm.stopBroadcast();
    }
}
```
### File CLI index.ts:
File: packages/cli/src/index.ts

Nội dung

```typescript

import dotenv from 'dotenv'
import { join } from 'path'
import { defineChain } from 'viem'
import { CONTRACT_DIR, CONTRACT_NAME } from '../lib/constants'
import { readContractABI, readContractAddress } from '../lib/utils'
import { App } from './app'

dotenv.config()

const seismicDevnet = defineChain({
  id: 5124,
  name: 'Seismic Devnet',
  network: 'seismic-devnet',
  nativeCurrency: {
    decimals: 18,
    name: 'Ether',
    symbol: 'ETH',
  },
  rpcUrls: {
    default: {
      http: ['https://node-2.seismicdev.net/rpc'],
      webSocket: ['wss://node-2.seismicdev.net/ws'],
    },
  },
  blockExplorers: {
    default: {
      name: 'SeismicScan',
      url: 'https://explorer-2.seismicdev.net',
    },
  },
})

async function main() {
  if (!process.env.CHAIN_ID || !process.env.RPC_URL) {
    console.error('Please set your environment variables.')
    process.exit(1)
  }

  const broadcastFile = join(
    CONTRACT_DIR,
    'broadcast',
    `${CONTRACT_NAME}.s.sol`,
    process.env.CHAIN_ID,
    'run-latest.json'
  )
  const abiFile = join(
    CONTRACT_DIR,
    'out',
    `${CONTRACT_NAME}.sol`,
    `${CONTRACT_NAME}.json'
  )

  const chain = seismicDevnet

  const players = [
    { name: 'Alice', privateKey: process.env.ALICE_PRIVKEY! },
    { name: 'Bob', privateKey: process.env.BOB_PRIVKEY! },
  ]

  const app = new App({
    players,
    wallet: {
      chain,
      rpcUrl: process.env.RPC_URL!,
    },
    contract: {
      abi: readContractABI(abiFile),
      address: readContractAddress(broadcastFile),
    },
  })

  await app.init()

  console.log('=== Round 1 ===')
  await app.shake('Alice', 2)
  await app.hit('Alice')
  await app.shake('Alice', 4)
  await app.hit('Alice')
  await app.shake('Alice', 1)
  await app.hit('Alice')
  await app.look('Alice')

  console.log('=== Round 2 ===')
  await app.reset('Bob')
  await app.hit('Bob')
  await app.shake('Bob', 1)
  await app.hit('Bob')
  await app.shake('Bob', 2)
  await app.hit('Bob')
  await app.look('Bob')

  console.log('=== Testing Access Control ===')
  console.log("Attempting Alice's look() in Bob's round (should revert)")
  try {
    await app.look('Alice')
    console.error('❌ Expected look() to revert but it succeeded')
    process.exit(1)
  } catch (error) {
    console.log('✅ Received expected revert')
  }
}

main()
```
### File app.ts:
File: packages/cli/src/app.ts

Nội dung
```typescript
import {
  type ShieldedContract,
  type ShieldedWalletClient,
  createShieldedWalletClient,
  getShieldedContract,
} from 'seismic-viem'
import { Abi, Address, Chain, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { getShieldedContractWithCheck } from '../lib/utils'

interface AppConfig {
  players: Array<{
    name: string
    privateKey: string
  }>
  wallet: {
    chain: Chain
    rpcUrl: string
  }
  contract: {
    abi: Abi
    address: Address
  }
}

export class App {
  private config: AppConfig
  private playerClients: Map<string, ShieldedWalletClient> = new Map()
  private playerContracts: Map<string, ShieldedContract> = new Map()

  constructor(config: AppConfig) {
    this.config = config
  }

  async init() {
    for (const player of this.config.players) {
      try {
        const walletClient = await createShieldedWalletClient({
          chain: this.config.wallet.chain,
          transport: http(this.config.wallet.rpcUrl),
          account: privateKeyToAccount(player.privateKey as `0x${string}`),
        })
        console.log(`Initialized wallet for ${player.name} with address: ${walletClient.account.address}`)
        this.playerClients.set(player.name, walletClient)

        const contract = await getShieldedContractWithCheck(
          walletClient,
          this.config.contract.abi,
          this.config.contract.address
        )
        this.playerContracts.set(player.name, contract)
      } catch (error) {
        console.error(`Failed to initialize wallet for ${player.name}:`, error)
        throw error
      }
    }
  }

  private getWalletClient(playerName: string): ShieldedWalletClient {
    const client = this.playerClients.get(playerName)
    if (!client) {
      throw new Error(`Wallet client for player ${playerName} not found`)
    }
    return client
  }

  private getPlayerContract(playerName: string): ShieldedContract {
    const contract = this.playerContracts.get(playerName)
    if (!contract) {
      throw new Error(`Shielded contract for player ${playerName} not found`)
    }
    return contract
  }

  async reset(playerName: string) {
    console.log(`- Player ${playerName} writing reset()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const hash = await contract.write.reset([], { gas: 200000n })
    console.log(`- Reset tx hash: ${hash}`)
    await walletClient.waitForTransactionReceipt({ hash })
  }

  async shake(playerName: string, numShakes: number) {
    console.log(`- Player ${playerName} writing shake(${numShakes})`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const hash = await contract.write.shake([numShakes], { gas: 100000n })
    console.log(`- Shake tx hash: ${hash}`)
    await walletClient.waitForTransactionReceipt({ hash })
  }

  async hit(playerName: string) {
    console.log(`- Player ${playerName} writing hit()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const hash = await contract.write.hit([], { gas: 200000n })
    console.log(`- Hit tx hash: ${hash}`)
    await walletClient.waitForTransactionReceipt({ hash })
  }

  async look(playerName: string) {
    console.log(`- Player ${playerName} reading look()`)
    const contract = this.getPlayerContract(playerName)
    const result = await contract.read.look()
    console.log(`- Player ${playerName} sees number:`, result)
    return result
  }
}
```
### File utils.ts:
File: packages/cli/lib/utils.ts

Nội dung:
```typescript
import fs from 'fs'
import { type ShieldedWalletClient, getShieldedContract } from 'seismic-viem'
import { Abi, Address } from 'viem'

async function getShieldedContractWithCheck(
  walletClient: ShieldedWalletClient,
  abi: Abi,
  address: Address
) {
  const contract = getShieldedContract({
    abi: abi,
    address: address,
    client: walletClient,
  })
  const code = await walletClient.getCode({
    address: address,
  })
  if (!code) {
    throw new Error('Please deploy contract before running this script.')
  }
  return contract
}

function readContractAddress(broadcastFile: string): `0x${string}` {
  const broadcast = JSON.parse(fs.readFileSync(broadcastFile, 'utf8'))
  if (!broadcast.transactions?.[0]?.contractAddress) {
    throw new Error('Invalid broadcast file format')
  }
  return broadcast.transactions[0].contractAddress
}

function readContractABI(abiFile: string): Abi {
  const abi = JSON.parse(fs.readFileSync(abiFile, 'utf8'))
  if (!abi.abi) {
    throw new Error('Invalid ABI file format')
  }
  return abi.abi
}

export { getShieldedContractWithCheck, readContractAddress, readContractABI }
```
### File constants.ts:
File: packages/cli/lib/constants.ts

Nội dung:
```typescript

import { join } from 'path'

const CONTRACT_NAME = 'Walnut'
const CONTRACT_DIR = join(__dirname, '../../contracts')

export { CONTRACT_NAME, CONTRACT_DIR }
```
## Bước 5: Cấu hình biến môi trường
File .env cho CLI:
File: packages/cli/.env

Nội dung:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
ALICE_PRIVKEY=<your-alice-private-key>
BOB_PRIVKEY=<your-bob-private-key>
```
Thay <your-alice-private-key> và <your-bob-private-key> bằng khóa riêng từ bước 2.

File .env cho contracts:
File: packages/contracts/.env

Nội dung:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
PRIVKEY=<your-deployer-private-key>
```
Thay <your-deployer-private-key> bằng khóa riêng của ví triển khai.

Bước 6: Cài đặt phụ thuộc
CLI:
```bash
cd ~/walnut-app/packages/cli
bun install
```
Contracts:
Foundry tự quản lý phụ thuộc, không cần cài thêm.

## Bước 7: Biên dịch và triển khai hợp đồng
Biên dịch:
```bash
cd ~/walnut-app/packages/contracts
sforge compile
```
Triển khai:
```bash
source .env
sforge script script/Walnut.s.sol:WalnutScript --rpc-url $RPC_URL --broadcast
```
Kiểm tra:
Mở packages/contracts/broadcast/Walnut.s.sol/5124/run-latest.json.

Tìm dòng "contractAddress": "0x..." để lấy địa chỉ hợp đồng.

Kiểm tra hợp đồng trên explorer:

https://explorer-2.seismicdev.net/address/<contract-address>
Bước 8: Chạy CLI
Chạy chương trình:
```bash
cd ~/walnut-app/packages/cli
bun run src/index.ts > output.txt
```
Kiểm tra output:
Mở output.txt, bạn sẽ thấy:
```
Initialized wallet for Alice with address: 0x...
Initialized wallet for Bob with address: 0x...
=== Round 1 ===
- Player Alice writing shake(2)
- Shake tx hash: 0x...
- Player Alice writing hit()
- Hit tx hash: 0x...
- Player Alice writing shake(4)
- Shake tx hash: 0x...
- Player Alice writing hit()
- Hit tx hash: 0x...
- Player Alice writing shake(1)
- Shake tx hash: 0x...
- Player Alice writing hit()
- Hit tx hash: 0x...
- Player Alice reading look()
- Player Alice sees number: 7n
=== Round 2 ===
- Player Bob writing reset()
- Reset tx hash: 0x...
- Player Bob writing hit()
- Hit tx hash: 0x...
- Player Bob writing shake(1)
- Shake tx hash: 0x...
- Player Bob writing hit()
- Hit tx hash: 0x...
- Player Bob writing shake(2)
- Shake tx hash: 0x...
- Player Bob writing hit()
- Hit tx hash: 0x...
- Player Bob reading look()
- Player Bob sees number: 3n
=== Testing Access Control ===
Attempting Alice's look() in Bob's round (should revert)
- Player Alice reading look()
✅ Received expected revert
```


