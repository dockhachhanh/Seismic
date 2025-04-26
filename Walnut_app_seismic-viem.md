# Hướng Dẫn Cài Đặt và Triển Khai Walnut App trên Seismic Devnet - Sử dụng seismic-viem v1.0.38
Hướng dẫn này cung cấp các bước chi tiết để thiết lập, triển khai, và chạy dự án Walnut App trên Seismic Devnet, một ứng dụng blockchain sử dụng hợp đồng thông minh và CLI để mô phỏng trò chơi multiplayer. Hướng dẫn được viết rõ ràng, phù hợp cho cả người mới bắt đầu, và bao gồm các mẹo xử lý sự cố.

- Ngày cập nhật: 26/04/2025
- Yêu cầu: Ubuntu Linux 22.04 lts 
- Kết nối internet ổn định.
---
# Hướng dẫn này sẽ giúp bạn cài đặt từ đầu và chạy dự án thành công.
# 1. Chuẩn bị môi trường
## 1.1 ## Yêu cầu trước khi bắt đầu
- Máy tính: Linux (Ubuntu khuyến nghị)
- Kết nối internet: Để tải gói và tương tác với Seismic Devnet.
- Kiến thức cơ bản: Biết cách dùng terminal và chỉnh sửa file văn bản.
- Ví ETH: Bạn sẽ cần ít nhất 2 ví với ít nhất 0.01 ETH mỗi ví trên Seismic Devnet.
---
## 1.2 Cài đặt các công cụ cần thiết
### 1.2.1 Cập nhật hệ thống (Linux)
Nếu dùng Linux, cập nhật hệ thống và cài đặt các gói hỗ trợ:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
### 1.2.2 Cài đặt Node.js và npm
Sử dụng nvm để cài Node.js phiên bản 23:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
source ~/.bashrc
nvm install 23
node -v  # Nên hiển thị v23.x.x
npm -v  # Nên hiển thị 10.x.x
```
```bash
npm install -g npm@latest
```
### 1.2.3 Cài đặt Bun
Cài Bun, một công cụ thay thế npm/yarn:
```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun -v  # Nên hiển thị 1.x.x
```
### 1.2.4 Cài đặt rust
```bash
# install rust and cargo
curl https://sh.rustup.rs -sSf | sh
```
Chọn 1 và enter để cài đặt
### 1.2.5 Cài đặt sfoundryup
```bash
curl -L \
     -H "Accept: application/vnd.github.v3.raw" \
     "https://api.github.com/repos/SeismicSystems/seismic-foundry/contents/sfoundryup/install?ref=seismic" | bash
source ~/.bashrc
```
```bash
sfoundryup
source ~/.bashrc
```
---
# 2. Thiết lập cấu trúc dự án
## 2.1 Tạo thư mục dự án
Tạo thư mục walnut-app và thiết lập cấu trúc monorepo:
```bash
mkdir walnut-app
cd walnut-app
mkdir -p packages/contracts packages/cli
```
## 2.2 Khởi tạo monorepo
Khởi tạo dự án Bun tại thư mục gốc:
```bash
bun init -y
rm index.ts tsconfig.json
touch .prettierrc .gitmodules .gitignore
```
### 2.2.1 Cấu hình package.json
Mở package.json và thay bằng:
```json
{
  "workspaces": ["packages/**"],
  "dependencies": {},
  "devDependencies": {
    "@trivago/prettier-plugin-sort-imports": "^5.2.1",
    "prettier": "^3.4.2"
  }
}
```
### 2.2.2 Cấu hình .prettierrc
Mở .prettierrc và thêm:
```json
{
  "semi": false,
  "tabWidth": 2,
  "singleQuote": true,
  "printWidth": 80,
  "trailingComma": "es5",
  "plugins": ["@trivago/prettier-plugin-sort-imports"],
  "importOrder": [
    "<TYPES>^(?!@)([^.].*$)</TYPES>",
    "<TYPES>^@(.*)$</TYPES>",
    "<TYPES>^[./]</TYPES>",
    "^(?!@)([^.].*$)",
    "^@(.*)$",
    "^[./]"
  ],
  "importOrderParserPlugins": ["typescript", "jsx", "decorators-legacy"],
  "importOrderSeparation": true,
  "importOrderSortSpecifiers": true
}
```
### 2.2.3 Cấu hình .gitignore
Mở .gitignore và thêm:
```
# Compiler files
cache/
out/
# Ignores development broadcast logs
!/broadcast
/broadcast/*/31337/
/broadcast/**/dry-run/
# Docs
docs/
# Dotenv file
.env
node_modules/
```
### 2.2.4 Cấu hình .gitmodules
Mở .gitmodules và thêm:
```
[submodule "packages/contracts/lib/forge-std"]
path = packages/contracts/lib/forge-std
url = https://github.com/foundry-rs/forge-std
```
---
# 3. Thiết lập thư mục packages/contracts
## 3.1 Khởi tạo hợp đồng
Di chuyển vào packages/contracts:
```bash
cd packages/contracts
sforge init --no-commit
rm -rf .github
rm src/Counter.sol test/Counter.t.sol script/Counter.s.sol
touch src/Walnut.sol test/Walnut.t.sol script/Walnut.s.sol
touch .gitignore
```
### 3.1.1 Cấu hình .gitignore
Mở packages/contracts/.gitignore và thêm:
```
.env
broadcast/
cache/
```
## 3.2 Thêm hợp đồng Walnut.sol
Mở packages/contracts/src/Walnut.sol và thêm:
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
## 3.3 Thêm script triển khai Walnut.s.sol
Mở packages/contracts/script/Walnut.s.sol và thêm:
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
## 3.4 Cấu hình .env
Tạo packages/contracts/.env:
```bash
nano .env
```
Thêm nội dung:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
PRIVKEY=0x<your-private-key>  # thêm 0x nếu thiếu.
```
- PRIVKEY: Khóa riêng của ví đã nhận ETH từ https://faucet-2.seismicdev.net/. Tạo ví qua MetaMask và yêu cầu ETH (ít nhất 0.1 ETH).
## 3.5 Triển khai hợp đồng
Chạy lệnh triển khai:
```bash
cd packages/contracts
source .env
sforge script script/Walnut.s.sol:WalnutScript --rpc-url $RPC_URL --broadcast
```
- Sao chép địa chỉ hợp đồng từ packages/contracts/broadcast/Walnut.s.sol/5124/run-latest.json (trường contractAddress).
---
# 4. Thiết lập thư mục packages/cli
## 4.1 Khởi tạo CLI
Di chuyển vào packages/cli:
```bash
cd ../cli
bun init -y
mkdir -p src lib
mv index.ts src/
touch src/app.ts lib/constants.ts lib/utils.ts
touch .gitignore
```
### 4.1.1 Cấu hình .gitignore
Mở packages/cli/.gitignore và thêm:
```
node_modules
```
## 4.2 Cấu hình package.json
Mở packages/cli/package.json và thay bằng:
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
    "seismic-viem": "^1.0.38",
    "viem": "^2.28.0"
  },
  "devDependencies": {
    "@types/node": "^22.7.6",
    "typescript": "^5.6.3"
  }
}
```
Cài đặt dependencies:
```
bun install
```
## 4.3 Thêm constants.ts
Mở packages/cli/lib/constants.ts và thêm:
```typescript
import { join } from 'path'

const CONTRACT_NAME = 'Walnut'
const CONTRACT_DIR = join(__dirname, '../../contracts')

export { CONTRACT_NAME, CONTRACT_DIR }
```
## 4.4 Thêm utils.ts
Mở packages/cli/lib/utils.ts và thêm:
```typescript
import fs from 'fs'
import { getShieldedContract } from 'seismic-viem'
import { Abi, Address } from 'viem'

async function getShieldedContractWithCheck(
  walletClient: any,
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
## 4.5 Thêm app.ts
Mở packages/cli/src/app.ts và thêm:
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
      const walletClient = await createShieldedWalletClient({
        chain: this.config.wallet.chain,
        transport: http(this.config.wallet.rpcUrl),
        account: privateKeyToAccount(player.privateKey as `0x${string}`),
      })
      this.playerClients.set(player.name, walletClient)

      const contract = await getShieldedContractWithCheck(
        walletClient,
        this.config.contract.abi,
        this.config.contract.address
      )
      this.playerContracts.set(player.name, contract)
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
    const nonce = await walletClient.getTransactionCount({
      address: walletClient.account.address,
      blockTag: 'latest',
    })
    await walletClient.waitForTransactionReceipt({
      hash: await contract.write.reset([], { gas: 100000n, nonce }),
    })
  }

  async shake(playerName: string, numShakes: number) {
    console.log(`- Player ${playerName} writing shake()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const nonce = await walletClient.getTransactionCount({
      address: walletClient.account.address,
      blockTag: 'latest',
    })
    await walletClient.waitForTransactionReceipt({
      hash: await contract.write.shake([numShakes], { gas: 50000n, nonce }),
    })
  }

  async hit(playerName: string) {
    console.log(`- Player ${playerName} writing hit()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const nonce = await walletClient.getTransactionCount({
      address: walletClient.account.address,
      blockTag: 'latest',
    })
    await walletClient.waitForTransactionReceipt({
      hash: await contract.write.hit([], { gas: 100000n, nonce }),
    })
  }

  async look(playerName: string) {
    console.log(`- Player ${playerName} reading look()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    const nonce = await walletClient.getTransactionCount({
      address: walletClient.account.address,
      blockTag: 'latest',
    })
    const result = await contract.read.look([], { nonce })
    console.log(`- Player ${playerName} sees number:`, result)
  }
}
```
## 4.6 Thêm index.ts
Mở packages/cli/src/index.ts và thêm:
```typescript
import dotenv from 'dotenv'
import { join } from 'path'
import { seismicDevnet } from 'seismic-viem'
import { CONTRACT_DIR, CONTRACT_NAME } from '../lib/constants'
import { readContractABI, readContractAddress } from '../lib/utils'
import { App } from './app'

dotenv.config()

async function main() {
  if (!process.env.CHAIN_ID || !process.env.RPC_URL || !process.env.ALICE_PRIVKEY || !process.env.BOB_PRIVKEY) {
    console.error('Please set all required environment variables.')
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
    `${CONTRACT_NAME}.json`
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
  await app.reset('Alice')
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
  await app.shake('Bob', 1)
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
## 4.7 Cấu hình .env
Tạo packages/cli/.env:
```bash
nano .env
```
thêm nội dung:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
ALICE_PRIVKEY=0x<alice-private-key>
BOB_PRIVKEY=0x<bob-private-key>
# thêm 0x nếu thiếu.
```
- ALICE_PRIVKEY, BOB_PRIVKEY: Hai khóa riêng khác nhau, mỗi ví cần ít nhất 0.1 ETH từ https://faucet-2.seismicdev.net/.
---
# 5. Chạy ứng dụng
## 5.1 Cài đặt dependencies
Từ thư mục gốc walnut-app:
```bash
bun install
```
## 5.2 Chạy CLI
Di chuyển vào packages/cli và chạy:
```bash
cd packages/cli
bun dev
```
## 5.3 Kết quả mong đợi
Bạn sẽ thấy output như sau:
```
=== Round 1 ===
- Player Alice writing reset()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice reading look()
- Player Alice sees number: 7n
=== Round 2 ===
- Player Bob writing reset()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob reading look()
- Player Bob sees number: 2n
=== Testing Access Control ===
Attempting Alice's look() in Bob's round (should revert)
✅ Received expected revert
```
---
# 6. Kiểm tra giao dịch trên Explorer
- Truy cập https://explorer-2.seismicdev.net/.
- Nhập địa chỉ hợp đồng (từ run-latest.json) hoặc địa chỉ ví của Alice/Bob để xem lịch sử giao dịch.
- Xác nhận các giao dịch hit, shake, reset, và look được ghi nhận.
---

# CHÚC CÁC BẠN THÀNH CÔNG
